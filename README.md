# vps-lightningnode-wg
HOWTO set up a tunnel from local Bitcoin/Lightning node to a personal VPS wireguard server and tunnel all traffic

Credits: https://www.youtube.com/watch?v=TzKj5garlIE

# Prepare the VPS Server

1. Update server, and install packages for hardening
Log in as root. Else use `sudo`
```
apt update && apt upgrade
apt autoremove
apt install ufw
apt install fail2ban
```

2. Create user `wg` 
```
adduser wg
usermod -aG sudo wg
mkdir /home/wg/.ssh
cat /root/.ssh/authorized_keys > /home/wg/.ssh/authorized_keys  #assuming you had already your personal ssh key in root's authorized_keys file, else
# nano /home/wg/.ssh/authorized_keys # and paste it here
chmod 600 /home/wg/.ssh/authorized_keys 
chown wg:wg /home/wg/.ssh/authorized_keys 
chmod 700 /home/wg/.ssh/
chown wg:wg /home/wg/.ssh/
```

3, Harden server 

- Edit sshd_config `nano /etc/ssh/sshd_config 
- Uncomment and change these settings
```
PermitRootLogin no
AllowUsers wg

AuthorizedKeysFile      .ssh/authorized_keys .ssh/authorized_keys2

PasswordAuthentication no
PermitEmptyPasswords no
```

- Restart the ssh daemon 
`systemctl restart sshd.service`


Now open a new ssh from your local machine, with the `wg` user 
`ssh wg@<<vps_server_ip>`

Configure and enable User firewall `ufw`
```
sudo ufw allow ssh
sudo ufw enable
```

Configure and start `fail2ban`
```
cd /etc/fail2ban/
sudo cp jail.conf jail.local
sudo nano jail.local
```

and change/set
``` 
bantime  = 60m

maxretry = 3

```

Reload fail2ban
`sudo systemctl reload fail2ban.service`

Check status
`sudo systemctl status fail2ban.service`

If there is an error `ERROR   Failed during configuration: Have not found any log file for sshd jail` then you need to manually create the sshd log file, which by default is in `/var/log/auth.log`
`sudo touch /var/log/auth.log` 

