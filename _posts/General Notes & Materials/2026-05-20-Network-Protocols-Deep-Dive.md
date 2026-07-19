---
layout: post
title: "Network Protocols Deep-Dive"
date: 2026-05-20 10:00:00 +0800
categories:
  - General Notes
tags:
  - Networking
  - Protocols
  - Reference
  - Security
author: muhammed
description: A deep-dive reference on the core network protocols — how they work, what ports they use, and their security implications. Built for quick lookup during CTFs, pentests, and cloud security work.
toc: true
pin: false
math: false
mermaid: false
---

## The OSI Model — Quick Reference

| Layer | Number | Name | Protocol Examples |
|-------|--------|------|------------------|
| Application | 7 | Application | HTTP, DNS, FTP, SMTP, SSH |
| Presentation | 6 | Presentation | TLS/SSL, JPEG, ASCII |
| Session | 5 | Session | NetBIOS, RPC |
| Transport | 4 | Transport | TCP, UDP |
| Network | 3 | Network | IP, ICMP, OSPF |
| Data Link | 2 | Data Link | Ethernet, ARP, MAC |
| Physical | 1 | Physical | Cables, Wi-Fi, fiber |

**TCP/IP model** (what actually matters in practice):

| TCP/IP Layer | Maps to OSI | Protocols |
|-------------|-------------|-----------|
| Application | 5, 6, 7 | HTTP, DNS, SSH, FTP, SMTP |
| Transport | 4 | TCP, UDP |
| Internet | 3 | IP, ICMP |
| Network Access | 1, 2 | Ethernet, ARP |

---

## Port Reference — Most Common

| Port | Protocol | Service |
|------|----------|---------|
| 20, 21 | TCP | FTP (data, control) |
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP |
| 53 | TCP/UDP | DNS |
| 67, 68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 |
| 119 | TCP | NNTP |
| 123 | UDP | NTP |
| 135 | TCP | RPC / MSRPC |
| 137–139 | TCP/UDP | NetBIOS |
| 143 | TCP | IMAP |
| 161, 162 | UDP | SNMP |
| 389 | TCP | LDAP |
| 443 | TCP | HTTPS |
| 445 | TCP | SMB |
| 465 | TCP | SMTPS |
| 587 | TCP | SMTP (submission) |
| 631 | TCP | IPP (printing) |
| 636 | TCP | LDAPS |
| 993 | TCP | IMAPS |
| 995 | TCP | POP3S |
| 1433 | TCP | MSSQL |
| 1521 | TCP | Oracle DB |
| 2049 | TCP/UDP | NFS |
| 3306 | TCP | MySQL / MariaDB |
| 3389 | TCP | RDP |
| 5432 | TCP | PostgreSQL |
| 5900 | TCP | VNC |
| 6379 | TCP | Redis |
| 8080, 8443 | TCP | HTTP/HTTPS alt ports |
| 8888 | TCP | Jupyter Notebook |
| 9200 | TCP | Elasticsearch |
| 27017 | TCP | MongoDB |

---

## TCP — Transmission Control Protocol

### How It Works

TCP is **connection-oriented** — a connection must be established before data flows.

**Three-way handshake:**
```
Client          Server
  |--- SYN ------->|   Client wants to connect
  |<-- SYN-ACK ----|   Server acknowledges, also requests
  |--- ACK ------->|   Client acknowledges — connection open
  |=== DATA ======>|
  |--- FIN ------->|   Client done sending
  |<-- FIN-ACK ----|   Server acknowledges close
```

**TCP Flags:**
| Flag | Meaning |
|------|---------|
| SYN | Synchronize — initiate connection |
| ACK | Acknowledge — confirm receipt |
| FIN | Finish — graceful close |
| RST | Reset — abrupt close (port closed, error) |
| PSH | Push — send data immediately without buffering |
| URG | Urgent — prioritize this data |

### Security Implications

