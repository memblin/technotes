# Podman

## Containers in systemd via quadlets

Quadlet is an opinionated tool for easily running podman system containers under systemd in an optimal way.

- [github.com/containers/quadlet](https://github.com/containers/quadlet) *(Merged into Podman 4.4)*
- [Making systemd better for Podman with Quadlet](https://www.redhat.com/sysadmin/quadlet-podman)

### Usage Example: Sonatype Nexus Repository 3

Sonatype Nexus Repository 3 is a repository manager with a very full featured OSS option compared to alternatives. I use it as a passthrough cache for `YUM`, `APT`, `PyPI`, `Docker`, and `generic HTTP (Raw)` content as well as a hosted repository for each of these types.

This is not an exhaustive list, more repository types are supported.

- [Sonatype Nexus Repository](https://help.sonatype.com/en/sonatype-nexus-repository.html)
- [hub.docker.com - sonatype/nexus3](https://hub.docker.com/r/sonatype/nexus3)

#### Pre-config

When I run Nexus3 in a container I normally mount the persistent data store directly from a host directory so it doesn't fill up the default container storage with cached repository content. If you don't care about this you can use a standard container volume and skip all of this.

When running the container as root the host directory needs to belong to user and group `200:200` because nexus runs as uid/gid 200 inside the container. As root, that looks something like this without using Quadlet.

```bash
# Create the persistent directory
mkdir /srv/nexus-data && chown -R 200:200 /srv/nexus-data

# Start the container
podman run -d -p 8081:8081 --name nexus -v /srv/nexus-data:/nexus-data sonatype/nexus3
```

For nexus containers running as a normal user you will need to learn your subuid and subgid to properly adjust the directory permissions correctly. You won't use `200:200`.

```bash
# Get our subuid/subgid for the logged in user
grep $USER /etc/subuid /etc/subgid

# This command outputs like this on my test machine used to create this example
/etc/subuid:tkcadmin:524288:65536
/etc/subgid:tkcadmin:524288:65536

# Create a subuid/subgid adjusted directory for the container
# Take the first subuid: 524288
# Subtract 1 from the container UID: 199
# Add these values to get the UID for your data directory
# 524288 + 199 = 524487
# So our directory for nexus persistent data should be owned
# by 524487:524487
mkdir /srv/nexus-data && chown -R 524487:524487
```

With the persistent data directory in-place we can create our Quadlet configuration and adjust our linger configuration so the container stays running when we logout.

#### Nexus Quadlet

```bash
# Create our containers path; this is a systemd thing so
# don't try to put the path elsewhere.
mkdir -p $HOME/.config/containers/systemd

# Heredoc a nexus.container file into place
cat <<EOF > $HOME/.config/containers/systemd/nexus.container
[Unit]
Description=Sonatype Nexus3
After=local-fs.target

[Container]
ContainerName=nexus
Image=docker.io/sonatype/nexus3
PublishPort=8081:8081
Volume=/srv/nexus-data:/nexus-data:Z

[Install]
# Start by default on boot
WantedBy=multi-user.target default.target
EOF

# If you used the short-name for the Image above pull
# the contianer image manually so the registry is not
# prompted for. This causes the service to fail to start
podman pull docker.io/sonatype/nexus3

# execute a user-level daemon-reload so systemd sees
# the new service file 
systemctl --user daemon-reload

# Check that it shows up
systemctl --user status nexus.service

# Start up the service container
systemctl --user start nexus.service

# Pop some holes on the firewall for 8081/tcp
sudo firewall-cmd --add-port=8081/tcp
sudo firewall-cmd --add-port=8081/tcp --permanent

# Enable Linger so container keeps running
sudo loginctl enable-linger tkcadmin
```

Now the host should have Nexus Repository Manager running on port 8081. In my case I ran it on a machine named host01.tkclabs.io.

- https://host01.tkclabs.io:8081

Default login credentials are written to `/srv/nexus-data/admin.password` on the host after Nexus starts up the first time.

```bash
# ls the credentials file for the example
ls -la /srv/nexus-data/admin.password 

# Should get something like..
-rw-r--r--. 1 524487 524487 36 Mar  2 15:49 /srv/nexus-data/admin.password

# The content in this file is the default generated admin password.
```
