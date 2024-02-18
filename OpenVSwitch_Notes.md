# OpenVSwitch Notes

Notes for OpenVSwitch coniguration

## Libvirt Setup

On a system with 4 NICs for example.

- eno1 : Management Interface (1Gb)
- eno2 : VM Uplink Bridge (1Gb)
- eno3 : Storage Bridge (10Gb)
- eno4 : Clusters Bridge (10Gb)

*Matching the bridge number to the VLAN is style and required but self-documenting systems are easier to manage.*

The following examples show how I add VLAN10 to my Libvirt hosts. The process can be repeated for multiple VLANs or other Interfaces.

### VLAN and OVS Configuration

```bash
# Create an OVS bridge for VLAN10
ovs-vsctl add-br br10

# Create the VLAN10 interface with Network Manager CLI
nmcli connection add type vlan con-name eno2.10 ifname eno2.10 dev eno2 id 10

# Disable DHCP on the VLAN10 interface with Network Manager CLI
nmcli connection modify eno2.10 ipv4.method disabled ipv6.method disabled

# Bring up the VLAN10 Interface with Network Manager CLI
nmcli connection up eno2.10

# Add the VLAN10 Interface to the OVS bridge
ovs-vsctl add-port br10 eno2.10
```

### Libvirt Configuration - VLAN

Create Libvirt network definition file for the VLAN10 bridge.

Use the `uuidgen` command to create a new UUID for each network you create.

Create a file with content similar to this:

```xml
<network>
  <name>vlan10</name>
  <uuid>7c84b6a1-21a5-44c2-9631-4f3367cd94f5</uuid>
  <forward mode='bridge'/>
  <bridge name='br10'/>
  <virtualport type='openvswitch'/>
</network>
```

For this example we use the file name `vlan10.xml`, once it is written to disk we can define our network and mark it for auto-start.

```bash
# Import / define the network
virsh net-define vlan10.xml

# Configure network to auto-start when libvirt starts
virsh net-autostart vlan10
```

### Physical port and OVS Configuration

No VLAN on my storage network so we use the base interface and call it bridge 210

```bash
# Create an OVS bridge for VLAN10
ovs-vsctl add-br br210

# Disable DHCP on the physical interface with Network Manager CLI
nmcli connection modify eno4 ipv4.method disabled ipv6.method disabled

# Bring up the physical Interface with Network Manager CLI
nmcli connection up eno4

# Add the physical Interface to the OVS bridge
ovs-vsctl add-port br210 eno4
```

### Libvirt Configuration - physical port

```bash
root@host01:~/ovs-networks# cat 10gbit.xml 
<network>
  <name>10gbit</name>
  <uuid>6e12140e-6dbf-4c95-8e1b-3d8626894873</uuid>
  <forward mode='bridge'/>
  <bridge name='br210'/>
  <virtualport type='openvswitch'/>
</network>
root@host01:~/ovs-networks# virsh net-define 10gbit.xml 
Network 10gbit defined from 10gbit.xml

root@host01:~/ovs-networks# virsh net-autostart 10gbit 
Network 10gbit marked as autostarted

root@host01:~/ovs-networks# 
```

## Set an IP directly on the bridge interface

- Could not get network manager to allocate one, setting directly works.

```bash
# Can we set an IP directly since Network Manager is being a jerk?
ip addr add 100.64.15.21/24 dev br210
ip link set br210 up
```