- **SYN flood (DoS):** Attacker sends mass SYN packets, server allocates state for each → exhausts memory. Mitigated by SYN cookies.
- **TCP RST injection:** Attacker spoofs RST packet to tear down a connection (used in censorship and DoS).
- **Port scanning:** `SYN` scan (`nmap -sS`) — sends SYN, if `SYN-ACK` returns → port open, then sends RST without completing handshake (stealth).
- **Session hijacking:** Predict/spoof sequence numbers to inject data into an established TCP session.

```bash
# Nmap TCP connect scan (full handshake, logged)
nmap -sT target

# Nmap SYN scan (stealth, half-open)
nmap -sS target

# Grab banner via TCP
nc -nv target 80
```

---

## UDP — User Datagram Protocol

**Connectionless** — no handshake, no guaranteed delivery, no ordering.

**When to use:** Speed over reliability — DNS, DHCP, NTP, video streaming, VoIP, gaming.

**Security implications:**
- **UDP amplification attacks:** Attacker sends small spoofed request to UDP service (DNS, NTP, SSDP) → large response sent to victim. Amplification factor can be 10-500x.
- **UDP port scanning:** No response = open or filtered. ICMP port unreachable = closed. Slower than TCP scanning.

```bash
# Nmap UDP scan (slow, requires root)
nmap -sU target

# Top UDP ports
nmap -sU --top-ports 20 target
```

---

## IP — Internet Protocol

IP handles addressing and routing. Every packet carries:
- Source IP
- Destination IP
- TTL (Time To Live — decremented at each hop, packet dropped at 0)
- Protocol (6=TCP, 17=UDP, 1=ICMP)

### IPv4 vs IPv6

| | IPv4 | IPv6 |
|--|------|------|
| Address size | 32 bits (4 bytes) | 128 bits (16 bytes) |
| Notation | 192.168.1.1 | 2001:db8::1 |
| Header size | 20 bytes | 40 bytes |
| Broadcast | Yes | No (uses multicast) |
| NAT | Common | Not needed |
| IPSec | Optional | Built-in |

### Private IP Ranges (RFC 1918)

```
10.0.0.0     – 10.255.255.255   (/8)
172.16.0.0   – 172.31.255.255   (/12)
192.168.0.0  – 192.168.255.255  (/16)
127.0.0.0    – 127.255.255.255  loopback
169.254.0.0  – 169.254.255.255  link-local (APIPA)
```

### CIDR Notation

| CIDR | Subnet Mask | Hosts |
|------|-------------|-------|
| /8 | 255.0.0.0 | 16,777,214 |
| /16 | 255.255.0.0 | 65,534 |
| /24 | 255.255.255.0 | 254 |
| /25 | 255.255.255.128 | 126 |
| /26 | 255.255.255.192 | 62 |
| /27 | 255.255.255.224 | 30 |
| /28 | 255.255.255.240 | 14 |
| /29 | 255.255.255.248 | 6 |
| /30 | 255.255.255.252 | 2 |
| /32 | 255.255.255.255 | 1 (single host) |

---

## ICMP — Internet Control Message Protocol

ICMP carries control messages — errors, diagnostics. Not used for data transfer.

**Key ICMP types:**

| Type | Code | Meaning |
|------|------|---------|
| 0 | 0 | Echo Reply (ping response) |
| 3 | 0 | Destination Network Unreachable |
| 3 | 1 | Destination Host Unreachable |
| 3 | 3 | Destination Port Unreachable |
| 5 | 0 | Redirect (routing) |
| 8 | 0 | Echo Request (ping) |
| 11 | 0 | TTL Exceeded (traceroute) |

**Security implications:**
- **Ping sweep:** ICMP echo requests to find live hosts (`nmap -sn`)
- **ICMP tunneling:** Data exfiltration by embedding payload in ICMP packets (tools: icmpsh, ptunnel)
- **Ping of Death:** Oversized ICMP packet causes buffer overflow (historical, patched)
- Firewalls often block ICMP — a host not responding to ping doesn't mean it's down

```bash
# Ping sweep
nmap -sn 192.168.1.0/24

# Traceroute (uses TTL-exceeded ICMP messages)
traceroute target      # Linux
tracert target         # Windows
```

---

