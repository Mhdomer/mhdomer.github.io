---
layout: post
title: "Homelab Infrastructure Breakdown — How It All Comes Together"
date: 2026-05-24 12:00:00 +0800
categories:
  - General Notes
tags:
  - Homelab
  - Kubernetes
  - Proxmox
  - Infrastructure
  - DevOps
  - Reference
author: muhammed
description: A breakdown of a real homelab running Proxmox, K3s, Istio, ArgoCD, and a full observability stack on a single laptop — how every layer connects and why each piece is there.
toc: true
pin: false
math: false
mermaid: false
---

## The Big Picture

This homelab runs an entire production-like cloud infrastructure on a single laptop. The goal isn't to replace AWS — it's to understand how every layer of modern infrastructure actually works by building it yourself.

```
Internet
    │
    ▼
Cloudflare (DNS + Tunnel)
    │
    ▼
Your Laptop (Proxmox Hypervisor)
    │
    ├── VM: AdGuard Home (DNS)
    │
    └── VM: K3s Kubernetes Cluster
            │
            ├── MetalLB (Load Balancer)
            ├── Istio (Service Mesh + Ingress)
            ├── Cert-Manager (TLS)
            ├── Authentik (SSO)
            ├── ArgoCD (GitOps)
            ├── Longhorn (Storage)
            └── Grafana + VictoriaMetrics + Jaeger (Observability)
```

One machine. One laptop. Everything from DNS to GitOps deployments to distributed storage.

---

## Layer 1 — The Hardware

The foundation is a single laptop running everything. In the reference setup it's a Dell with 32GB RAM, 24 logical CPU cores, and an NVMe SSD. You have 40GB RAM which is actually more than enough.

**What the hardware needs to support:**
- Proxmox itself: ~2GB RAM, 2 cores
- AdGuard VM: ~512MB RAM, 1 core
- K3s VM: the rest — roughly 20–30GB RAM and 8–16 cores
- Longhorn (distributed storage): needs fast SSD I/O, not HDD

**The 4TB external SSD** in the reference setup is for Longhorn backups and VM disk images that don't fit on the internal drive. Without it you can still run everything — just with less storage capacity.

---

## Layer 2 — Proxmox (The Hypervisor)

Proxmox VE is a bare-metal hypervisor. You install it directly on the laptop (replacing or dual-booting the OS), and then everything else runs as virtual machines inside it.

**Why Proxmox instead of just running Docker/K3s directly?**
- Isolation — each service lives in its own VM with its own kernel
- Snapshots — you can checkpoint a VM before breaking something
- Networking — Proxmox gives you virtual bridges, VLANs, and software-defined networks
- It's what real infrastructure teams use (Proxmox is used in production datacenters)

**What you create inside Proxmox:**
- `vm-dns` — AdGuard Home (lightweight, 512MB)
- `vm-k3s` — the Kubernetes node (the big one, most resources)

You could also add more VMs later — a Vault VM, a CI runner VM, a firewall VM — all isolated from each other on the same physical machine.

---

## Layer 3 — AdGuard Home (DNS Forwarder)

AdGuard Home runs in its own small VM and acts as the DNS server for your entire home network. Your router points all DNS queries to it.

**What it does:**
- Blocks ads and trackers at the DNS level for every device on your network
- Resolves internal hostnames — so `grafana.home.lab` resolves to the right K3s service IP
- Forwards external queries to upstream resolvers (1.1.1.1, 8.8.8.8)

**Why it matters for the homelab:**
Without internal DNS, you'd have to hardcode IPs everywhere. With AdGuard as your DNS, you can give every service a clean hostname and access it from your phone, iPad, or laptop on the same network by name.

---

## Layer 4 — Cloudflare (DNS + Zero Trust Tunnel)

Cloudflare handles two things:

**1. Public DNS** — your domain (e.g. `yourdomain.com`) is registered and managed in Cloudflare. They resolve it for the internet.

**2. Cloudflare Tunnel (Zero Trust)** — this is the clever part. Instead of opening ports on your router (which exposes your home IP), you run a small `cloudflared` daemon inside your K3s cluster. It creates an outbound-only encrypted tunnel to Cloudflare's edge.

```
User types grafana.yourdomain.com
    → Cloudflare edge receives request
    → Forwards through the tunnel to cloudflared inside K3s
    → cloudflared sends to Istio Ingress Gateway
    → Istio routes to Grafana pod
```

**Why this is better than port forwarding:**
- Your home IP is never exposed
- Cloudflare handles DDoS protection, rate limiting, and TLS termination at the edge
- Works even if your ISP blocks inbound connections (most do)
- Free tier is more than enough for a homelab

---

## Layer 5 — K3s (Lightweight Kubernetes)

K3s is a stripped-down Kubernetes distribution made for resource-constrained environments. It runs as a single binary, uses SQLite instead of etcd by default, and removes cloud-provider dependencies.

**It's still real Kubernetes** — every kubectl command, every YAML manifest, every concept you learn in K3s applies directly to EKS, GKE, and AKS.

**What's running inside:**

| Component | Type | What it does |
|-----------|------|-------------|
| MetalLB | Load Balancer | Gives Kubernetes services real IPs on your local network |
| Istio Ingress | Gateway | Handles all incoming traffic, TLS, routing rules |
| Istio Service Mesh | Mesh | Encrypts pod-to-pod traffic with mTLS automatically |
| Cert-Manager | Controller | Automatically issues and renews TLS certs from Let's Encrypt |
| Authentik | Identity Provider | SSO — one login for all your services |
| ArgoCD | GitOps | Watches a Git repo and keeps the cluster in sync with it |
| Longhorn | Storage | Distributed block storage for pods that need persistent data |
| Grafana | Dashboard | Visualises all metrics, logs, and traces |
| VictoriaMetrics | Metrics | Stores metrics (drop-in Prometheus replacement, lighter) |
| VictoriaLogs | Logs | Stores logs from all pods |
| Jaeger | Tracing | Traces requests across services |

