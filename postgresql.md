# PostgreSQL

Setup the PostgreSQL repositories

- https://yum.postgresql.org/

## Repository Configuration

```bash
# Heredoc a repository file file into place
cat <<EOF > /etc/yum.repos.d/postgresql15.repo
[postgresql-15]
name=PostgreSQL 15 for Fedora $releasever - $basearch
baseurl=https://download.postgresql.org/pub/repos/yum/15/fedora/fedora-$releasever-$basearch/
enabled=1
countme=1
metadata_expire=7d
repo_gpgcheck=0
type=rpm
gpgcheck=1
gpgkey=https://download.postgresql.org/pub/repos/yum/keys/PGDG-RPM-GPG-KEY-Fedora
skip_if_unavailable=False

EOF

# Install PostgreSQL 15
dnf install postgresql15-server

# Initialize the DB
/usr/pgsql-15/bin/postgresql-15-setup initdb

# Enable and start PostgreSQL
systemctl enable --now postgresql-15.service

# Open the firewall for PG
firewall-cmd --add-service postgresql
firewall-cmd --add-service postgresql --permanent
```

## Adding a user for terrform backend

This is an example of creating a database for a Terraform backend then adding a user with full access to that database.

```sql
-- Create the Database for Terraform Backend
CREATE DATABASE tf_vault_config;

-- Create the Role with the login privilege
-- Roles in PostgreSQL can be both Users and Groups
CREATE ROLE terraform LOGIN PASSWORD 'SecurePassword';

-- Grant all privileges on tf_vault_config Database
GRANT ALL PRIVILEGES ON DATABASE tf_vault_config TO terraform;

-- Connect to tf_vault_config
\c tf_vault_config;

-- Grant all privileges on tf_vault_config
GRANT ALL ON SCHEMA public TO terraform;
```