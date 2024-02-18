# Debian Config Notes

Generic Debian configuration notes

## Statkc IP Address

```bash
auto enp0s3
iface enp0s3 inet static
        address 192.168.1.240/24
        network 192.168.1.0
        broadcast 192.168.1.255
        gateway 192.168.1.1fa
        dns-nameservers 8.8.8.8
```

## SSH Host Keys

```bash
# default host keys 
/etc/ssh/ssh_host_ecdsa_key
/etc/ssh/ssh_host_ecdsa_key.pub
/etc/ssh/ssh_host_ed25519_key
/etc/ssh/ssh_host_ed25519_key.pub
/etc/ssh/ssh_host_rsa_key
/etc/ssh/ssh_host_rsa_key.pub

# When removed sshd will not start and Debian
# will not regenerate them automatically the
# ay EL does when the host keys are missing.
#
# Regenerate with
dpkg-reconfigure openssh-server

# Using this command with host keys present
# may change the host keys.
```

### /etc/rc.local to set SSH host keys

Add this to `/etc/rc.local`:

```bash
#!/usr/bin/bash

# Check if any of the SSH host keys are missing
if [ ! -f /etc/ssh/ssh_host_ecdsa_key ] || \
   [ ! -f /etc/ssh/ssh_host_ed25519_key ] || \
   [ ! -f /etc/ssh/ssh_host_rsa_key ]; then
    
    # If any key is missing, run dpkg-reconfigure openssh-server
    echo "Regenerating OpenSSH Host Keys.."
    dpkg-reconfigure openssh-server
fi
```

Then run

```bash
chmod 0700 /etc/rc.local

# If it doesn't run automatically might need to
# enable rc-local
systemctl enable rc-local
```

