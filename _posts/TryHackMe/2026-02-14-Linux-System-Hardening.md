---
layout: post
title: Linux System Hardening
date: 2026-02-14T20:51:00
categories:
  - TryHackMe
tags:
  - linux
  - logs
  - firewall
  - thm
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



We can consider adding a GRUB password depending on the Linux system we want to protect. Many tools help achieve that. One tool is **grub2-mkpasswd-pbkdf2,** which prompts you to input your password twice and generates a hash for you. The resulting hash should be added to the appropriate configuration file depending on the Linux distribution (examples: Fedora and Ubuntu). This configuration would prevent unauthorized users from resetting your root password. It will require the user to supply a password to access advanced boot configurations via GRUB, including logging in with root access.



`root@AttackBox# grub2-mkpasswd-pbkdf2`
`Enter password:` 
`Reenter password:` 
`PBKDF2 hash of your password is grub.pbkdf2.sha512.10000.534B77859C13DCF094E90B926E26C586F5DC9D00687853487C4BB1500D57EC29E2D6D07A586262E093DCBDFF4B3552742A25700BAB6B76A8206B3BFCB273EEB4.4BA1447590EA8451CD224AA1C5F8623FE85D23F6D34E2026E3F08C5AA79282DB65B330BAB4944E9374EC51BF11EFF418EDA5D66FF4D7AAA86F662F793B92DA61`

---


## Filesystem Encryption

![firewall](/assets/thm/Pasted%20image%2020260217074732.png)


using **LUKS( Linux unified key setup) we can provide encryption to Linux systems

![disk layout](/assets/Pasted%20image%2020260218230113.png)
We have the following fields:  

- **LUKS phdr:** It stands for _LUKS Partition Header_. LUKS phdr stores information about the UUID (Universally Unique Identifier), the used cipher, the cipher mode, the key length, and the checksum of the master key.
- **KM**: KM stands for _Key Material_, where we have KM1, KM2, …, KM8. Each key material section is associated with a key slot, which can be indicated as active in the LUKS phdr. When the key slot is active, the associated key material section contains a copy of the master key encrypted with a user's password. In other words, we might have the master key encrypted with the first user's password and saved in KM1, encrypted with the second user's password and saved in KM2, and so on.
- **Bulk Data**: This refers to the data encrypted by the master key. The master key is saved and encrypted by the user's password in a key material section.

**LUKS** reuses existing block encryption implementations. The pseudocode to encrypt data uses the following syntax:

`enc_data = encrypt(cipher_name, cipher_mode, key, original, original_length)`

and to decrypt using the similar pseudocode but with the enc_data as an input 

`original = decrypt(cipher_name, cipher_mode, key, enc_data, original_length)`


to install **LUKS** in the command line use the following steps : 

- Install `cryptsetup-luks`. (You can issue `apt install cryptsetup`, `yum install cryptsetup-luks` or `dnf install cryptsetup-luks` for Ubuntu/Debian, RHEL/Cent OS, and Fedora, respectively.)

- Confirm the partition name using `fdisk -l`, `lsblk` or `blkid`. (Create a partition using `fdisk` if necessary.)
- Set up the partition for LUKS encryption: `cryptsetup -y -v luksFormat /dev/sdb1`. (Replace `/dev/sdb1` with the partition name you want to encrypt.)
- Create a mapping to access the partition: `cryptsetup luksOpen /dev/sdb1 EDCdrive`.
- Confirm mapping details: `ls -l /dev/mapper/EDCdrive` and `cryptsetup -v status EDCdrive`.
- Overwrite existing data with zero: `dd if=/dev/zero of=/dev/mapper/EDCdrive`.
- Format the partition: `mkfs.ext4 /dev/mapper/EDCdrive -L "Strategos USB"`.
- Mount it and start using it like a usual partition: `mount /dev/mapper/EDCdrive /media/secure-USB`.


If you want to check the LUKS setting, you can issue the command `cryptsetup luksDump /dev/sdb1`

---

## Firewalls 

Let’s briefly revisit the client/server model before we start. Any networked device can be a client, a server, or both simultaneously. The server offers a service, and the client connects to the server to use it. Examples of servers include web servers (HTTP and HTTPS), mail servers (SMTP(S), POP3(S), and IMAP(S)), name servers (DNS), and SSH servers. A server listens on a known TCP or UDP port number awaiting incoming client connection requests. The client initiates the connection request to the listening server, and the server responds to it.

A firewall decides which packets can enter a system and which packets can leave a system


Setting up a firewall offers many security benefits. First and foremost, firewall rules provide fine control over which packets can leave your system and which packets can enter your system. Consequently, firewall rules help mitigate various security risks by controlling network traffic between devices. More importantly, firewall rules can be devised to ensure that no client can act as a server. In other words, an attacker cannot start a reachable listening port on a target machine; the exploit can start a listening port, but the firewall will prevent all incoming connection attempts.

