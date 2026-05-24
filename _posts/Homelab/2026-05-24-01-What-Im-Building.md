---
layout: post
title: "Homelab Part 1 — What I'm Building and Why"
date: 2026-05-24 10:00:00 +0800
categories:
  - Homelab
tags:
  - Homelab
  - VMware
  - Kubernetes
  - K3s
  - Infrastructure
author: muhammed
description: The plan, the goal, and the full stack for my self-hosted homelab — running on a single laptop with VMware 17 Pro, K3s, Jellyfin, Nextcloud, and Cloudflare Tunnel.
toc: true
pin: false
math: false
mermaid: false
---

## Why I'm Doing This

I came across a diagram of a homelab someone had built on a single laptop — running Kubernetes, a service mesh, GitOps, full observability, and self-hosted media. My first reaction was: that's insane. My second reaction was: I need to try this.

The reason isn't just to have cool infrastructure. It's to actually understand how modern cloud infrastructure is built — not by reading about it, but by running it. Every service I deploy, every networking decision I make, every failure I debug teaches me something that no tutorial video can.

And on top of that — I want something practical at the end. The goal of this homelab is:

> **Host my own media library and personal file storage, accessible from any device on my home network, with the option to open it to the outside world securely.**

Jellyfin for movies. Nextcloud for files. Running on Kubernetes. Accessible over HTTPS with proper DNS. And a Cloudflare Tunnel if I ever want external access without exposing my home IP.

---

## My Hardware

I'm running this on my main laptop — not a dedicated server.

| Component | Spec |
|-----------|------|
| RAM | 40GB |
| Hypervisor | VMware Workstation 17 Pro |
| OS | Windows (host) |
| Storage | Internal SSD (will use a partition/large VMDK) |

The important thing: I'm using my main laptop day-to-day. The VMs will be configured with resource limits so they don't eat everything. I can pause them when I need the full machine.

I'm using **VMware Workstation 17 Pro** as my hypervisor. It runs on top of Windows (Type 2), so I don't need to wipe my machine. Everything lives inside VMware.

---

## The Stack

```
My Laptop (Windows + VMware 17 Pro)
    │
    ├── VM 1: AdGuard Home (DNS)
    │         ↳ Internal DNS for homelab hostnames
    │
    └── VM 2: K3s Node (Ubuntu 22.04)
              │
              ├── MetalLB          → assigns real local IPs to services
              ├── Nginx Ingress    → routes traffic to the right service
              ├── Cert-Manager     → manages TLS certificates
              ├── Longhorn         → persistent storage for pods
              ├── Jellyfin         → media server (movies, shows)
              ├── Nextcloud        → personal file storage
              └── Cloudflare Tunnel → secure external access (future)
```

Two VMs. Everything else runs inside Kubernetes on the K3s VM.

---

## What Each Piece Does

### VMware Workstation 17 Pro
My hypervisor. It runs virtual machines on top of Windows without needing to reinstall anything. Each VM is isolated — its own CPU, RAM, disk, and network. If a VM breaks, I delete it and start over.

### AdGuard Home
A DNS server running in a small VM. It does two things:
1. Blocks ads and trackers for every device on my network
2. Resolves my internal hostnames — so `jellyfin.home.lab` resolves to the right K3s IP

Without this, I'd be accessing everything by IP address. With it, everything has a clean domain.

### K3s (Lightweight Kubernetes)
K3s is real Kubernetes — it runs everything the same way AWS EKS does, just without the cloud overhead. I'm running a single-node cluster, meaning one VM acts as both the control plane and the worker. Everything I learn here applies directly to cloud Kubernetes.

### MetalLB
In cloud Kubernetes, when you create a LoadBalancer service, the cloud gives it a real IP automatically. On bare metal (my VM), there's no cloud. MetalLB fills that gap — I give it an IP range from my local network and it assigns real IPs to Kubernetes services.

### Nginx Ingress
The front door for all HTTP/HTTPS traffic into the cluster. Instead of exposing a different port for every service, Nginx Ingress reads the hostname and routes to the right pod. `jellyfin.home.lab` goes to Jellyfin, `nextcloud.home.lab` goes to Nextcloud.

### Cert-Manager
Automatically issues and renews TLS certificates so all my services run on HTTPS. For internal services it uses a self-signed CA. For external services (if I add Cloudflare Tunnel later) it uses Let's Encrypt.

### Longhorn
Kubernetes pods don't keep their data when they restart. Longhorn creates persistent volumes backed by my SSD, with snapshot and backup support. Jellyfin needs this to remember its library. Nextcloud needs it to store files.

### Jellyfin
My self-hosted media server. I point it at a folder of movies and shows, it builds a library, and I can stream from any device on my network — phone, iPad, laptop, TV.

### Nextcloud
My personal cloud storage — like Google Drive but running on my own hardware. File sync, calendar, contacts, notes. Accessible from the Nextcloud mobile app on my phone and iPad.

### Cloudflare Tunnel (Future)
When I'm ready to access my homelab from outside my home network — at a café, while travelling — Cloudflare Tunnel creates an outbound-only encrypted connection from my cluster to Cloudflare's edge. No port forwarding. No exposing my home IP.

---

## The Series Plan

| Part | Topic |
|------|-------|
| Part 1 | What I'm Building (this post) |
| Part 2 | VMware Setup and VM Planning |
| Part 3 | Internal DNS with AdGuard Home |
| Part 4 | K3s Single Node Setup |
| Part 5 | MetalLB and Nginx Ingress |
| Part 6 | Persistent Storage with Longhorn |
| Part 7 | Jellyfin — Self-Hosted Media Server |
| Part 8 | Nextcloud — Personal Cloud Storage |
| Part 9 | Cloudflare Tunnel — Secure External Access |

Each part is a working system before moving to the next. I won't touch Cloudflare Tunnel until Jellyfin and Nextcloud are running locally first.

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
