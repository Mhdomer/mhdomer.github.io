---
layout: post
title: "Homelab Part 3 — Internal DNS with AdGuard Home"
date: 2026-05-24 12:00:00 +0800
categories:
  - Homelab
tags:
  - Homelab
  - DNS
  - AdGuard
  - Networking
author: muhammed
description: Installing AdGuard Home as the internal DNS server for the homelab — so every service gets a clean hostname instead of an IP address, plus network-wide ad blocking as a bonus.
toc: true
pin: false
math: false
mermaid: false
---

## Why I Need Internal DNS

Right now, to access a service running on my K3s VM, I'd type something like `http://192.168.1.51:30080`. That's ugly, hard to remember, and breaks the moment I change the IP.

What I want is: `http://jellyfin.home.lab` — from any device on my network.

To make that work, I need a DNS server that knows about my internal hostnames. My router's built-in DNS only knows about the internet — it has no idea what `jellyfin.home.lab` means. I need my own DNS server that I control.

AdGuard Home does this and blocks ads across the entire network as a side effect.

---

## Installing AdGuard Home

I SSH into the DNS VM (`192.168.1.50`):

```bash
ssh homelab@192.168.1.50
```

Download and run the AdGuard Home installer:

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

The installer:
- Downloads the AdGuard Home binary
- Installs it as a systemd service
- Starts it automatically

> `[SCREENSHOT]` — *Terminal showing AdGuard Home installation completing with "AdGuard Home is now installed and running"*

---

## Initial Setup via Web UI

AdGuard Home has a web interface. After installation, I open a browser on my laptop and go to:

```
http://192.168.1.50:3000
```

The setup wizard runs:

1. **Admin interface port:** keep `3000` (or change to `80` if you want)
2. **DNS server port:** keep `53`
3. **Username and password:** set something secure
4. **DNS upstream servers:** I use Cloudflare (`1.1.1.1`) and Google (`8.8.8.8`)

Click **Next → Next → Open Dashboard**.

> `[SCREENSHOT]` — *AdGuard Home web dashboard showing query stats and DNS activity*

---

## Configuring Upstream DNS

In **Settings → DNS settings → Upstream DNS servers**, I configure which servers AdGuard uses for queries it can't answer itself:

```
https://dns.cloudflare.com/dns-query
https://dns.google/dns-query
```

Using DNS-over-HTTPS (DoH) means my DNS queries are encrypted — my ISP can't see what I'm looking up.

**Bootstrap DNS servers** (used to resolve the DoH hostnames themselves):

```
1.1.1.1
8.8.8.8
```

> `[SCREENSHOT]` — *AdGuard DNS settings page showing upstream servers configured with DoH URLs*

---

## Adding Internal DNS Records

This is the key step — telling AdGuard Home about my homelab hostnames.

Go to **Filters → DNS rewrites → Add DNS rewrite**:

| Domain | Answer |
|--------|--------|
| `jellyfin.home.lab` | `192.168.1.200` |
| `nextcloud.home.lab` | `192.168.1.200` |
| `k3s.home.lab` | `192.168.1.51` |
| `adguard.home.lab` | `192.168.1.50` |

`192.168.1.200` is the IP MetalLB will assign to my Nginx Ingress — all services share one IP, with the hostname determining which service they reach. I'll configure MetalLB in Part 5.

> `[SCREENSHOT]` — *AdGuard DNS rewrites page showing all four entries added*

---

## Pointing My Devices to AdGuard

Now I need my devices to use AdGuard Home (`192.168.1.50`) as their DNS server instead of the router's built-in DNS.

### Option A — Router Level (Recommended)

Best option — every device on the network automatically uses AdGuard without any per-device config.

In my router admin panel:
- Find the DHCP settings
- Change the **DNS server** (or "Primary DNS") from `192.168.1.1` (router) to `192.168.1.50` (AdGuard VM)

After saving, any device that reconnects to WiFi or renews its DHCP lease will use AdGuard.

> `[SCREENSHOT]` — *Router admin panel showing DHCP DNS field changed to 192.168.1.50*

### Option B — Per Device (Fallback)

If I can't change the router (ISP-locked), I set DNS manually per device:

**Windows:**
- Network adapter settings → IPv4 → DNS: `192.168.1.50`

**iPhone/iPad:**
- Settings → WiFi → (i) next to network → Configure DNS → Manual → `192.168.1.50`

**Android:**
- Settings → WiFi → long press network → Modify → Advanced → DNS: `192.168.1.50`

---

## Testing DNS Resolution

From my Windows laptop, I test that AdGuard is working:

```powershell
# Flush DNS cache first
ipconfig /flushdns

# Test internal hostname resolution
nslookup jellyfin.home.lab 192.168.1.50
# Should return: 192.168.1.200

# Test external resolution still works
nslookup google.com 192.168.1.50
# Should return Google's IPs

# Test ad blocking
nslookup doubleclick.net 192.168.1.50
# Should return 0.0.0.0 (blocked)
```

> `[SCREENSHOT]` — *PowerShell showing nslookup results — internal hostname resolving to 192.168.1.200, external resolving correctly*

---

## AdGuard Home — Useful Settings to Enable

### Blocklists

**Filters → DNS blocklists → Add blocklist**

I add these:
- AdGuard DNS filter (built-in)
- EasyList
- Steven Black's Unified Hosts

> `[SCREENSHOT]` — *AdGuard blocklists page showing enabled filter lists and total blocked domains count*

### Safe Browsing and Parental Controls

**Settings → General settings:**
- Enable Safe Browsing — blocks malicious domains
- Leave parental controls off (my network, my rules)

### Query Log

**Query Log** in the top menu shows every DNS query from every device in real time. Useful for debugging and also eye-opening — you see exactly how many tracking requests every device makes.

> `[SCREENSHOT]` — *AdGuard query log showing device IPs, queried domains, and blocked/allowed status*

---

## Making AdGuard Survive Reboots

AdGuard Home is installed as a systemd service, so it starts automatically:

```bash
sudo systemctl status AdGuardHome
sudo systemctl enable AdGuardHome    # should already be enabled
```

The VM itself auto-starts in VMware if I set it:
**VMware → VM → Settings → Options → Advanced → Power on this virtual machine after the host powers on**

> `[SCREENSHOT]` — *systemctl status AdGuardHome showing active (running)*

---

## Where I Am Now

At the end of Part 3 I have:

- ✅ AdGuard Home running on `192.168.1.50`
- ✅ All my home devices using it as DNS
- ✅ Internal hostnames (`jellyfin.home.lab`, `nextcloud.home.lab`) resolving to `192.168.1.200`
- ✅ Ad blocking active network-wide
- ✅ Encrypted DNS queries (DoH to Cloudflare and Google)

The DNS is ready. Next: set up the K3s Kubernetes cluster on the main VM.

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
