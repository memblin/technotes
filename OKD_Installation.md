# OKD installation

## Get the OKD Installer & Client

Latest stable releases at https://github.com/okd-project/okd/releases

```bash
# Installer
wget https://github.com/okd-project/okd/releases/download/4.15.0-0.okd-2024-02-10-035534/openshift-install-linux-4.15.0-0.okd-2024-02-10-035534.tar.gz

# Client
wget https://github.com/okd-project/okd/releases/download/4.15.0-0.okd-2024-02-10-035534/openshift-client-linux-4.15.0-0.okd-2024-02-10-035534.tar.gz
```

## Setup for Install

```bash
# Create a cluster config directory
mkdir okd01 && cd okd01

```
