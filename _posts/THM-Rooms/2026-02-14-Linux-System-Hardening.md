---
layout: post
title: Linux System Hardening
date: 2026-02-14T20:51:00
categories:
  - THM
tags:
  - linux
  - logs
  - firewall
author: muhammed
description: Learn how to improve the security posture of your Linux systems.
toc: true
pin: false
math: false
mermaid: false
image: https://tryhackme-images.s3.amazonaws.com/room-icons/68ab00dd034f022f110e1b2ed098bf7d.png
---

## Objectives

- Physical Security
- Filesystem Encryption
- Firewall Configuration
- Remote Access
- Software and Services
- Updates and Upgrades
- Logs

--- 
## Physical Security



We can consider adding a GRUB password depending on the Linux system we want to protect. Many tools help achieve that. One tool is **grub2-mkpasswd-pbkdf2,** which prompts you to input your password twice and generates a hash for you. The resulting hash should be added to the appropriate configuration file depending on the Linux distribution (examples: Fedora and Ubuntu). This configuration would prevent unauthorised users from resetting your root password. It will require the user to supply a password to access advanced boot configurations via GRUB, including logging in with root access.



`root@AttackBox# grub2-mkpasswd-pbkdf2`
`Enter password:` 
`Reenter password:` 
`PBKDF2 hash of your password is grub.pbkdf2.sha512.10000.534B77859C13DCF094E90B926E26C586F5DC9D00687853487C4BB1500D57EC29E2D6D07A586262E093DCBDFF4B3552742A25700BAB6B76A8206B3BFCB273EEB4.4BA1447590EA8451CD224AA1C5F8623FE85D23F6D34E2026E3F08C5AA79282DB65B330BAB4944E9374EC51BF11EFF418EDA5D66FF4D7AAA86F662F793B92DA61`

---


## Filesystem Encryption













---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer )
- **GitHub:** [Mhdomer](https://github.comMhdomer)  
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/) 
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
