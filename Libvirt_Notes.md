# Libvirt Notes

## Clone an existing VM

```bash
# Clone an existing domain (vm) to a new named domain (vm) providing the filename
virt-clone --original template-fedora39 --name nats01.tkclabs.io --file /var/lib/libvirt/images/nats01.tkclabs.io.qcow2

# Then start it up
virsh start nats01.tkclabs.io

```

## QEMU-Images in Libvirt: QCOW2

The default type of disk image used when issuing a `virsh attach-disk` command is RAW.

Mounting a QCOW2 disk image as a RAW image will not work and you will see a much smaller disk in the VM.

Instead, pass the appropriate driver options when mounting a QCOW2 disk.

```bash
# Create a 20GB Disk Image
qemu-img create -f /var/lib/libvirt/images/debian12.local.disk2.qcow2 20G

# Attach that disk image to the VM with appropriate driver
virsh attach-disk debian12.local /var/lib/libvirt/images/debian12.local.disk2.qcow2 vdb --driver qemu --subdriver qcow2 --targetbus virtio --persistent
```