---

## How Traffic Flows — End to End

Here's what happens when someone on the internet hits `grafana.yourdomain.com`:

```
1. DNS query → Cloudflare resolves yourdomain.com
2. Request hits Cloudflare edge
3. Cloudflare Tunnel delivers it to cloudflared pod in K3s
4. cloudflared forwards to Istio Ingress Gateway
5. Cert-Manager has already issued a TLS cert → connection is HTTPS
6. Istio checks the routing rules — this hostname maps to the Grafana service
7. Istio Service Mesh encrypts the pod-to-pod hop with mTLS
8. Request arrives at Grafana pod
9. Authentik SSO checks if the user is logged in before Grafana loads
```

Nine hops. All automated. All configured as Kubernetes YAML.

---

## MetalLB — Why Kubernetes Needs This on Bare Metal

In AWS/GCP, when you create a `Service` of type `LoadBalancer`, the cloud provider automatically provisions a real IP and a load balancer. On bare metal (your laptop), there's no cloud provider — Kubernetes doesn't know how to assign IPs.

MetalLB fills that gap. You give it a small IP range from your local network (e.g. `192.168.1.200–192.168.1.220`) and it acts as a software load balancer, assigning real local IPs to Kubernetes services.

**Example:** Istio Ingress Gateway gets `192.168.1.200` from MetalLB. Your router or AdGuard can then point `*.home.lab` to that IP.

---

## Istio — The Service Mesh

Istio is the most complex piece. It runs a sidecar proxy (`envoy`) next to every pod. All network traffic goes through the sidecar — which means Istio can:

- Encrypt every pod-to-pod connection with mTLS automatically (no app code changes)
- Apply traffic policies (canary deployments, circuit breakers, retries)
- Collect metrics and traces from every connection
- Enforce access control between services

**Istio Ingress Gateway** is the front door — it's the single entry point for all external traffic into the cluster, replacing nginx/Traefik as the ingress controller.

---

## ArgoCD — GitOps

ArgoCD watches a Git repository. Every Kubernetes manifest (deployments, services, config maps) lives in Git. When you push a change:

```
Git push → ArgoCD detects diff → Applies to cluster → Cluster matches Git
```

**Why this matters:**
- Your cluster state is always described in code
- Rollback = git revert
- You can see exactly what changed and when
- It's how real platform teams manage production

---

## Longhorn — Distributed Storage

Pods are stateless by default — if a pod restarts, its data is gone. Longhorn solves this by creating persistent volumes backed by your SSD, with snapshots and backup support.

For a single-node setup (one laptop) Longhorn is slightly overkill — you could use `local-path-provisioner` (built into K3s) instead. But Longhorn teaches you how production storage works in Kubernetes.

---

## Authentik — SSO for Everything

Authentik is a self-hosted identity provider. Instead of creating separate logins for Grafana, ArgoCD, and every other service, you configure them all to authenticate through Authentik using OIDC or SAML.

One login. One place to revoke access. Proper MFA.

This is the same pattern enterprises use — just Okta or Azure AD instead of Authentik.

---

## Observability Stack

| Tool | What it stores | Access |
|------|---------------|--------|
| VictoriaMetrics | CPU, memory, request rates, latency | Grafana |
| VictoriaLogs | Pod logs, system logs | Grafana |
| Jaeger | Request traces across services | Jaeger UI |
| Grafana | Dashboards combining all three | Browser |

Istio automatically exports metrics and traces from every service. You get observability without instrumenting your apps.

---

## Running This on Your Laptop

**What's realistic:**
- Run everything except GPU scheduling (just skip that section)
- Use K3s single node (no multi-node cluster needed)
- Use `local-path-provisioner` instead of Longhorn initially — add Longhorn later
- Start with just K3s + MetalLB + Ingress. Add Istio after you're comfortable

**RAM budget estimate:**

| Component | RAM |
|-----------|-----|
| Proxmox host | 2GB |
| AdGuard VM | 512MB |
| K3s VM base | 4GB |
| Istio sidecars | 2–4GB |
| Monitoring stack | 3–4GB |
| ArgoCD + Authentik | 2GB |
| Longhorn | 1GB |
| **Total** | ~18–22GB |

With 40GB you have headroom to run your normal laptop workload alongside it — Proxmox VMs can be paused or have resource limits set so your day-to-day work isn't impacted.

**Recommended order to build:**
1. Install Proxmox
2. Create K3s VM + AdGuard VM
3. Deploy MetalLB + Ingress
4. Connect Cloudflare Tunnel
5. Add Cert-Manager + TLS
6. Deploy ArgoCD and manage everything else through Git from that point
7. Add Authentik
8. Add Longhorn
9. Add full observability stack last

Each step is a working system before moving to the next.

---

## Key Takeaways

- Proxmox gives you a real hypervisor layer — isolation, snapshots, and network control
- K3s is production Kubernetes, just lighter — everything you learn here transfers directly
- Cloudflare Tunnel removes the need to expose your home IP
- Istio teaches you service mesh concepts that are used in large-scale production clusters
- ArgoCD teaches you GitOps — the dominant deployment pattern in modern platform engineering
- The full stack maps directly to what AWS/GCP managed services do — you're just running the open-source equivalents yourself

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
