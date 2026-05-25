---
layout: post
title: "Homelab Part 2 — VMware Setup and VM Planning"
date: 2026-05-24 11:00:00 +0800
categories:
  - Homelab
tags:
  - Homelab
  - VMware
  - VirtualMachine
  - Networking
  - Ubuntu
author: muhammed
description: Setting up VMware Workstation 17 Pro, planning the VM layout, configuring networking, and creating the two VMs that power the entire homelab.
toc: true
pin: false
math: false
mermaid: false
---

## Why VMware and Not Proxmox

The reference homelab uses Proxmox — a bare-metal Type 1 hypervisor you install by wiping your machine. I'm not doing that. I use my laptop daily for everything, and I'm not ready to commit to a full OS replacement.

VMware Workstation 17 Pro is a Type 2 hypervisor — it runs on top of Windows like any other application. I can start and stop my VMs whenever I want, pause them when I need full performance for other work, and keep my normal Windows setup untouched.

The trade-off is a small performance overhead. For a homelab running media and file storage, that overhead is irrelevant.

---

## VMware Networking — Which Mode to Use

VMware gives you three network modes for VMs. Picking the right one matters.

| Mode | What it does | Use case |
|------|-------------|----------|
| NAT | VM shares host's IP, hidden from network | Default — good for internet access but other devices can't reach the VM |
| Bridged | VM gets its own IP on your real network | **What I'm using** — devices on your network can reach the VM directly |
| Host-only | VM can only talk to the host | Isolated testing, no internet |

I'm using **Bridged mode** for both VMs. This means each VM gets its own IP address from my router — they appear as separate devices on my home network, just like a physical computer would. My phone, iPad, and other devices can reach them directly.

> `[SCREENSHOT]` — *VMware network adapter settings showing Bridged mode selected*

---

## VM Planning

I'm creating two VMs. Everything in the homelab runs inside one of these two.

### VM 1 — AdGuard Home (DNS)

| Setting | Value |
|---------|-------|
| Name | `homelab-dns` |
| OS | Ubuntu Server 22.04 LTS |
| RAM | 1GB |
| CPU | 1 core |
| Disk | 20GB |
| Network | Bridged |
| IP | Static — `192.168.1.50` |

Small and lightweight. It runs 24/7. I'll give it a static IP by reserving it in my router's DHCP settings.

### VM 2 — K3s Node

| Setting | Value |
|---------|-------|
| Name | `homelab-k3s` |
| OS | Ubuntu Server 22.04 LTS |
| RAM | 24GB |
| CPU | 8 cores |
| Disk | 200GB (thin provisioned) |
| Network | Bridged |
| IP | Static — `192.168.1.51` |

This is the main VM. Everything — Jellyfin, Nextcloud, all Kubernetes pods — runs here. I'm giving it 24GB RAM (leaving ~15GB for my Windows host and other work) and 8 cores. Thin-provisioned disk means it only uses actual space on my SSD, not the full 200GB upfront.

---

## Creating the VMs

### Step 1 — Download Ubuntu Server 22.04 LTS

I download the ISO from ubuntu.com (server edition, not desktop — no GUI, lighter).

> `[SCREENSHOT]` — *Ubuntu downloads page with Ubuntu Server 22.04.x LTS selected*

### Step 2 — Create VM in VMware

**File → New Virtual Machine → Typical**

- Installer disc image file (ISO): select the Ubuntu ISO
- Guest OS: Linux → Ubuntu 64-bit
- Name: `homelab-dns` (or `homelab-k3s`)
- Location: somewhere with enough disk space

> `[SCREENSHOT]` — *VMware new VM wizard — guest OS selection screen*

### Step 3 — Customize Hardware Before Finishing

Before clicking Finish, click **Customize Hardware**:

For `homelab-dns`:
- RAM: 1024MB
- Processors: 1
- Network Adapter: Bridged (Replicate physical network connection state)

For `homelab-k3s`:
- RAM: 24576MB (24GB)
- Processors: 2 sockets × 4 cores = 8 logical cores
- Network Adapter: Bridged
- Hard Disk: 200GB, thin provisioned

