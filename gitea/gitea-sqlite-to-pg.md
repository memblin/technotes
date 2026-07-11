# Gitea SQLite → PostgreSQL Migration Plan

**Gitea host:** `git.tkclabs.io` — tkcadmin@100.64.10.6
**Hypervisor:** `host02.tkclabs.io` — claude@host02, go-virt virtd 0.4.7
**VM UUID:** `f48c0fcf-c401-4dac-985c-8128351700d7`
**VM pool:** `local`
**Disk UUID:** `f0c4a157-d626-4e4d-9e84-961a9049a11e` (75 GiB qcow2 on AlmaLinux 10 base)
**Current Gitea:** 1.26.4 rootless, Podman quadlet, systemd user service
**Current DB:** SQLite3 at `/var/lib/gitea/data/gitea.db` (~7.7 MB)
**Total data volume:** ~38 MB

---

## Overview

Migrate from embedded SQLite to PostgreSQL running as a separate rootless Podman container managed by quadlet/systemd. The process is fully reversible — the original SQLite database is never modified, and reverting is a config swap.

**Tested on:** Podman 5.8.2 (rootless), AlmaLinux 10, Gitea 1.26.4-rootless, PostgreSQL 16-alpine.

The migration has 7 phases:
0. VM-level disk snapshot (host02 — go-virt hypervisor)
1. Pre-flight health check (gitea)
2. Full backup (revert anchor)
3. Generate PostgreSQL-compatible SQL dump
4. Add PostgreSQL as a standalone quadlet container
5. Import data and switch config
6. Start and validate
7. *(Optional)* Clean up old SQLite config

---

## Phase 0: VM-Level Disk Snapshot (Hypervisor)

Run on **host02.tkclabs.io** as `claude`. This creates a qcow2 external snapshot of the git VM disk so you can revert the entire VM to its pre-migration state if anything goes catastrophically wrong.

> **virtctl flag ordering:** Flags (`--insecure`, `--token`, etc.) must come **before** the instance ID argument.

### Authentication

```bash
# Get a bearer token from the go-virt API
TOKEN=$(curl -sk -X POST https://127.0.0.1:8080/v1/auth/tokens \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"<bootstrap-admin-password>"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

echo "Token: $TOKEN"
# Token valid for 24h
```

### VM Identity

| Field | Value |
|---|---|
| VM UUID | `f48c0fcf-c401-4dac-985c-8128351700d7` |
| VM Name | `git-tkclabs-io` |
| Disk UUID | `f0c4a157-d626-4e4d-9e84-961a9049a11e` |
| Disk Path | `/var/lib/tkc/virt/pools/local/f48c0fcf-c401-4dac-985c-8128351700d7/disk.qcow2` |
| Pool | `local` |
| State | `running` |

### Create Snapshot

```bash
# The VM can stay running — external qcow2 snapshots are non-disruptive.
# The snapshot is a point-in-time copy of the disk at creation time;
# the VM continues writing to the original disk.qcow2 unaffected.

SNAP_NAME="pre-pg-migration-$(date +%Y%m%d)"

curl -sk -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"$SNAP_NAME\"}" \
  "https://127.0.0.1:8080/v1/instances/f48c0fcf-c401-4dac-985c-8128351700d7/disks/f0c4a157-d626-4e4d-9e84-961a9049a11e/snapshots" \
  | python3 -m json.tool

# Response will include the snapshot overlay path, e.g.:
# /var/lib/tkc/virt/pools/local/f48c0fcf-c401-4dac-985c-8128351700d7/<snap-id>.qcow2
```

### Verify Snapshot

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://127.0.0.1:8080/v1/instances/f48c0fcf-c401-4dac-985c-8128351700d7/disks/f0c4a157-d626-4e4d-9e84-961a9049a11e/snapshots" \
  | python3 -m json.tool
```

### Revert to Snapshot (if needed)

```bash
# 1. Stop the VM
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  "https://127.0.0.1:8080/v1/instances/f48c0fcf-c401-4dac-985c-8128351700d7/stop"

# 2. Revert disk to the snapshot
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  "https://127.0.0.1:8080/v1/instances/f48c0fcf-c401-4dac-985c-8128351700d7/disks/f0c4a157-d626-4e4d-9e84-961a9049a11e/snapshots/<SNAPSHOT_ID>/revert"

