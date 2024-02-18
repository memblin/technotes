# Ceph on Debian 12

Ceph Installation & Config

## Debian 12

```bash
# Install the packages we need
apt update
apt install -y podman cephadm

# Bootstrap the first Node
cephadm bootstrap --mon-ip 100.64.15.51 --cluster-network 100.64.15.0/24

# should produce output and then drop this infos
Ceph Dashboard is now available at:

             URL: https://ceph01.tkclabs.io:8443/
            User: admin
        Password: g00ufelnlj

Enabling client.admin keyring and conf on hosts with "admin" label
Enabling autotune for osd_memory_target
You can access the Ceph CLI as following in case of multi-cluster or non-default config:

        sudo /usr/sbin/cephadm shell --fsid ffd7f3dc-cdda-11ee-bcac-5254007db803 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

Or, if you are only running a single cluster on this host:

        sudo /usr/sbin/cephadm shell 

Please consider enabling telemetry to help improve Ceph:

        ceph telemetry on

For more information see:

        https://docs.ceph.com/en/pacific/mgr/telemetry/

Bootstrap complete.



# Add a host
#
# Install the cluster's key
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph02.tkclabs.io

# Add the node
# Only admin nodes need _admin label
cephadm shell -- ceph orch host add ceph02 100.64.15.52 --labels _admin
cephadm shell -- ceph orch host add ceph03 100.64.15.53 --labels _admin

# Check cluster status
cephadm shell -- ceph status

# Add Storage
cephadm shell -- ceph orch apply osd --all-available-devices

```

## object Storage

```bash
cephadm shell -- ceph dashboard set-rgw-credentials
cephadm shell -- ceph dashboard set-rgw-api-ssl-verify False
```

## rbds

```bash

```

```bash
apt install ceph-common

# Get client keys fo Ceph

# Map a rados device, pool/image_name 
rbd device map rados-pool01/test01

# Umap
rbd device unmap rados-pool01/test01
```


## cephadm on AlmaLinux 9

```bash
# Get Standalone
CEPH_RELEASE=18.2.1
curl --silent --remote-name --location https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm

# Then install
chmod 0700 ./cephadm
./cephadm add-repo --release reef
./cephadm install

# Set Storage Net IPs
nmcli connection modify "Wired connection 1" connection.id enp11s0 ipv4.method manual ipv4.addresses 100.64.15.51/24 ipv4.never-default yes ipv6.method disabled
nmcli connection modify "Wired connection 1" connection.id enp11s0 ipv4.method manual ipv4.addresses 100.64.15.52/24 ipv4.never-default yes ipv6.method disabled
nmcli connection modify "Wired connection 1" connection.id enp11s0 ipv4.method manual ipv4.addresses 100.64.15.53/24 ipv4.never-default yes ipv6.method disabled

# Bootstrap away
```