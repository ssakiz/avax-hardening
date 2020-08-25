# How to Secure Your AVAX Node
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
sudo apt-get update -qq
sudo apt-get upgrade -y -qq
sudo apt-get autoremove -y
sudo apt-get autoclean -y


# Install unattended-upgrades
# sudo apt-get install unattended-upgrades apt-listchanges -y -qq
# Enable unattended upgrades

sudo apt-get install mailutils wget curl -y
sudo apt-get install cron-apt
sudo reboot
```




# Step 1 - Creating your standard user
### create admin user
```bash
# Settings
USERNAME=avaxops
PASSWORD=XXXXXXXX   # Change this before run
NODE_PUBLIC_IP=XX.XX.XX.XX # Change this before run
SYS_ADMIN_EMAIL=ssakiz@gmail.com

sudo deluser --remove-home ${USERNAME}
sudo useradd -m -s /bin/bash -g sudo -c "Administrative User" "${USERNAME}"
sudo sh -c "echo \"${USERNAME} ALL=(ALL) NOPASSWD:ALL\" >> /etc/sudoers"

sudo usermod -aG systemd-journal ${USERNAME}

echo "${USERNAME}:${PASSWORD}" | sudo chpasswd 

# Create & copy ssh key in your local linux server to NODE Server
ssh-keygen -b 4096
ssh-copy-id ${USERNAME}@<hostname>

# ssh to NODE server via ssh key without passwd.
ssh ${USERNAME}@<hostname>

```


# set log rotation
sudo journalctl --vacuum-size=1G
```


### service definition
```bash

# installl ava node with gecko user / upgrade 
sudo systemctl stop gecko.service
sudo useradd -r -m gecko
sudo su - gecko
cd /home/gecko
unlink avalanche
rm -rf avalanche-0.*
rm -rf ./gecko/db/
wget https://github.com/ava-labs/gecko/releases/download/v0.6.2/avalanche-linux-0.6.2.tar.gz
tar xvfz avalanche-linux-0.6.2.tar.gz
ln -s avalanche-0.6.2/avalanche avalanche
rm -rf *.tar.gz
exit


# create and start gecko service


sudo bash -c 'cat << EOF > /etc/systemd/system/gecko.service
[Unit]
Description=AVA node
Wants=network-online.target
After=network.target network-online.target

[Service]
User=gecko
Group=gecko
WorkingDirectory=/home/gecko
ExecStart=/home/gecko/avalanche
KillMode=process
KillSignal=SIGINT
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=avalanche

Restart=on-failure
RestartSec=10

# Hardening measures
####################
# Provide a private /tmp and /var/tmp.
PrivateTmp=true

# Mount /usr, /boot/ and /etc read-only for the process.
ProtectSystem=full

# Deny access to /home, /root and /run/user
# ProtectHome=true

# Disallow the process and all of its children to gain
# new privileges through execve().
NoNewPrivileges=true

# Use a new /dev namespace only populated with API pseudo devices
# such as /dev/null, /dev/zero and /dev/random.
PrivateDevices=true

# Deny the creation of writable and executable memory mappings.
MemoryDenyWriteExecute=true

[Install]
WantedBy=multi-user.target

EOF'


sudo systemctl daemon-reload
sudo systemctl enable gecko.service
sudo systemctl start gecko.service
sudo systemctl stop gecko.service
sudo systemctl status gecko.service
sudo journalctl -u gecko
sudo journalctl -u gecko -f
```


### Controlling services

- Start: `sudo systemctl start gecko.service>`
- Restart: `sudo systemctl restart gecko.service>`
- Stop: `sudo systemctl stop gecko.service`
- Status: `sudo systemctl status gecko.service`



### Security
```bash
# Setup firewall
sudo apt-get install ufw -y  &&
sudo ufw default deny incoming &&
sudo ufw default allow outgoing &&
sudo ufw allow ssh/tcp && 
sudo ufw allow http/tcp &&
sudo ufw allow 22/tcp &&
sudo ufw logging on  &&
sudo ufw enable &&
sudo ufw status verbose &&
sudo systemctl enable ufw

# Intrusion detection using Fail2Ban
# After 10 failed login attempts from a single IP address, it blocks that IP address from trying to login again for 10 minutes.
sudo apt-get install fail2ban -y -qq
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo systemctl fail2ban status
sudo fail2ban-client status
sudo tail -f /var/log/fail2ban.log

sudo systemctl restart fail2ban


# Secure shared memory
echo "tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0" | sudo tee -a /etc/fstab
```

### Edit /etc/sysctl.conf
```bash
wget https://raw.githubusercontent.com/ssakiz/avax-hardening/master/sysctl.conf
sudo cp /etc/sysctl.conf /etc/sysctl.conf.backup
sudo cp sysctl.conf /etc/sysctl.conf
sudo sysctl -e -p
```



## SSH access (optional)

```bash

sudo cp  /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

# disable root login, disable password auth
sudo sed -i 's/^PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config &&
sudo sed -i 's/^LoginGraceTime .*/LoginGraceTime 60/' /etc/ssh/sshd_config &&
sudo sed -i 's/^MaxStartups .*/MaxStartups 2/' /etc/ssh/sshd_config &&
sudo sed -i 's/^StrictModes .*/StrictModes yes/' /etc/ssh/sshd_config &&
sudo sed -i 's/^PasswordAuthentication .*/PasswordAuthentication no/' /etc/ssh/sshd_config &&
sudo sed -i 's/^PermitEmptyPasswords .*/PermitEmptyPasswords no/' /etc/ssh/sshd_config &&
sudo service ssh reload
```


## Tuning (optional)

```bash


# tuning 
sudo bash -c 'cat << EOF >> /etc/security/limits.conf
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
EOF'

reboot

#  Adding more swap
sudo cat /proc/swaps
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo swapon -s

sudo cat << 'EOF' | sudo tee -a /etc/fstab
/swapfile  none  swap  sw  0 0
EOF

# Install rkhunter and do an initial file scan.
sudo apt -y install rkhunter
sudo rkhunter --propupd

```


## Time synchonization 

```bash

###  set correct timezone & logi
```bash
sudo timedatectl set-timezone Europe/Istanbul
sudo timedatectl status
                      Local time: Fri 2020-08-21 20:10:06 +03
                  Universal time: Fri 2020-08-21 17:10:06 UTC
                        RTC time: Fri 2020-08-21 20:10:06
                       Time zone: Europe/Istanbul (+03, +0300)
       System clock synchronized: yes
systemd-timesyncd.service active: yes
                 RTC in local TZ: yes

```