## DNS — Domain Name System

DNS translates domain names to IP addresses. Uses UDP port 53 (TCP for zone transfers and large responses).

### Record Types

| Record | Purpose | Example |
|--------|---------|---------|
| A | IPv4 address | `example.com → 93.184.216.34` |
| AAAA | IPv6 address | `example.com → 2606:2800::1` |
| CNAME | Alias to another name | `www → example.com` |
| MX | Mail server | `example.com → mail.example.com` |
| TXT | Arbitrary text | SPF, DKIM, domain verification |
| NS | Name servers | Who is authoritative for the domain |
| PTR | Reverse DNS (IP → name) | `34.216.184.93.in-addr.arpa → example.com` |
| SOA | Start of Authority | Zone metadata |
| SRV | Service location | `_http._tcp.example.com` |

### DNS Resolution Flow

```
Browser → OS cache → /etc/hosts → Recursive resolver (ISP/8.8.8.8)
    → Root nameserver (.) → TLD nameserver (.com) → Authoritative NS
    → Answer returned and cached
```

### DNS Security Implications

- **DNS enumeration:** Find subdomains, mail servers, zone info
- **Zone transfer (AXFR):** If misconfigured, dumps entire DNS zone — full subdomain list
- **DNS cache poisoning:** Inject malicious records into resolver cache
- **DNS amplification:** Small query → large response, used for DDoS
- **DNS tunneling:** Encode data in DNS queries for C2 or exfiltration
- **Subdomain takeover:** DNS points to a cloud resource (S3, GitHub Pages) that no longer exists — attacker claims it

```bash
# Basic lookups
nslookup example.com
dig example.com A
dig example.com MX
dig example.com TXT

# Reverse lookup
dig -x 93.184.216.34

# Zone transfer attempt
dig axfr @ns1.example.com example.com

# DNS enumeration
dnsenum example.com
gobuster dns -d example.com -w /usr/share/wordlists/dns.txt

# Subdomain brute force
ffuf -w subdomains.txt -u http://FUZZ.example.com
```

---

## HTTP — HyperText Transfer Protocol

### Request Structure

```
GET /path?param=value HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html
Cookie: session=abc123
Authorization: Bearer eyJ...
Connection: keep-alive
[blank line]
[body for POST/PUT]
```

