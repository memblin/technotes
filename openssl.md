# OpenSSL


## Quick Self-signed Cert

Use Case:

A quick throw-away self-signed certificate to use during lab testing or shipped configuration content.

It doesn't attempt subj alt names or anything else fancy, just here's a cert and key with the CN provided; in this case `salt.local`

```bash
# Create cert / key pair combo valid for 10 years for the CN, "salt.local"
openssl req -x509 -newkey rsa:4096 -keyout selfsigned.key -out selfsigned.crt -sha256 -days 3650 -nodes -subj '/C=US/ST=Colorado/L=Denver/O=TKC Labs/OU=InfraTesting/CN=salt.local'
```