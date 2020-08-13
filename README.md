# avax-hardening
avax-hardening



# Setting up a full AVAX node
### A step-by-step guide for setting up the perfect Bitcoin box on Ubuntu. Including:
* [Preparing the environment](https://github.com/bitembassy/home-node/blob/master/README.md#preparing-the-environment)
* Services:
  * [Tor](https://github.com/bitembassy/home-node/blob/master/README.md#tor)
  * [Bitcoin Core](https://github.com/bitembassy/home-node/blob/master/README.md#bitcoin-core)

* [Adding to startup](https://github.com/bitembassy/home-node/blob/master/README.md#startup-services)
* [Updating software](https://github.com/bitembassy/home-node/blob/master/README.md#updating-software)

> Note: a dedicated, always-online computer and a fresh Ubuntu 18.04 install are recommended. Some of the settings may interfere with existing software.



## Preparing the environment

### Updates
```bash
# Fetch the list of available updates, upgrade current
sudo apt update &&
sudo apt upgrade -y &&
sudo apt autoremove -y
```




# CREATE ADMIN USER
### create admin user
```bash
USERNAME=admin
PASSWORD=Akhisar2014.

sudo deluser --remove-home ${USERNAME}
sudo useradd -m -s /bin/bash -g sudo -c "Administrative User" "${USERNAME}"
sudo sh -c "echo \"${USERNAME} ALL=(ALL) NOPASSWD:ALL\" >> /etc/sudoers"

sudo usermod -aG systemd-journal ${USERNAME}

echo "${USERNAME}:${PASSWORD}" | sudo chpasswd 
```


### Security
```bash
# Setup firewall
sudo apt-get install ufw  &&
sudo ufw default deny incoming &&
sudo ufw default allow outgoing &&
sudo ufw allow ssh/tcp && 
sudo ufw allow http/tcp &&
sudo ufw allow 30303/tcp &&
sudo ufw logging on  &&
sudo ufw enable &&
sudo ufw status verbose &&
sudo systemctl enable ufw

# install fail2ban
sudo apt-get install fail2ban
sudo service fail2ban status
sudo fail2ban-client status


# Secure shared memory
echo "tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0" | sudo tee -a /etc/fstab
```

Edit `/etc/sysctl.conf`
```bash
# IP Spoofing protection
sudo sed -i 's/#net.ipv4.conf.all.rp_filter=1/net.ipv4.conf.all.rp_filter=1/' /etc/sysctl.conf &&
sudo sed -i 's/#net.ipv4.conf.default.rp_filter=1/net.ipv4.conf.default.rp_filter=1/' /etc/sysctl.conf &&

# Ignore ICMP broadcast requests
sudo sed -i 's/#net.ipv4.icmp_echo_ignore_broadcasts = 1/net.ipv4.icmp_echo_ignore_broadcasts = 1/' /etc/sysctl.conf &&

# Disable source packet routing
sudo sed -i 's/#net.ipv4.conf.all.accept_source_route = 0/net.ipv4.conf.all.accept_source_route = 0/' /etc/sysctl.conf &&
sudo sed -i 's/#net.ipv6.conf.all.accept_source_route = 0/net.ipv6.conf.all.accept_source_route = 0/' /etc/sysctl.conf &&
sudo sed -i 's/#net.ipv4.conf.default.accept_source_route = 0/net.ipv4.conf.default.accept_source_route = 0/' /etc/sysctl.conf &&
sudo sed -i 's/#net.ipv6.conf.default.accept_source_route = 0/net.ipv6.conf.default.accept_source_route = 0/' /etc/sysctl.conf &&

# Ignore send redirects
sudo sed -i 's/#net.ipv4.conf.all.send_redirects = 0/net.ipv4.conf.all.send_redirects = 0/' /etc/sysctl.conf &&
sudo sed -i 's/#net.ipv4.conf.default.send_redirects = 0/net.ipv4.conf.default.send_redirects = 0/' /etc/sysctl.conf &&


# Block SYN attacks
sudo sed -i 's/#net.ipv4.tcp_syncookies=1/net.ipv4.tcp_syncookies=1/' /etc/sysctl.conf &&
sudo sed -i 's/#net.ipv4.tcp_max_syn_backlog = 2048/net.ipv4.tcp_max_syn_backlog = 2048/' /etc/sysctl.conf &&
sudo sed -i 's/#net.ipv4.tcp_synack_retries = 2/net.ipv4.tcp_synack_retries = 2/' /etc/sysctl.conf &&
sudo sed -i 's/#net.ipv4.tcp_syn_retries = 5/net.ipv4.tcp_syn_retries = 5/' /etc/sysctl.conf &&

# Log Martians
net.ipv4.conf.all.log_martians = 1
sudo sed -i 's/#net.ipv4.conf.all.log_martians = 1/net.ipv4.conf.all.log_martians = 1/' /etc/sysctl.conf &&
sudo sed -i 's/#net.ipv4.icmp_ignore_bogus_error_responses = 1/net.ipv4.icmp_ignore_bogus_error_responses = 1/' /etc/sysctl.conf &&


# Ignore ICMP redirects
sudo sed -i 's/#net.ipv4.conf.all.accept_redirects = 0/net.ipv4.conf.all.accept_redirects = 0/' /etc/sysctl.conf &&
sudo sed -i 's/#net.ipv6.conf.all.accept_redirects = 0/net.ipv6.conf.all.accept_redirects = 0/' /etc/sysctl.conf &&
sudo sed -i 's/#net.ipv4.conf.default.accept_redirects = 0/net.ipv4.conf.default.accept_redirects = 0/' /etc/sysctl.conf &&
sudo sed -i 's/#net.ipv6.conf.default.accept_redirects = 0/net.ipv6.conf.default.accept_redirects = 0/' /etc/sysctl.conf &&

# Ignore Directed pings
sudo sed -i 's/#net.ipv4.icmp_echo_ignore_all = 1/net.ipv4.icmp_echo_ignore_all = 1/' /etc/sysctl.conf &&
```
Copy [this](https://github.com/bitembassy/home-node/raw/master/misc/sysctl.conf), paste at the bottom and save.


### Common dependencies

```bash
sudo apt install -y  git &&
```

## Bitcoin Core

### Installing
```bash
# Create dir for installation files
mkdir -p ~/bitcoin-installation && cd ~/bitcoin-installation && rm -rf * &&

# Download binaries
wget https://bitcoincore.org/bin/bitcoin-core-0.17.1/bitcoin-0.17.1-x86_64-linux-gnu.tar.gz &&

# Download signature
wget https://bitcoincore.org/bin/bitcoin-core-0.17.1/SHA256SUMS.asc &&

# Verify signature - should see "Good signature from Wladimir J. van der Laan (Bitcoin Core binary release signing key) <laanwj@gmail.com>"
gpg --verify SHA256SUMS.asc &&

# Verify the binary matches the signed hash in SHA256SUMS.asc - should see "bitcoin-0.17.1-x86_64-linux-gnu.tar.gz: OK"
grep bitcoin-0.17.1-x86_64-linux-gnu.tar.gz SHA256SUMS.asc | sha256sum -c - &&

# Unpack binaries
tar xvf bitcoin-0.17.1-x86_64-linux-gnu.tar.gz &&

# Install binaries system-wide (requires password)
sudo cp bitcoin-0.17.1/bin/* /usr/bin
```


## Startup services

> Note: If you already have the services running from the terminal, stop them before starting the systemd service.

### Stage 1: Bitcoin, EPS, btc-rpc-explorer

```bash
# Download home-node repo
git clone https://github.com/bitembassy/home-node ~/home-node && cd ~/home-node &&

# Verify signature - should see "Good signature from Nadav Ivgi <nadav@shesek.info>"
git verify-commit HEAD &&

# Copy service init files
sudo cp init/{bitcoind,eps,btc-rpc-explorer}.service /etc/systemd/system/ &&

# Reload systemd, enable services, start them
sudo systemctl daemon-reload &&
sudo systemctl start bitcoind && sudo systemctl enable bitcoind &&
sudo systemctl start eps && sudo systemctl enable eps &&
sudo systemctl start btc-rpc-explorer && sudo systemctl enable btc-rpc-explorer
```

### Controlling services

- Start: `sudo systemctl start <name>`
- Restart: `sudo systemctl restart <name>`
- Stop: `sudo systemctl stop <name>`
- Status: `sudo systemctl status <name>`

The services names are: `bitcoind`,`lightningd`,`btc-rpc-explorer`,`eps` and `spark-wallet`



## SSH access (optional)

```bash


# disable root login, disable password auth
sudo sed -i 's/^PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config &&
sudo sed -i 's/^PasswordAuthentication .*/PasswordAuthentication no/' /etc/ssh/sshd_config &&
sudo sed -i 's/^PermitEmptyPasswords .*/PermitEmptyPasswords no/' /etc/ssh/sshd_config &&
sudo service ssh reload


sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak && 
sudo sed -i 's/Port 22/Port '$MY_SERVER_SSH_PORT'/' /etc/ssh/sshd_config &&
sudo sed -i 's/#Port '$MY_SERVER_SSH_PORT'/Port '$MY_SERVER_SSH_PORT'/' /etc/ssh/sshd_config &&
sudo sed -i 's/X11Forwarding yes/X11Forwarding no/' /etc/ssh/sshd_config &&
sudo sed -i 's/AcceptEnv LANG LC_*/AcceptEnv LANG LC_* GIT_/' /etc/ssh/sshd_config &&
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config &&
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config &&
sudo sed -zi 's/ClientAliveInterval 60\|$/ClientAliveInterval 60\n/' /etc/ssh/sshd_config &&
sudo sed -zi 's/ClientAliveCountMax 600\|$/ClientAliveCountMax 600\n/' /etc/ssh/sshd_config &&
sudo sed -zi 's/AllowAgentForwarding no\|$/AllowAgentForwarding yes/' /etc/ssh/sshd_config

```


## Tuning (optional)

```bash


# disable root login, disable password auth
cat >> /etc/security/limits.conf <<'EOF'
*               hard    core            128000
root            hard    core            128000
*               soft    core            128000
root            soft    core            128000
*               hard    nproc           10000
root            hard    nproc           10000
*               soft    nproc           10000
root            soft    nproc           10000
*               hard    memlock         32000 
root            hard    memlock         32000 
*               soft    memlock         32000 
root            soft    memlock         32000 
*               hard    nofile          128000 
root            hard    nofile          128000 
*               soft    nofile          128000 
root            soft    nofile          128000 
*               hard    msgqueue        8192000 
root            hard    msgqueue        8192000 
*               soft    msgqueue        8192000 
root            soft    msgqueue        8192000 
EOF
```





## Time synchonization 

```bash
#Install chrony service:

sudo apt-get install chrony
sudo service chrony status

Add NTP servers:

sudo nano /etc/chrony.conf
Insert ntp servers:

server 0.pool.ntp.org iburst
server 1.pool.ntp.org iburst
server 2.pool.ntp.org iburst
server 3.pool.ntp.org iburst


Check status:

sudo chronyc tracking
apt install chrony -y
sed '1a server ntp.aliyun.com iburst' /etc/chrony/chrony.conf -i
sed '1a server 0.cn.pool.ntp.org iburst' /etc/chrony/chrony.conf -i
sed '1a server ntp1.aliyun.com iburst' /etc/chrony/chrony.conf -i
systemctl restart chronyd

```
