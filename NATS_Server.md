# NATS Server

From the nats.io website:

"Software applications and services need to exchange data. NATS is an infrastructure that allows such data exchange, segmented in the form of messages. We call this a "message oriented middleware"."

- [nats.io](https://nats.io/)
- [github.com/nats-io/nats-server](https://github.com/nats-io/nats-server)

## 3 Node Cluster Configuration with Jetstream

The `pass` entries below should be bcrypted with `nats server passwd` utility in the NATS CLI tool.  

The `nats-box` container contains this utility:

`podman run -it --rm natsio/nats-box nats server passwd`

### Configs

```conf
server_name=nats01.tkclabs.io
listen=4222

accounts {
  $SYS {
    users = [
      { user: "admin",
        pass: "$2WriteABCryptedValueCreatedWithNatsServerPasswdHere"
      }
    ]
  }
}

jetstream {
   store_dir=/srv/nats-data
}

cluster {
  name: tkclabs
  listen: 0.0.0.0:6222
  routes: [
    nats://nats02.tkclabs.io:6222
    nats://nats03.tkclabs.io:6222
  ]
}
```

```conf
server_name=nats02.tkclabs.io
listen=4222

accounts {
  $SYS {
    users = [
      { user: "admin",
        pass: "$2WriteABCryptedValueCreatedWithNatsServerPasswdHere"
      }
    ]
  }
}

jetstream {
   store_dir=/srv/nats-data
}

cluster {
  name: tkclabs
  listen: 0.0.0.0:6222
  routes: [
    nats://nats01.tkclabs.io:6222
    nats://nats03.tkclabs.io:6222
  ]
}
```

```conf
server_name=nats03.tkclabs.io
listen=4222

accounts {
  $SYS {
    users = [
      { user: "admin",
        pass: "$2WriteABCryptedValueCreatedWithNatsServerPasswdHere"
      }
    ]
  }
}

jetstream {
   store_dir=/srv/nats-data
}

cluster {
  name: tkclabs
  listen: 0.0.0.0:6222
  routes: [
    nats://nats02.tkclabs.io:6222
    nats://nats01.tkclabs.io:6222
  ]
}
```

### Pre-config

```bash
# Make jetstream data dir
sudo mkdir --mode 0700 /srv/nats-data
sudo chown tkcadmin:tkcadmin /srv/nats-data

# Pull the nats server image
podman pull docker.io/library/nats

# Pull NATs CLI tools image
podman pull docker.io/natsio/nats-box

# Create a password (it will prompt and output bcrypted value)
# Update value in configs
podman run -it --rm natsio/nats-box nats server passwd

# Drop server config on host in
sudo mkdir --mode 0700 /etc/nats
sudo chown tkcadmin:tkcadmin /etc/nats
# Write file and set access
sudo chmod 0600 /etc/nats/nats-server.conf

# Open firewall ports
sudo firewall-cmd --add-port={4222/tcp,6222/tcp,8222/tcp}
sudo firewall-cmd --add-port={4222/tcp,6222/tcp,8222/tcp} --permanent
```

### Start NATS

```bash
# Start NATS in ephemeral mode
podman run -it --rm --name nats -v "/etc/nats/nats-server.conf:/nats-server.conf:z" -v "/srv/nats-data:/srv/nats-data:z" --publish-all nats
# Start NATs in daemon mode
podman run -d --name nats -v "/etc/nats/nats-server.conf:/nats-server.conf:z" -v "/srv/nats-data:/srv/nats-data:z" --publish-all nats
```

### Launch NATS via systemd with Quadlet

```bash
# Create our containers path; this is a systemd thing so
# don't try to put the path elsewhere.
mkdir -p $HOME/.config/containers/systemd

# Heredoc a nats.container file into place
cat <<EOF > $HOME/.config/containers/systemd/nats.container
[Unit]
Description=NATS.io Server
After=local-fs.target

[Container]
ContainerName=nats
Image=docker.io/library/nats
PublishPort=4222:4222
PublishPort=6222:6222
PublishPort=8222:8222
Volume=/srv/nats-data:/srv/nats-data:z
Volume=/etc/nats/nats-server.conf:/nats-server.conf:z

[Install]
# Start by default on boot
WantedBy=multi-user.target default.target
EOF

# If you used the short-name for the Image above pull
# the contianer image manually so the registry is not
# prompted for. This causes the service to fail to start
podman pull docker.io/library/nats

# execute a user-level daemon-reload so systemd sees
# the new service file 
systemctl --user daemon-reload

# Check that it shows up
systemctl --user status nats.service

# Start up the service container
systemctl --user start nats.service

# Enable Linger so container keeps running
sudo loginctl enable-linger tkcadmin
```