# 3. Start the VM
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  "https://127.0.0.1:8080/v1/instances/f48c0fcf-c401-4dac-985c-8128351700d7/start"
```

### Cleanup Snapshot (after migration is confirmed stable)

```bash
# Must stop the VM first
curl -sk -X DELETE -H "Authorization: Bearer $TOKEN" \
  "https://127.0.0.1:8080/v1/instances/f48c0fcf-c401-4dac-985c-8128351700d7/disks/f0c4a157-d626-4e4d-9e84-961a9049a11e/snapshots/<SNAPSHOT_ID>"
```

> **Note:** There is an existing test snapshot from discovery (`3446f191-d9f9-47e7-8b2f-4be197b43e77`, name "pre-pg-migration") on the VM. It is harmless (the VM still writes to `disk.qcow2`). Delete it after stopping the VM, or leave it as an extra restore point.

---

## Phase 1: Pre-flight Health Check

Clean up any database inconsistencies before dumping.

```bash
# Stop Gitea for consistency
systemctl --user stop gitea.service

# Health checks
podman exec --user git systemd-gitea \
  /usr/local/bin/gitea -c /etc/gitea/app.ini doctor check --all --fix

podman exec --user git systemd-gitea \
  /usr/local/bin/gitea -c /etc/gitea/app.ini doctor recreate-table
```

---

## Phase 2: Full Backup (Revert Anchor)

This is your revert point. If anything goes wrong, this ZIP gets you back to exactly the current state.

> **Important:** When Gitea is stopped via systemd, the container is **removed** — you can't `podman exec` or `podman cp` from it. Use a temporary container that mounts the same volumes.

```bash
# Stop Gitea (container will be removed)
systemctl --user stop gitea.service

# Run dump in a temporary container with the same volume mounts
podman run --rm --user git \
  -v /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-config/_data:/etc/gitea:Z \
  -v /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-data/_data:/var/lib/gitea:Z \
  docker.gitea.com/gitea:1.26.4-rootless \
  /usr/local/bin/gitea dump -c /etc/gitea/app.ini -t /var/lib/gitea/tmp

# The dump lands inside the data volume; copy it out
sudo cp /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-data/_data/gitea-dump-*.zip \
  ~/backup-pre-migration-$(date +%Y%m%d).zip
sudo chown tkcadmin:tkcadmin ~/backup-pre-migration-$(date +%Y%m%d).zip

# Verify contents
python3 -c "
import zipfile
z = zipfile.ZipFile('/home/tkcadmin/backup-pre-migration-$(date +%Y%m%d).zip')
for info in z.infolist():
    print(f'{info.file_size:>10}  {info.filename}')
"
```

---

## Phase 3: Generate PostgreSQL-Compatible SQL Dump

Produce a portable SQL file from the SQLite database in PostgreSQL dialect.

```bash
# Run dump in a temporary container (Gitea is still stopped)
podman run --rm --user git \
  -v /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-config/_data:/etc/gitea:Z \
  -v /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-data/_data:/var/lib/gitea:Z \
  docker.gitea.com/gitea:1.26.4-rootless \
  /usr/local/bin/gitea dump -c /etc/gitea/app.ini -t /var/lib/gitea/tmp --database postgres

# Extract the SQL file from the zip (the dump lands inside the data volume)
sudo python3 -c "
import zipfile
z = zipfile.ZipFile('/home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-data/_data/gitea-dump-*.zip')
# find the latest dump
import glob, os
files = sorted(glob.glob('/home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-data/_data/gitea-dump-*.zip'))
latest = files[-1]
z = zipfile.ZipFile(latest)
with open('/tmp/gitea-pg.sql', 'wb') as f:
    f.write(z.read('gitea-db.sql'))
print(f'Extracted {len(z.read(\"gitea-db.sql\"))} bytes from {latest}')
"