> `[SCREENSHOT]` — *VMware hardware customisation screen showing RAM and processor settings for the K3s VM*

### Step 4 — Ubuntu Server Installation

Power on the VM and walk through the Ubuntu installer:

- Language: English
- Keyboard: your layout
- Network: leave as DHCP for now (I'll set static IP after)
- Storage: use entire disk (the virtual disk only)
- Profile: set your username and password — I use `homelab` as the username
- **SSH:** check **Install OpenSSH server** — essential so I can SSH in from Windows

> `[SCREENSHOT]` — *Ubuntu server installer — OpenSSH server option checked*

Installation takes about 5 minutes. Reboot when done.

---

## Setting Static IPs

I don't want the VM IPs changing. Two ways to do this:

**Option A — Router DHCP reservation (easier)**
Go to my router admin page, find the VM's MAC address in the DHCP leases table, and assign it a fixed IP. The VM still uses DHCP but always gets the same address.

**Option B — Static IP on the VM itself using Netplan (Ubuntu)**

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  ethernets:
    ens33:                        # check your interface name with: ip link show
      dhcp4: false
      addresses:
        - 192.168.1.50/24         # change for each VM
      routes:
        - to: default
          via: 192.168.1.1        # your router IP
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
  version: 2
```

```bash
sudo netplan apply
ip addr show ens33               # verify new IP
```

> `[SCREENSHOT]` — *Terminal showing the VM's new static IP after netplan apply*

I use Option B — it's self-contained in the VM and doesn't depend on my router's admin panel.

---

## SSH Access from Windows

Once the VMs have static IPs, I SSH into them from Windows Terminal instead of using VMware's console window. Much more comfortable.

```powershell
# From Windows PowerShell or Terminal
ssh homelab@192.168.1.50         # AdGuard VM
ssh homelab@192.168.1.51         # K3s VM
```

First time connecting, accept the host key fingerprint. I also copy my SSH public key so I don't need a password:

```powershell
# Generate key if you don't have one
ssh-keygen -t ed25519

# Copy to VM
ssh-copy-id homelab@192.168.1.51
```

> `[SCREENSHOT]` — *Windows Terminal with two SSH sessions — one to each VM*

---

## Post-Install Setup (Both VMs)

I run these on every fresh Ubuntu VM before doing anything else:

```bash
# Update everything
sudo apt update && sudo apt upgrade -y

# Install useful tools
sudo apt install -y curl wget git htop net-tools unzip

# Disable swap (required for Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab   # comment out swap in fstab permanently

# Set hostname
sudo hostnamectl set-hostname homelab-k3s   # or homelab-dns
```

### Verify swap is off

```bash
free -h
# Swap line should show 0B total
```

> `[SCREENSHOT]` — *`free -h` output showing Swap: 0B 0B 0B*

---

## VMware Tools

VMware Tools improves performance and enables features like copy-paste between host and VM. On Ubuntu:

```bash
sudo apt install -y open-vm-tools
sudo systemctl enable open-vm-tools
```

---

## Resource Management — Running Alongside Daily Work

Since I'm using this laptop daily, I configure VMware to not let the VMs starve my host:

**VMware → VM Settings → Processors → Advanced**
- Check: "Limit processor usage" if needed

**The approach I use:** I suspend the VMs when I'm doing heavy work (video, compiling, etc.) and resume them when I just need them for browsing or writing. VMware suspend/resume is instant — the VMs come back in 2–3 seconds exactly where they left off.

```
VMware toolbar → Suspend (pause icon)  → resumes from exact state later
```

> `[SCREENSHOT]` — *VMware showing both VMs — one running, showing power/suspend controls*

---

## Where I Am Now

At the end of Part 2 I have:

- ✅ Two Ubuntu 22.04 VMs created in VMware 17 Pro
- ✅ Both on Bridged networking with static IPs
- ✅ SSH access from Windows working
- ✅ Swap disabled on both
- ✅ VMware Tools installed

Next: set up AdGuard Home on the DNS VM so I have internal DNS for my homelab hostnames.

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
