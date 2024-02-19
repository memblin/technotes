# Ceph

Ceph Installation & Config

## Debian 12

```bash
# Install the packages we need
apt update
apt install -y podman cephadm

# Bootstrap the first Node
cephadm bootstrap --mon-ip 100.64.15.51 --cluster-network 100.64.15.0/24

# The bootstrap command should produce output and then drop info like this
# 
Ceph Dashboard is now available at:

             URL: https://ceph01.tkclabs.io:8443/
            User: admin
        Password: <--REDACTED-->

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

## AlmaLinux 9

AlmaLinux 9 does not appear to have cephadm directly in the repos but we can install cephadm like this.

```bash
# Install the packages we need
dnf install -y podman

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

# Bootstrap the first Node
cephadm bootstrap --mon-ip 100.64.15.51 --cluster-network 100.64.15.0/24

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

## Ceph Operations

### Pool Management

In the default configuration it won't let you delete a pool that has been created.

Toggle this and you should be able to delete a pool.

```bash
# Launch the cephadm shell container
[root@ceph01 ~]# cephadm shell
# Check It
[ceph: root@ceph01 /]# ceph config get mon mon_allow_pool_delete
false
# Set It
[ceph: root@ceph01 /]# ceph config set mon mon_allow_pool_delete true
# Check It Again
[ceph: root@ceph01 /]# ceph config get mon mon_allow_pool_delete
true
```

### User Management

The `client.<username>` naming is required

- [Ceph: User Management](https://docs.ceph.com/en/latest/rados/operations/user-management/#user)

In the examples below we create the user `client.block` with access to the pool `block01` which happens to be allocated for rbd usage.

```bash
# Other Commands
ceph auth list

# Create a client user for pool block01
ceph auth add client.block mon 'allow r' osd 'allow rw pool=block01'

# Get a users key and capabilities
ceph auth get client.block

# Get users key in keyfile format with username
ceph auth get-or-create client.block mon 'allow r' osd 'allow rw pool=block01'

# Same as above but write the output to a file
ceph auth get-or-create client.block mon 'allow r' osd 'allow rw pool=block01' -o client.block.key

# Get the users key and nothing else for applications that just need the key
ceph auth get-or-create-key client.block mon 'allow r' osd 'allow rw pool=block01' -o client.block.key

# Adjusting a users capabilities; adding more permisisons for using rbd
ceph auth caps client.libvirt mon 'profile rbd' osd 'profile rbd pool=block01'
ceph auth caps client.block mon 'profile rbd' osd 'profile rbd pool=block01'

```

### Client Configuration

- Install `ceph-common` on the client machine
- Copy the `/etc/ceph/ceph.conf` from the ceph cluster to the client machine `/etc/ceph/ceph.conf`
- Deploy user keys
- Test access

```bash
# Example Config dir
root@host01:/etc/ceph# ls -la /etc/ceph/
total 28
drwxr-xr-x.   2 root root  105 Feb 19 18:46 .
drwxr-xr-x. 126 root root 8192 Feb 19 18:39 ..
-rw-r-----.   1 root qemu  143 Feb 19 18:41 ceph.client.block.keyring
-rw-r-----.   1 root qemu  145 Feb 19 18:42 ceph.client.libvirt.keyring
-rw-r--r--.   1 root root  286 Feb 19 18:45 ceph.conf
-rw-r--r--.   1 root root   92 Jan 10 00:00 rbdmap

# Listing available images on pool block01
# Here I was testing with both users, docs make it look like libvirt user
# has to be named libvirt so I creaed both to test.
rbd -n client.libvirt list block01
rbd -n client.block list block01

# Create a disk image if none exist
rbd create testing-libvirt --size 4096 -p block01
```

### rbds

Distribute client key first.

```bash
apt install ceph-common

# Get client keys fo Ceph

# Map a rados device, pool/image_name 
rbd device map rados-pool01/test01

# Umap
rbd device unmap rados-pool01/test01
```

### Libvirt

Integration with libvirt as a storage pool

#### Create a Libvirt secret for our client.libvirt key


```bash
# Get a UUID
UUID=$(uuidgen)

# Create our secret definition XML file
cat > "libvirt-secret.xml" <<EOF
<secret ephemeral='no' private='no'>
  <uuid>${UUID}</uuid>
  <usage type='ceph'>
    <name>client.libvirt secret</name>
  </usage>
</secret>
EOF

# Define our secret in libvirt
virsh secret-define --file libvirt-secret.xml

# Populate the secret w/ content
# this assumes we have client.admin credentials on the host
virsh secret-set-value --secret "${UUID}" --base64 "$(ceph auth get-key client.libvirt)"

# Create our pool definition
<pool type="rbd">
  <name>block01</name>
  <source>
    <name>block01</name>
    <host name='ceph01.tkclabs.io,ceph02.tkclabs.io,ceph03.tkclabs.io'/>
    <auth username='libvirt' type='ceph'>
      <secret uuid='${UUID}'/>
    </auth>
  </source>
</pool>

# Define the pool, set it to autostart, and start it
virsh pool-define rbd-block01.xml
virsh pool-autostart block01
virsh pool-start block01
```
