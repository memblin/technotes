# Libvirt Notes

## Clone an existing VM

```bash
# Clone an existing domain (vm) to a new named domain (vm) providing the filename
virt-clone --original template-fedora39 --name nats01.tkclabs.io --file /var/lib/libvirt/images/nats01.tkclabs.io.qcow2

# Then start it up
virsh start nats01.tkclabs.io

```
