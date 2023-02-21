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

3. Harden SERVER 

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


# Install and Configure Wireguard on CLIENT and SERVER

- Install wireguard on both client and server

`sudo apt install wireguard`

### On SERVER 
edit control file and enable ipv4 forwarding
```
sudo nano /etc/sysctl.conf

#comment
net.ipv4.ip_forward=1

#apply changes
sudo sysctl -p
sudo sysctl --system
```

### On BOTH client and server

generate a public private keypair each

```
#become root
sudo -i
cd /etc/wireguard
wg genkey | tee privatekey | wg pubkey > publickey
chmod 400 privatekey
```

### On the SERVER

create a config file `wg0.conf`

```
cat privatekey  #copy the result

#create a conf file
nano wg0.conf
```

Add this template, and replace `ens3`with the name of your network device (use: `ip a` to find it) 

```
[Interface]
PrivateKey = <privatekey of the server> 
Address = 10.9.9.1
ListenPort = 51820
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE

[Peer]
PublicKey = <publickey of the client>
AllowedIPs = 10.9.9.2/32


```
Harden file permissions
`chmod 600 wg0.conf`


Allow incoming udp port `51820`
`ufw allow 51820/udp`


### On the CLIENT 
`nano wg0.conf`
 
```
[Interface]
PrivateKey = <privatekey of the client> 
Address = 10.9.9.2   
DNS = 1.1.1.1 #Cloudflare DNS server. Could be any public DNS server, eg. Google 8.8.8.8


[Peer]
PublicKey = < publickey of the server>
Endpoint = <external public IP of the server>:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

Harden file permissions
`chmod 600 wg0.conf`

# Start up Wireguard on CLIENT and SERVER, and test connection

- On the SERVER
`wg-quick up wg0`


- On the Client 
`wg-quick up wg0`

then test the Client'd public IP
`curl 'https://checkip.dedyn.io'` 

this should resolve into the public IP of the SERVER!!


# Enable wireguard at startup

On BOTH the CLIENT and SERVER
```
#shut down the service wg0
wg-quick down wg0

#then add it to the system startup 
systemctl enable --now wg-quick@wg0.service

#check the status
systemctl status wg-quick@wg0.service #should not display errors
``` 
