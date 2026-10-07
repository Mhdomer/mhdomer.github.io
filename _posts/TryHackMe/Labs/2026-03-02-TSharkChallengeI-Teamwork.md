---
layout: post
title: "TryHackMe: TShark Challenge I - Directory Curiosity Walkthrough"
date: 2026-03-02T12:12:00+03:00
categories:
  - TryHackMe
  - Labs
tags:
  - thm
  - network
  - labs
  - tshark
  - wireshark
  - packet-analysis
  - malware-analysis
author: muhammed
description: Walkthrough of the TryHackMe TShark Challenge I (Directory Curiosity). Analyzing malicious network traffic from the command line, filtering DNS queries, tracking HTTP streams, and carving out malware payloads.
toc: true
pin: false
math: false
mermaid: true
permalink: /posts/TryHackMe-TShark-Challenge-I/
---

## Overview

When learning network traffic analysis, most people (including myself) naturally gravitate toward the Wireshark graphical interface. It has friendly buttons, color-coded packets, and a visual layout.

However, in real incident response and SOC workflows, you often find yourself working on headless Linux instances or SSH sessions where GUI Wireshark is not available. That is where **TShark** comes in.

[TShark Challenge I](https://tryhackme.com/room/tsharkchallenges) on TryHackMe is designed to push you out of your comfort zone by analyzing a malicious packet capture (`directory-curiosity.pcap`) purely from the command line.

The scenario:
> An alert has been triggered: *"A user came across a poor file index, and their curiosity led to problems."*

Our mission is to inspect the capture file, follow the attacker's traffic, identify the malicious domain and infrastructure, and carve out the downloaded malware payload.

Here is my complete step-by-step investigation.

---

## 1. Finding the Malicious Domain via DNS Queries

Every web interaction usually begins with a DNS lookup. To see what domains the user looked up before running into trouble, I used `tshark` with a display filter targeting DNS traffic:

```bash
tshark -r directory-curiosity.pcap -Y "dns" | grep 'com'
```

Looking through the output, one unusual domain stood out immediately among standard internal requests:

```text
jx2-bavuong.com
```

To confirm whether this domain was known to be malicious, I checked it against VirusTotal. Multiple security vendors flagged it as a known malicious host used in malware delivery campaigns.

- **Malicious domain:** `jx2-bavuong.com`

---

## 2. Tracking HTTP Activity and Request Counts

Now that we know the suspicious domain, how many times did the victim machine interact with it over HTTP?

I combined the `http.request` display filter with `http.host` and piped the results into `wc -l` to count the total requests:

```bash
tshark -r directory-curiosity.pcap -Y "http.request and http.host contains jx2-bavuong.com" | wc -l
```

The count came back:

```text
14
```

The victim browser sent **14 HTTP requests** to the malicious server.

---

## 3. Extracting the Destination IP Address

Next, I needed to identify the actual IP address hosting the malicious domain.

Instead of parsing raw text, `tshark` allows you to extract specific protocol fields cleanly using `-T fields -e`:

```bash
tshark -r directory-curiosity.pcap \
  -Y "http.request and http.host contains jx2-bavuong.com" \
  -T fields -e ip.dst | sort -u
```

This isolated the destination IP:

```text
141.164.41.174
```

- **Malicious IP Address:** `141.164.41.174`

---

## 4. Identifying the Attacker Web Server

Web servers identify themselves through the `Server` HTTP response header. I queried the pcap file for all `http.server` response headers:

```bash
tshark -r directory-curiosity.pcap -T fields -e "http.server" | awk NF | sort -u
```

The server banner returned:

```text
Apache/2.2.11 (Win32) DAV/2 mod_ssl/2.2.11 OpenSSL/0.9.8i PHP/5.2.9
```

This banner tells us the attacker was running an older Apache 2.2 instance on Windows with mod_ssl and PHP 5.2.9 enabled.

---

## 5. Following TCP Streams: Inspecting the Directory Listing

The alert mentioned that the user stumbled upon a *"poor file index"*. This usually means directory indexing was enabled on the web server (an open directory listing).

To view the raw conversation between the client and server, I followed TCP Stream 0 in ASCII format:

```bash
tshark -r directory-curiosity.pcap -Y "tcp.stream eq 0" -z follow,tcp,ascii,0
```

Looking at the HTTP response body inside the stream:

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<html>
 <head>
  <title>Index of /</title>
 </head>
 <body>
<h1>Index of /</h1>
<ul>
  <li><a href="123.php"> 123.php</a></li>
  <li><a href="test.txt"> test.txt</a></li>
  <li><a href="vlauto.exe"> vlauto.exe</a></li>
</ul>
</body></html>
```

The server displayed an open directory index containing **3 files**:
1. `123.php`
2. `test.txt`
3. `vlauto.exe`

- **Number of listed files:** 3
- **First listed file:** `123.php`

---

## 6. Carving Out the Malware Payload: Exporting HTTP Objects

Looking at the index listing, `vlauto.exe` is clearly the suspicious binary that the victim downloaded.

One of TShark's coolest features is its ability to automatically extract files transferred over HTTP without needing external carving scripts. You can use the `--export-object` flag:

```bash
mkdir -p extracted_files
tshark -r directory-curiosity.pcap --export-object http,extracted_files
```

Listing the contents of our export directory:

```bash
ls -la extracted_files
```

Inside, we find `vlauto.exe` cleanly carved directly from the packet capture!

---

## 7. Hashing and Analyzing the Malware

With the executable saved to disk, I calculated its SHA256 checksum:

```bash
sha256sum extracted_files/vlauto.exe
```

Output:

```text
b4851333efaf399889456f78eac0fd532e9d8791b23a86a19402c1164aed20de
```

I searched this hash on VirusTotal to gather threat intelligence:
- **PEiD Packer:** Checking the file details section showed the binary was compiled as a **.NET executable**.
- **Lastline Sandbox Detection:** The automated sandbox analysis classified the file behaviour as: **MALWARE TROJAN**.

The investigation confirmed that the alert was indeed a true positive: a user visited an open directory on `jx2-bavuong.com` and downloaded a malicious .NET trojan executable.

---

## Lessons Learned

Taking on this challenge gave me a much greater appreciation for command-line packet analysis:

1. **`tshark` filters are identical to Wireshark:** Any display filter you use in Wireshark (`dns`, `http.host`, `tcp.stream eq 0`) works exactly the same in `tshark -Y`.
2. **Field extraction is a superpower:** Using `-T fields -e ip.dst` combined with standard Linux utilities like `sort -u` or `awk` makes extracting IPs, URLs, and User-Agents 10x faster than manually clicking through GUI packet trees.
3. **Automate payload recovery with `--export-object`:** Instead of manually rebuilding TCP streams to extract downloads, `tshark --export-object http,<folder>` extracts all transferred web files instantly.

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
