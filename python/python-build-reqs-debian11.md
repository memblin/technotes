# Python Build Requirements - Debian 11

These steps are to prepare a system to be a build and test system for python.

First, make sure you have enabled the source packages in the sources list. The `deb-src` should be uncommented in `/etc/apt/sources.list`

```bash
# Example
salt-dev:~/repos/github/python# cat /etc/apt/sources.list
deb http://deb.debian.org/debian bullseye main
deb-src http://deb.debian.org/debian bullseye main
deb http://security.debian.org/debian-security bullseye-security main
deb-src http://security.debian.org/debian-security bullseye-security main
deb http://deb.debian.org/debian bullseye-updates main
deb-src http://deb.debian.org/debian bullseye-updates main
deb http://deb.debian.org/debian bullseye-backports main
deb-src http://deb.debian.org/debian bullseye-backports main
```

Then you should update the packages index and system before installing the required packages.

This example was tested and applied on a VM started with this image as it's base:

- https://cloud.debian.org/images/cloud/bullseye/20241202-1949/debian-11-generic-amd64-20241202-1949.qcow2

## Shell Example

```bash
# Update Apt and the system
sudo apt-get update
sudo apt-get upgrade

# Install build dependencies and git
sudo apt-get build-dep python3 git
sudo apt-get install pkg-config

# If you want to build all optional modules, install the following packages and their dependencies:
sudo apt-get install build-essential gdb lcov pkg-config \
      libbz2-dev libffi-dev libgdbm-dev libgdbm-compat-dev liblzma-dev \
      libncurses5-dev libreadline6-dev libsqlite3-dev libssl-dev \
      lzma lzma-dev tk-dev uuid-dev zlib1g-dev libmpdec-dev

# For clang builds
sudo apt-get install clang

# If you're using root, create a regular user for building
useradd -c 'TKC Admin' -m -s /usr/bin/bash tkcadmin

# Give unconditional sudo
echo 'tkcadmin ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/tkcadmin

# Create /home/tkcadmin/.ssh and copy over our authorized keys
mkdir --mode=0700 /home/tkcadmin/.ssh
cp ~/.ssh/authorized_keys /home/tkcadmin/.ssh
chown -R tkcadmin:tkcadmin /home/tkcadmin/.ssh

# Assume tkcadmin user or logout and back in as tkcadmin
su - tkcadmin
# Check sudo
sudo -l

# As our regular user
#
# Create a location for your source repo; you can clone from a fork or the upstream
# but you'll want a personal fork if you intend to contributed back to the project.
mkdir -p ~/repos/github/python && cd $_
git clone https://github.com/python/cpython.git
```

ref: [Python Build Dependencies](https://devguide.python.org/getting-started/setup-building/#build-dependencies)
