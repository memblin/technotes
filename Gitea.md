# Gitea - TKCLabs

I didn't think that GitHub hit my usage when I'm using self-hosted runners but
I recently had a project shutdown because of Actions minutes consumption even
though I was 100% on self-hosted runners.

To work around this I watned to move to a self-hosted Git solution.

- [Gitea](https://about.gitea.com/)

## Quadlet

We're lauching in a systemd container via quadlet using podman volumes for our
persistance layer.

Note a few things to changes if you're copy / pating.

- Linger username: `tkcadmin` might not meet your need.

```bash
# Enable linger
sudo loginctl enable-linger tkcadmin

# Create systemd user-space directory
mkdir -p ~/.config/containers/systemd/

# Write volume files:
#  - ~/.config/containers/systemd/gitea-data.volume
#  - ~/.config/containers/systemd/gitea-config.volume
[Volume]

# Write a unit file: ~/.config/containers/systemd/gitea.container
[Unit]
Description=Gitea rootless server
After=network-online.target

[Container]
Image=docker.gitea.com/gitea:1.26.4-rootless
Volume=gitea-data.volume:/var/lib/gitea
Volume=gitea-config.volume:/etc/gitea
PublishPort=3000:3000
PublishPort=2222:2222

[Service]
Restart=always

[Install]
WantedBy=default.target

# Enable and start the quadlent based service
systemctl --user daemon-reload
systemctl start gitea.service
```

## Setting up Gitea Actions Runner

Install and register the Gitea Runner

- https://gitea.com/gitea/runner

```bash
mkdir -p /opt/gitia-runners/TKC-Labs && cd $_

wget https://dl.gitea.com/gitea-runner/2.0.1/gitea-runner-2.0.1-linux-amd64

chmod 0755 ./gitea-runner-2.0.1-linux-amd64

mv ./gitea-runner-2.0.1-linux-amd64 ./gitea-runner

./gitea-runner register
```

## Systemd unit file for the runner

Create the systemd unit file

```bash
# /etc/systemd/system/gitea-runner-tkc-labs.service

[Unit]
Description=Gitea Actions Runner (TKC-Labs)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=ghrunner
Group=ghrunner
WorkingDirectory=/opt/gitia-runners/TKC-Labs
ExecStart=/opt/gitia-runners/TKC-Labs/gitea-runner daemon
Restart=on-failure
RestartSec=5
# Uncomment if the runner writes logs/temp files that need a stable HOME
Environment=HOME=/opt/gitia-runners/TKC-Labs

# Hardening (loosen if the runner needs broader access, e.g. Docker socket)
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/opt/gitia-runners/TKC-Labs
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gitea-runner-tkc-labs.service
sudo systemctl status gitea-runner-tkc-labs.service
```

## Gitea CLI

Get the CLI

- https://gitea.com/gitea/tea