### Response Structure

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 1234
Set-Cookie: session=newtoken; HttpOnly; Secure; SameSite=Strict
X-Content-Type-Options: nosniff
[blank line]
[response body]
```

### Status Codes

| Code | Meaning |
|------|---------|
| 200 | OK |
| 201 | Created |
| 204 | No Content |
| 301 | Moved Permanently |
| 302 | Found (temporary redirect) |
| 400 | Bad Request |
| 401 | Unauthorized (need to authenticate) |
| 403 | Forbidden (authenticated but not allowed) |
| 404 | Not Found |
| 405 | Method Not Allowed |
| 429 | Too Many Requests |
| 500 | Internal Server Error |
| 502 | Bad Gateway |
| 503 | Service Unavailable |

### HTTP Methods

| Method | Purpose | Has Body |
|--------|---------|----------|
| GET | Retrieve resource | No |
| POST | Submit data | Yes |
| PUT | Replace resource | Yes |
| PATCH | Partial update | Yes |
| DELETE | Delete resource | No |
| HEAD | GET without body | No |
| OPTIONS | Discover allowed methods | No |

### Security Implications

- **HTTP vs HTTPS:** HTTP is plaintext — credentials, cookies, and data visible to any network observer
- **Verb tampering:** Server may handle unexpected methods (e.g., PUT, DELETE) without auth checks
- **Host header injection:** Server uses `Host:` header to route — inject to get password reset links for another domain
- **CORS misconfiguration:** `Access-Control-Allow-Origin: *` allows any site to read responses from your API

---

## TLS/SSL — Transport Layer Security

TLS encrypts traffic at the transport layer. Provides: **confidentiality, integrity, authentication**.

### TLS Handshake (TLS 1.3)

```
Client                          Server
  |--- ClientHello (cipher list, random) -->|
  |<-- ServerHello (chosen cipher) ---------|
  |<-- Certificate (server's cert) ---------|
  |<-- CertificateVerify --------------------|
  |<-- Finished ----------------------------|
  |--- Finished ----------------------------|
  |======= Encrypted data ================>|
```

### Key Concepts

- **Certificate:** Signed by a CA, proves the server's identity. Contains: domain, public key, expiry, CA signature.
- **Certificate chain:** Leaf cert → Intermediate CA → Root CA. Browser trusts Root CA.
- **SNI (Server Name Indication):** Allows one IP to serve multiple TLS certs (virtual hosting).
- **ALPN:** Protocol negotiation within TLS (HTTP/1.1 vs HTTP/2).

### TLS Versions

| Version | Status |
|---------|--------|
| SSL 2.0 | Broken — disable |
| SSL 3.0 | Broken (POODLE) — disable |
| TLS 1.0 | Deprecated — disable |
| TLS 1.1 | Deprecated — disable |
| TLS 1.2 | Acceptable — widely used |
| TLS 1.3 | Current — use this |

### Security Implications

- **Expired/self-signed certs:** Users get warnings, MITM possible if ignored
- **Weak cipher suites:** RC4, DES, 3DES, NULL ciphers — allow downgrade or decryption
- **BEAST, POODLE, HEARTBLEED:** Historical TLS/OpenSSL vulnerabilities
- **Certificate pinning bypass:** Mobile apps can pin certs — bypass with Frida or custom CA install
- **MITM:** Intercept TLS by presenting your own cert (requires victim to trust your CA)

```bash
# Check TLS version and ciphers
nmap --script ssl-enum-ciphers -p 443 target
sslscan target:443
testssl.sh target

# View certificate
openssl s_client -connect target:443 -showcerts
echo | openssl s_client -connect target:443 2>/dev/null | openssl x509 -noout -text
```

---

## SSH — Secure Shell

SSH provides encrypted remote shell access, tunneling, and file transfer. Default port 22.

### Authentication Methods

| Method | How it works |
|--------|-------------|
| Password | Username + password over encrypted channel |
| Public key | Client proves possession of private key matching server's `authorized_keys` |
| Certificate | SSH CA signs user keys — scalable for large teams |
| GSSAPI/Kerberos | Integrated with enterprise auth |

### Key Commands

```bash
# Connect
ssh user@host
ssh -p 2222 user@host           # non-default port
ssh -i ~/.ssh/id_rsa user@host  # specific key

# Key generation
ssh-keygen -t ed25519 -C "comment"   # modern, preferred
ssh-keygen -t rsa -b 4096            # legacy compatibility

# Copy public key to server
ssh-copy-id user@host

# SSH tunneling
ssh -L 8080:internal-server:80 user@jumphost   # local forward
ssh -R 9090:localhost:80 user@server           # remote forward
ssh -D 1080 user@server                        # SOCKS proxy

# Run command without interactive shell
ssh user@host "cat /etc/passwd"

# SCP file transfer
scp file.txt user@host:/path/
scp user@host:/path/file.txt ./

# SFTP
sftp user@host
```

### Security Implications

- **Default port 22:** Targeted by mass scanners — consider changing port or using port knocking
- **Password auth:** Brute-forceable — disable it, use key auth only
- **Root login:** Disable `PermitRootLogin` in `sshd_config`
- **Known hosts:** SSH warns on first connection — TOFU (Trust On First Use) — if skipped, MITM possible
- **Private key theft:** If `~/.ssh/id_rsa` is readable, attacker gets full access to all servers

```bash
# Brute force (authorized testing only)
hydra -l user -P passwords.txt ssh://target
medusa -h target -u user -P passwords.txt -M ssh

# Check for weak SSH config
nmap --script ssh-auth-methods target
nmap --script ssh2-enum-algos target
```

---

## SMB — Server Message Block

SMB provides file sharing, printer sharing, and named pipes on Windows networks. Port 445 (direct), 139 (NetBIOS).

### SMB Versions

| Version | OS | Notes |
|---------|-----|-------|
| SMBv1 | Windows XP/2003 | Dangerous — EternalBlue, WannaCry. Disable immediately |
| SMBv2 | Windows Vista/2008 | Major improvement |
| SMBv3 | Windows 8/2012+ | Encryption support |

### Security Implications

- **EternalBlue (MS17-010):** Remote code execution via SMBv1 — used by WannaCry/NotPetya
- **Pass-the-Hash:** Reuse captured NTLM hash without cracking it
- **SMB relay:** Capture and relay auth to another SMB server
- **Null session:** Anonymous access to share list and user enumeration (older systems)
- **Kerberoasting:** Request service tickets for SPN accounts, crack offline

```bash
# Enumerate SMB
nmap --script smb-enum-shares,smb-enum-users -p 445 target
enum4linux -a target
smbclient -L //target -N              # null session share list
smbclient //target/share -N          # connect to share

# Check for EternalBlue
nmap --script smb-vuln-ms17-010 -p 445 target

# CrackMapExec
crackmapexec smb target -u user -p password
crackmapexec smb target -u user -H hash  # pass the hash
```

---

## FTP — File Transfer Protocol

FTP transfers files. Port 21 (control), Port 20 (data in active mode).

**Active vs Passive:**
- **Active:** Server initiates data connection back to client — blocked by most firewalls
- **Passive:** Client initiates both connections — firewall-friendly, more common

### Security Implications

- **Plaintext:** Credentials and data sent unencrypted — visible to network sniffers
- **Anonymous login:** Many FTP servers allow login with `anonymous/anonymous` — check for writable dirs
- **FTPS vs SFTP:** FTPS = FTP + TLS (still port 21). SFTP = SSH file transfer (port 22) — different protocol entirely
- **Bounce attack:** Use FTP server to scan other hosts via `PORT` command

```bash
# Connect
ftp target
ftp -n target    # no auto-login

# In FTP shell
anonymous        # try anonymous login
ls               # list files
get file.txt     # download
put file.txt     # upload (if writable)
binary           # switch to binary mode for non-text files

# Nmap FTP scripts
nmap --script ftp-anon,ftp-bounce -p 21 target
```

---

## SMTP — Simple Mail Transfer Protocol

SMTP sends email. Port 25 (server-to-server), 587 (client submission), 465 (SMTPS).

### Email Security Records (DNS TXT)

| Record | Purpose |
|--------|---------|
| SPF | Lists servers allowed to send for the domain |
| DKIM | Cryptographic signature on outgoing mail |
| DMARC | Policy for what to do when SPF/DKIM fail |

**SPF example:**
```
v=spf1 include:_spf.google.com ~all
```
`~all` = softfail (mark as spam), `-all` = hardfail (reject)

### Security Implications

- **Open relay:** SMTP server that forwards mail for anyone — used for spam and phishing
- **User enumeration:** `VRFY user@domain` or `RCPT TO:` responses reveal valid users
- **Email spoofing:** Without DMARC enforcement, attacker sends email pretending to be your domain
- **STARTTLS downgrade:** Force plaintext by stripping STARTTLS negotiation

```bash
# Manual SMTP interaction
nc -nv target 25
HELO attacker.com
MAIL FROM:<attacker@attacker.com>
RCPT TO:<victim@target.com>
DATA
Subject: Test
Test body
.
QUIT

# Enumerate users
smtp-user-enum -M VRFY -U users.txt -t target
nmap --script smtp-enum-users -p 25 target

# Check open relay
nmap --script smtp-open-relay -p 25 target
```

---

## SNMP — Simple Network Management Protocol

SNMP monitors and manages network devices. UDP ports 161 (queries), 162 (traps).

**Versions:**
- **v1, v2c:** Community string auth (plaintext password) — `public` and `private` are defaults
- **v3:** Proper auth + encryption — use this

### Security Implications

- **Default community strings:** `public` (read) and `private` (write) — widely unchanged
- **Information disclosure:** SNMP can dump routing tables, interface info, running processes, installed software, user accounts
- **v1/v2c plaintext:** Community string visible on the wire
- **Write access:** SNMP write with `private` community can change device config

```bash
# Enumerate with default community strings
snmpwalk -c public -v2c target
snmpwalk -c private -v2c target

# Specific OIDs
snmpwalk -c public -v2c target 1.3.6.1.2.1.1     # system info
snmpwalk -c public -v2c target 1.3.6.1.2.1.25.4  # running processes

# Nmap SNMP scripts
nmap --script snmp-info,snmp-sysdescr -p 161 -sU target
```

---

## LDAP — Lightweight Directory Access Protocol

LDAP queries directory services (Active Directory, OpenLDAP). Port 389 (plaintext), 636 (LDAPS).

### Security Implications

- **Anonymous bind:** LDAP allows querying without credentials on many default configs — dump users, groups, org structure
- **LDAP injection:** User input injected into LDAP filter — similar to SQLi
- **Credential exposure:** LDAP often used to validate credentials — cleartext on port 389 unless LDAPS used
- **AD enumeration:** BloodHound uses LDAP to map Active Directory attack paths

```bash
# Anonymous LDAP query
ldapsearch -x -H ldap://target -b "dc=domain,dc=com"

# Authenticated
ldapsearch -x -H ldap://target -D "user@domain.com" -W -b "dc=domain,dc=com"

# Enum with nmap
nmap --script ldap-search -p 389 target
```

---

## ARP — Address Resolution Protocol

ARP maps IP addresses to MAC addresses on a local network. No authentication.

### Security Implications

- **ARP spoofing/poisoning:** Attacker broadcasts fake ARP replies claiming they have the gateway's IP → intercepts all traffic (MITM)
- **ARP cache poisoning:** Victim's ARP cache associates attacker's MAC with gateway IP
- **Gratuitous ARP:** Unsolicited ARP — sent to update caches (legitimate use) but also used for poisoning

```bash
# ARP spoofing (MITM)
arpspoof -i eth0 -t victim_ip gateway_ip
arpspoof -i eth0 -t gateway_ip victim_ip

# With Bettercap
bettercap -iface eth0
# In bettercap console:
net.probe on
arp.spoof.targets 192.168.1.105
arp.spoof on

# View ARP cache
arp -a          # Windows/Linux
ip neigh show   # Linux
```

---

## NFS — Network File System

NFS allows remote filesystem mounting. Port 2049.

### Security Implications

- **No auth by default (NFSv3):** Exports mounted by anyone on the network
- **root_squash bypass:** If `no_root_squash` is set, mounting client's root = remote root
- **Sensitive data exposure:** Home dirs, config files, keys exposed via NFS shares

```bash
# Enumerate NFS shares
showmount -e target
nmap --script nfs-showmount -p 2049 target

# Mount an NFS share
mount -t nfs target:/exported/path /mnt/local

# Check exports config
cat /etc/exports
```

---

## Key Takeaways

- **Protocol = attack surface.** Every open port running a protocol is an entry point — know what each one does.
- **UDP is underscanned.** Most scans focus on TCP — UDP services like SNMP, DNS, NFS are often left wide open.
- **Plaintext protocols are 2026 problems.** FTP, HTTP, Telnet, SMTP (port 25), SNMP v1/v2c, LDAP (port 389) — anything on these in a modern network is a finding.
- **Default credentials are everywhere.** SNMP `public`/`private`, FTP `anonymous`, SMB null session — always check.
- **DNS is both a recon target and an exfiltration channel.** Enumerate it thoroughly, and monitor for DNS tunneling.

---

## References

<div class="references">
<ul>
  <li><a href="https://nmap.org/book/man.html" target="_blank">Nmap Reference Guide</a></li>
  <li><a href="https://book.hacktricks.xyz/network-services-pentesting" target="_blank">HackTricks — Network Services Pentesting</a></li>
  <li><a href="https://www.rfc-editor.org/" target="_blank">RFC Editor — Protocol Specifications</a></li>
  <li><a href="https://www.wireshark.org/docs/" target="_blank">Wireshark Documentation</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
