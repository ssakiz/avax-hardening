# How to Secure Your AVAX Node
avax-hardening



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

sudo apt-get install mailutils wget curl jq net-tools -y
sudo apt-get install cron-apt
sudo reboot
```

167.114.36.162


# Step 1 - Creating your standard user
### create admin user
```bash
# Settings
HOSTNAME=burcusan
sudo hostname $HOSTNAME

USERNAME=avaxops
PASSWORD=XXXXXXXX   # Change this before run
NODE_PUBLIC_IP=XX.XX.XX.XX # Change this before run
SYS_ADMIN_EMAIL=ssakiz@gmail.com

sudo deluser --remove-home ${USERNAME}
sudo useradd -m -s /bin/bash -g sudo -c "Administrative User" "${USERNAME}"
sudo sh -c "echo \"${USERNAME} ALL=(ALL) NOPASSWD:ALL\" >> /etc/sudoers"

sudo usermod -aG systemd-journal ${USERNAME}

echo "${USERNAME}:${PASSWORD}" | sudo chpasswd 
ifco
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

# installl ava node with avalanchego user / upgrade 
sudo systemctl stop avalanchego.service
sudo useradd -r -m avalanchego
sudo su - avalanchego
cd /home/avalanchego
unlink latest
rm -rf avalanche-0.*
# rm -rf ./avalanchego/db/
wget https://github.com/ava-labs/avalanchego/releases/download/v1.0.1/avalanchego-linux-1.0.1.tar.gz
tar xvfz avalanchego-linux-1.0.1.tar.gz
ln -sf avalanche-1.0.1 latest
rm -rf *.tar.gz
exit


# create and start avalanchego service


sudo bash -c 'cat << EOF > /etc/systemd/system/avalanchego.service
[Unit]
Description=AVAX node
Wants=network-online.target
After=network.target network-online.target

[Service]
User=avalanchego
Group=avalanchego
WorkingDirectory=/home/avalanchego/latest
ExecStart=/home/avalanchego/latest/avalanchego
KillMode=process
KillSignal=SIGINT
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=avalanchego

Restart=on-failure
RestartSec=10

# Hardening measures
####################
# Provide a private /tmp and /var/tmp.
# PrivateTmp=true

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
sudo systemctl enable avalanchego.service
sudo systemctl start avalanchego.service
sudo systemctl stop avalanchego.service
sudo systemctl status avalanchego.service
sudo journalctl -u avalanchego
sudo journalctl -u avalanchego -f
```


### Controlling services

- Start: `sudo systemctl start avalanchego.service>`
- Restart: `sudo systemctl restart avalanchego.service>`
- Stop: `sudo systemctl stop avalanchego.service`
- Status: `sudo systemctl status avalanchego.service`



### Security
```bash
# Setup firewall
sudo apt-get install ufw -y  &&
sudo ufw default deny incoming &&
sudo ufw default allow outgoing &&
sudo ufw allow ssh/tcp && 
# sudo ufw allow http/tcp &&
sudo ufw allow 22/tcp &&
# sudo ufw allow proto tcp from 0.0.0.0/0 port 9651 to any &&
sudo ufw allow 9651/tcp &&
sudo ufw logging on  &&
sudo ufw enable &&
sudo ufw status verbose &&
sudo systemctl enable ufw
sudo tail -f /var/log/ufw.log

# Intrusion detection using Fail2Ban
# After 10 failed login attempts from a single IP address, it blocks that IP address from trying to login again for 10 minutes.
sudo apt-get install fail2ban -y -qq
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo systemctl fail2ban status
sudo fail2ban-client status sshd
sudo fail2ban-client status
sudo tail -f /var/log/fail2ban.log

sudo systemctl restart fail2ban


# Secure shared memory
sudo echo "tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0" | sudo tee -a /etc/fstab
```

### Edit /etc/sysctl.conf
```bash
wget https://raw.githubusercontent.com/ssakiz/avax-hardening/master/sysctl.conf
sudo cp /etc/sysctl.conf /etc/sysctl.conf.backup
sudo cp sysctl.conf /etc/sysctl.conf
sudo sysctl -e -p



sudo bash -c 'cat << EOF >> /etc/default/grub
GRUB_CMDLINE_LINUX="ipv6.disable=1"
GRUB_CMDLINE_LINUX_DEFAULT="ipv6.disable=1"
EOF'	

sudo update-grub

sudo systemctl reboot

```



## SSH access (optional)

```bash

sudo cp  /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

# disable root login, disable password auth
sudo sed -i 's/^PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config &&
sudo sed -i 's/^LoginGraceTime .*/LoginGraceTime 60/' /etc/ssh/sshd_config &&
sudo sed -i 's/^MaxStartups .*/MaxStartups 2/' /etc/ssh/sshd_config &&
sudo sed -i 's/^StrictModes .*/StrictModes yes/' /etc/ssh/sshd_config &&
# sudo sed -i 's/^PasswordAuthentication .*/PasswordAuthentication no/' /etc/ssh/sshd_config &&
sudo sed -i 's/^PermitEmptyPasswords .*/PermitEmptyPasswords no/' /etc/ssh/sshd_config &&
sudo service ssh reload
```

## Configure Multi-Factor Authentication using Google Authenticator 
```bash

# Installing the Google PAM Module
sudo apt-get update
sudo apt-get install libpam-google-authenticator -y

# Configuring 2FA for a User
google-authenticator

# Configuring ssh
sudo vi /etc/ssh/sshd_config

ChallengeResponseAuthentication yes

UseDNS no
AddressFamily inet
SyslogFacility AUTHPRIV
PermitRootLogin no
PasswordAuthentication yes

sudo vi /etc/pam.d/sshd 

# Standard Un*x password updating.
@include common-password
auth required pam_google_authenticator.so

sudo systemctl restart sshd.service
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

sudo reboot

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
