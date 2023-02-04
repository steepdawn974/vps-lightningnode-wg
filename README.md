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
```
sudo systemctl status fail2ban.service
```

If there is an error `ERROR   Failed during configuration: Have not found any log file for sshd jail` then you need to manually create the sshd log file, which by default is in `/var/log/auth.log`

```sudo touch /var/log/auth.log```


4. Log out from the server, add the ssh login conncetion to your local PCs ssh/config for convenience
```
echo "Host  wgserver
Hostname <<your_vps_servers_public_ip_Address>>
User  wg" >> .ssh/config
```

Test is by logging in
`ssh wgserver`


# Install and Confiugure WQireguard on Server and Client

- Install wireguard on both client and server machines

`sudo apt install wireguard`

On SERVER edit control file and enable ipv4 forwarding
```
sudo nano /etc/sysctl.conf

#comment
net.ipv4.ip_forward=1

#apply changes
sudo sysctl -p
sudo sysctl --system
```

On BOTH client and server generate a public private keypair

```
#become root
sudo -i
cd /etc/wireguard
wg genkey | tee privatekey | wg pubkey > publickey
chmod 400 privatekey
```

On SERVER private key

```
[Interface]
PrivateKey = 
Address = 10.0.0.1
ListenPort = 51820
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat4 -A POSTROUTING -o ens3 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat4 -D POSTROUTING -o ens3 -j MASQUERADE

[Peer]
PublicKey = 
AllowedIPs = 10.0.0.2/32

```