# Inspect the dump for sanity (should contain PostgreSQL-format CREATE TABLE, INSERT, etc.)
head -50 /tmp/gitea-pg.sql
wc -l /tmp/gitea-pg.sql
# Expected: lines starting with /*Generated by xorm ... from sqlite3 to postgres*/
#           CREATE TABLE with BIGSERIAL, CREATE INDEX, INSERT INTO
```

---

## Phase 4: Add PostgreSQL as a Standalone Quadlet Container

> **Why not a pod?** Podman 5.8.2 quadlet's pod support has quirks:
> - `.container` files with `Pod=` were silently ignored by the generator
> - `HealthCmd`/`HealthIntervalSec` directives caused the generator to skip the file entirely
> - Using standalone containers with `host.containers.internal` networking is simpler and more reliable

### Current Files

```
~/.config/containers/systemd/
├── gitea.container
├── gitea-config.volume
└── gitea-data.volume
```

### 4a. Create `gitea-postgres-data.volume`

```bash
cat > ~/.config/containers/systemd/gitea-postgres-data.volume << 'VOLEOF'
[Volume]
VOLEOF
```

### 4b. Create `postgres.container`

> **Naming note:** The file MUST be named `postgres.container` (not `gitea-postgres.container`).
> Quadlet may silently ignore files with certain naming patterns.
> **Do NOT use `HealthCmd` or `HealthIntervalSec`** — these directives cause quadlet to skip the file.

Generate a strong random password:

```bash
PG_PASSWORD=$(tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 32)
echo "PostgreSQL password: $PG_PASSWORD"
# SAVE THIS PASSWORD — you'll need it in app.ini
```

```bash
cat > ~/.config/containers/systemd/postgres.container << CNTEOF
[Container]
Image=docker.io/library/postgres:16-alpine
PublishPort=5432:5432
Volume=gitea-postgres-data.volume:/var/lib/postgresql/data
Environment=POSTGRES_USER=gitea
Environment=POSTGRES_PASSWORD=REPLACE_ME
Environment=POSTGRES_DB=gitea

[Service]
Restart=always

[Install]
WantedBy=default.target
CNTEOF

sed -i "s/REPLACE_ME/$PG_PASSWORD/" ~/.config/containers/systemd/postgres.container
```

> **Port binding:** Use `5432:5432` (all interfaces), NOT `127.0.0.1:5432:5432`. The loopback-only bind prevents Gitea's container from reaching PostgreSQL via `host.containers.internal`.

### 4c. Modify `gitea.container`

Backup the original and update to add postgres dependency and SELinux labels:

```bash
cp ~/.config/containers/systemd/gitea.container \
   ~/.config/containers/systemd/gitea.container.sqlite-backup
```

```bash
cat > ~/.config/containers/systemd/gitea.container << CNTEOF
[Unit]
Description=Gitea rootless server
After=network-online.target
Requires=postgres.service
After=postgres.service

[Container]
Image=docker.gitea.com/gitea:1.26.4-rootless
Volume=gitea-data.volume:/var/lib/gitea:Z
Volume=gitea-config.volume:/etc/gitea:Z
PublishPort=3000:3000
PublishPort=2222:2222

[Service]
Restart=always

[Install]
WantedBy=default.target
CNTEOF
```

> **SELinux `:Z`:** The `:Z` suffix on volume mounts is critical. Without it, the container gets `Permission denied` when accessing the data volume because the SELinux context doesn't match. This was the default from the original quadlet (which didn't specify `:Z`), but after stopping and re-creating the container, the context was lost.

### 4d. Reload Systemd

```bash
systemctl --user daemon-reload
```

---

## Phase 5: Import Data and Switch Config

### 5a. Start PostgreSQL

```bash
systemctl --user start postgres.service
sleep 5
podman exec systemd-postgres pg_isready -U gitea
# Expected: /var/run/postgresql:5432 - accepting connections
```

### 5b. Import the SQL Dump

```bash
# Import the SQL dump (from the /tmp/gitea-pg.sql extracted in Phase 3)
podman exec -i systemd-postgres psql -U gitea -d gitea \
  -c "SET synchronous_commit TO off; SET on_error_stop TO on;" \
  -f - < /tmp/gitea-pg.sql

# Verify tables were created
podman exec systemd-postgres \
  psql -U gitea -d gitea -c "\dt"