There is 2 types of firewalls: 
1) A host based firewall: 
2) A client based firewall 


The firewall has two main functions:

- What can enter? Allow or deny packets from entering a system.
- What can leave? Allow or deny packets from leaving a system.


## Linux Firewalls


On a Linux system, a solution such as [SELinux](https://github.com/SELinuxProject) or [AppArmor](https://www.apparmor.net/) can be used for more granular control over processes and their network access. For example, we can allow only the `/usr/bin/apache2` binary to use ports 80 and 443 while preventing any other binary from doing so on the underlying system. Both tools enforce access control policies based on the specific process or binary, providing a more comprehensive way to secure a Linux system.


Let’s look take a closer look at the different available Linux firewalls.

## Netfilter

At the very core, we have netfilter. The netfilter project provides packet-filtering software for the Linux kernel 2.4.x and later versions. The netfilter hooks require a front-end such as `iptables` or `nftables` to manage.


## iptables

As a front-end, iptables provides the user-space command line tools to configure the packet filtering rule set using the netfilter hooks. For filtering the traffic, iptables has the following default chains:

- **Input**: This chain applies to the packets incoming to the firewall.
- **Output**: This chain applies to the packets outgoing from the firewall.
- **Forward** This chain applies to the packets routed through the system.

Let’s say that we want to be able to access the SSH server on our system remotely. For the SSH server to be able to communicate with the world, we need two things:

1. Accept incoming packets to TCP port 22.
2. Accept outgoing packets from TCP port 22.

Let’s translate the above two requirements into `iptables` commands:

`iptables -A INPUT -p tcp --dport 22 -j ACCEPT`

- `-A INPUT` appends to the INPUT chain, i.e., packets destined for the system.
- `-p tcp --dport 22` applies to TCP protocol with destination port 22.
- `-j ACCEPT` specifies (jump to) target rule ACCEPT.

`iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT`

- `-A OUTPUT` append to the OUTPUT chain, i.e., packets leaving the system.
- `-p tcp --sport 22` applies to TCP protocol with source port 22.

Let’s say you only want to allow traffic to the local SSH server and block everything else. In this case, you need to add two more rules to set the default behaviour of your firewall:

- `iptables -A INPUT -j DROP` to block all incoming traffic not allowed in previous rules.
- `iptables -A OUTPUT -j DROP` to block all outgoing traffic not allowed in previous rules.

In brief, the rules below need to be applied in the following order:

```shell-session
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT
iptables -A INPUT -j DROP
iptables -A OUTPUT -j DROP
```

## UFW

UFW stands for uncomplicated firewall. this following simple command can set a firewall rule to allow SSH traffic 

`ufw allow 22/tcp`

then confirm the status using `ufw status`

In the end before you configure a firewall you need to set the **Firewall policy** to be used  It all depends on the system you are protecting

 the two main approaches are:

- Block everything and allow certain exceptions.
- Allow everything and block certain exceptions.

--- 
## Remote Access 

The configuration of the OpenSSH server can be controlled via the `sshd_config` file, usually located at `/etc/ssh/sshd_config`. You can disable the root login by adding the following line:

`PermitRootLogin no`

Although a password such as `9bNfX2gmDZ4o` is difficult to guess, most users find memorising it inconvenient. Imagine if the account belongs to the _sudoers_ (`sudo` group), and the user needs to type this password every time they need to issue a command with `sudo`. You may have to discipline to do that, but you cannot expect this to work for everyone.

Many users are tempted to select a user-friendly password or share the same password across multiple accounts. Either approach would make the password easier for the attacker to guess.

It would be best to rely on public key authentication with SSH to help improve the security of the remote login system and make it as fail-proof as possible.

If you haven’t created an SSH key pair, you must issue the command `ssh-keygen -t rsa`. It will generate a private key saved in `id_rsa` and a public key saved in `id_rsa.pub`.

For the SSH server to authenticate you using your public key instead of your passwords, your public key needs to be copied to the target SSH server. An easy way to do it would be by issuing the command `ssh-copy-id username@server` where `username` is your username, and `server` is the hostname or IP address of the SSH server.

It is best to ensure you have access to the physical terminal before you disable password authentication to avoid locking yourself out. You might need to ensure having the following two lines in your `sshd_config` file.

- `PubkeyAuthentication yes` to enable public key authentication
    
- `PasswordAuthentication no` to disable password authentication

---
## Secure User Account 











---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer )
- **GitHub:** [Mhdomer](https://github.comMhdomer)  
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/) 
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
