# SaltStack: salt

## Bootstrap installation cheat sheet

Get the latest release URL from https://github.com/saltstack/salt-bootstrap/releases

```bash
# Downloading using wget
wget https://github.com/saltstack/salt-bootstrap/releases/download/v2024.12.12/bootstrap-salt.sh

# Downloading using curl
curl -o bootstrap-salt.sh -L https://github.com/saltstack/salt-bootstrap/releases/download/v2024.12.12/bootstrap-salt.sh

# Set permissions
chmod 0700 ./bootstrap-salt.sh

# Run it for purpose
#
# -A : Set the initial master fqdn for the minion to use
# -i : Set the initial minion_id for the minion to use
# -M : Install salt-master
# -W : Install salt-api
# -N : Do NOT install the salt-minion
# -X : Do not start salt services after install

# Install salt-master, salt-minion, & salt-api
./bootstrap-salt.sh -A 127.0.0.1 -i $(hostname -f) -M -W stable 3006.9

# Install only salt-master, no salt-minion or salt-api
./bootstrap-salt.sh -M -N -X stable 3006.9

# Install salt-minion only
./bootstrap-salt.sh -A salt01.tkclabs.io -i $(hostname -f) stable 3006.9
```
