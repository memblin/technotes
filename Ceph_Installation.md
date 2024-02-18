# Ceph on Debian 12

Ceph Installation & Config

## Debian 12

```bash
# Install the packages we need
apt update
apt install -y podman cephadm

# Bootstrap the first Node
cephadm bootstrap --mon-ip 100.64.10.51 --cluster-network 100.64.15.0/24

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
cephadm shell -- ceph orch host add ceph02 100.64.10.52 --labels _admin

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

