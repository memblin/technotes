# Ez GHAR Setup

```bash
# Create an unprivileged user to run the Action Runner service
useradd -c 'Github Actions user' -m -s /usr/bin/bash ghrunner

# Install Required Packages
apt-get update
apt-get install -y sudo ca-certificates curl

# Create runners dir and change it's ownership
mkdir /opt/runners
chown -R ghrunner:ghrunner /opt/runners

# Add Docker's Official GPG Key
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker's Apt repository source
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add ghrunner to sudoers
cat <<EOF > /etc/sudoers.d/ghrunner
ghrunner ALL=(ALL) NOPASSWD: ALL
EOF
chmod 0440 /etc/sudoers.d/ghrunner
chown root:root /etc/sudoers.d/ghrunner

# Add ghrunner to docker group
usermod -aG docker ghrunner

# Become ghrunner
su - ghrunner

# Change directory into the /opt/runners directory
cd /opt/runners

# Create our org directory and descend into it
mkdir TKC-Labs && cd $_

# Download the lastest runner package
curl -O -L https://github.com/actions/runner/releases/download/v2.330.0/actions-runner-linux-x64-2.330.0.tar.gz

# Extract the runner package
tar xzf ./actions-runner-linux-x64-2.330.0.tar.gz

# Install dependencies
sudo ./bin/installdependencies.sh

# Configure runner
#
# Get token from the organization or repo settings action runner provisiniong page
 export GH_TOKEN=GOOD_TOKEN

# Configure runner
./config.sh --unattended --url https://github.com/TKC-Labs --token $GH_TOKEN --name ghar-1-tkclabs --labels debian13,vm

# Install the systemd runner service
sudo ./svc.sh install

# Start the runner service
sudo ./svc.sh start
```

