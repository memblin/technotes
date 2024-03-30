# Generic GPG Notes


```bash
# Decrypt a set of private keys from an ascii armored file using it's passphrase
# the "--pinentry-mode loopback" can help when working system users, odd TTYs,
# and other times when it's difficult to pop a passphrase input mechanism.
gpg --pinentry-mode loopback --decrypt tkclabs-salt-ssh-gpg-privkey.asc | gpg --homedir ~/.gnupg --import
```