```

Expected output should show 112 tables including: `user`, `repository`, `issue`, `pull_request`, `milestone`, `action`, `attachment`, `comment`, `label`, `project`, `project_board`, etc.

### 5c. Backup and Update app.ini

```bash
# Backup current SQLite config
sudo cp /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-config/_data/app.ini \
   /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-config/_data/app.ini.sqlite-backup

# Update app.ini with sed (file is owned by container UID, use sudo)
sudo sed -i "s|DB_TYPE = sqlite3|DB_TYPE = postgres|" \
  /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-config/_data/app.ini
sudo sed -i "s|HOST = localhost:3306|HOST = host.containers.internal:5432|" \
  /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-config/_data/app.ini
sudo sed -i "s|USER = root|USER = gitea|" \
  /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-config/_data/app.ini
sudo sed -i "s|PASSWD = |PASSWD = $PG_PASSWORD|" \
  /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-config/_data/app.ini
sudo sed -i "s|PATH = /var/lib/gitea/data/gitea.db|# PATH removed - using PostgreSQL|" \
  /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-config/_data/app.ini
```

> **Networking:** Use `host.containers.internal:5432` (not `localhost:5432`). In rootless Podman, each container has its own network namespace — `localhost` means the container itself, not the host. `host.containers.internal` is a special DNS name that resolves to the host's gateway IP (169.254.1.2) and allows container-to-container communication via published ports.

The final `[database]` section should look like:

```ini
[database]
# PATH removed - using PostgreSQL
DB_TYPE = postgres
HOST = host.containers.internal:5432
NAME = gitea
USER = gitea
PASSWD = <the-generated-password>
SSL_MODE = disable
LOG_SQL = false
```

---

## Phase 6: Start Gitea and Validate

```bash
# Start Gitea (connects to PostgreSQL via host.containers.internal:5432)
systemctl --user start gitea.service
sleep 5

# Check that both containers are running
podman ps --format "table {{.Names}} {{.Status}} {{.Ports}}"

# Run full health check
podman exec --user git systemd-gitea \
  /usr/local/bin/gitea -c /etc/gitea/app.ini doctor check --all

# Verify data integrity
podman exec systemd-postgres psql -U gitea -d gitea -c \
  "SELECT (SELECT count(*) FROM repository) as repos,
          (SELECT count(*) FROM issue) as issues,
          (SELECT count(*) FROM pull_request) as prs,
          (SELECT count(*) FROM \"user\") as users;"

# Smoke test: visit the web UI
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/
# Should return 200
```

### Validation Checklist

- [ ] `podman ps` shows both containers running and healthy
- [ ] `doctor check --all` passes with no errors
- [ ] Web UI loads and login works
- [ ] Repositories are visible with correct content
- [ ] Issues and pull requests are intact
- [ ] Git clone/push works over SSH (port 2222)
- [ ] Git clone works over HTTP

---

## Phase 7: Cleanup (After Validation)

Once you're confident the migration is solid and you don't need to revert:

```bash
# Remove the old SQLite database file (or archive it)
mv ~/.local/share/containers/storage/volumes/systemd-gitea-data/_data/data/gitea.db \
   ~/.local/share/containers/storage/volumes/systemd-gitea-data/_data/data/gitea.db.old

# Remove SQLite config backup (or keep for reference)
# rm ~/.local/share/containers/storage/volumes/systemd-gitea-config/_data/app.ini.sqlite-backup
```

---

## Revert Procedure

If anything goes wrong during Phase 5 or 6:

```bash
# 1. Stop Gitea
systemctl --user stop gitea.service

# 2. Restore original SQLite app.ini
sudo cp /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-config/_data/app.ini.sqlite-backup \
   /home/tkcadmin/.local/share/containers/storage/volumes/systemd-gitea-config/_data/app.ini

# 3. Start Gitea — it's back on SQLite
systemctl --user start gitea.service

# 4. Stop PostgreSQL (optional — can leave it running, Gitea just won't use it)
systemctl --user stop postgres.service

# 5. To fully revert the quadlet setup:
rm ~/.config/containers/systemd/postgres.container
rm ~/.config/containers/systemd/gitea-postgres-data.volume
rm ~/.config/containers/systemd/gitea.container
cp ~/.config/containers/systemd/gitea.container.sqlite-backup \
   ~/.config/containers/systemd/gitea.container
systemctl --user daemon-reload
systemctl --user start gitea.service
```

The original `gitea.db` SQLite file is never touched — revert is simply telling Gitea to read from it again.

## Field Notes: What Diverged from the Original Plan

These are the real-world findings from executing this migration on Podman 5.8.2 rootless, AlmaLinux 10.

| Issue | Original Plan | What Actually Works |
|---|---|---|
| **Pod-based quadlet** | `.container` files with `Pod=gitea.pod` | Generator silently ignored them. Use standalone containers with `host.containers.internal` networking. |
| **Container file naming** | `gitea-postgres.container` | Generator ignored it. `postgres.container` works. |
| **`HealthCmd` in quadlet** | Used `HealthCmd=pg_isready` + `HealthIntervalSec=10s` | Generator silently skips any `.container` file containing these directives. Remove them. |
| **Backup from stopped container** | `podman exec` into the stopped Gitea container | Systemd-stopped quadlet containers are **removed**, not paused. Must run a temp container with the same volume mounts. |
| **SELinux volume labels** | Plain `Volume=name:/path` in quadlet | After stopping and restarting, new containers need `:Z` suffix (`Volume=name:/path:Z`) for SELinux relabeling. |
| **Container-to-container networking** | `localhost:5432` inside Gitea container to reach PostgreSQL | Each rootless container has its own network namespace. Use `host.containers.internal:5432` instead. |
| **PostgreSQL port bind** | `PublishPort=127.0.0.1:5432:5432` (loopback only) | `host.containers.internal` resolves to 169.254.1.2, not 127.0.0.1. Must bind to all interfaces: `PublishPort=5432:5432`. |
| **Doctor checks** | Run with Gitea stopped | Doctor needs the container running. Run it before stopping for the dump, or start a temp container. |
| **virtctl flag ordering** | Flags after instance ID | Flags (`--insecure`, `--token`) must come **before** the instance ID argument. |
| **Volume file ownership** | `sudo` not needed | Volume files are owned by the container's remapped UID (525287), not `tkcadmin` (1000). Use `sudo` to copy/move them. |

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| `gitea dump --database postgres` produces bad SQL | Low | Medium — import fails, stay on SQLite | Inspect the SQL file before importing. Failed import doesn't touch original data. |
| PostgreSQL container fails to start | Low | Medium — Gitea can't connect | Check `podman logs systemd-postgres`. Verify volume permissions. |
| Data corruption during import | Very low | High — but revert exists | Phase 2 backup is the anchor. `on_error_stop=on` aborts on first error. |
| Session loss (users logged out) | Certain | Low — minor inconvenience | Sessions are file-based in the data volume. They survive the migration. Users may need to re-login due to cookie changes. |

---

## Post-Migration Backup Procedure

With PostgreSQL, the backup procedure changes slightly. You can now do online backups with `pg_dump` (no downtime needed for the DB part), but the repos still need to be stopped for consistency.

```bash
# Stop Gitea
systemctl --user stop gitea.service

# Dump PostgreSQL
podman exec systemd-postgres \
  pg_dump -U gitea gitea > gitea-db-$(date +%Y%m%d).sql

# Full gitea dump still works (run in temp container since gitea is stopped)
podman run --rm --user git \
  -v systemd-gitea-data:/var/lib/gitea:Z \
  -v systemd-gitea-config:/etc/gitea:Z \
  docker.gitea.com/gitea:1.26.4-rootless \
  /usr/local/bin/gitea dump -c /etc/gitea/app.ini -t /var/lib/gitea/tmp

# Start Gitea
systemctl --user start gitea.service
```

---

## Files Created During Migration

| File | Purpose |
|---|---|
| `backup-pre-migration-YYYYMMDD.zip` | Full backup anchor — keep until migration confirmed stable |
| `/tmp/gitea-pg.sql` | Extracted PostgreSQL-format SQL dump (from Phase 3) |
| `postgres.container` | PostgreSQL standalone quadlet (named `postgres`, not `gitea-postgres`) |
| `gitea-postgres-data.volume` | PostgreSQL data volume |
| `gitea.container.sqlite-backup` | Original container file (revert path) |
| `app.ini.sqlite-backup` | Original config with SQLite settings |
