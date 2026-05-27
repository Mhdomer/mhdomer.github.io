---
layout: post
title: "Homelab Part 4 — K3s Single Node Kubernetes Setup"
date: 2026-05-24 13:00:00 +0800
categories:
  - Homelab
tags:
  - Homelab
  - Kubernetes
  - K3s
  - kubectl
  - Helm
author: muhammed
description: Installing K3s on a single Ubuntu VM, configuring kubectl access from Windows, and installing Helm — the foundation everything else in the homelab is built on.
toc: true
pin: false
math: false
mermaid: false
---

## Why Kubernetes for a Home Media Server

Fair question. You could run Jellyfin and Nextcloud with a single `docker-compose.yml`. Simpler, faster, done in 20 minutes.

But the point of this homelab is to understand how modern infrastructure works — not just to get Jellyfin running. Kubernetes teaches you:

- How container orchestration actually works at the scheduler level
- How networking between services is handled (services, endpoints, DNS)
- How persistent storage is managed separately from compute
- How to deploy and update services without downtime
- Everything that maps directly to AWS EKS, Azure AKS, or GKE in a real job

K3s is the right choice here — it's full Kubernetes, just without the cloud-provider extras. It runs comfortably on a single VM and uses far less resources than a full kubeadm cluster.

---

## Pre-Installation Checks

SSH into the K3s VM:

```bash
ssh homelab@192.168.1.51
```

Verify swap is off (required for Kubernetes):

```bash
free -h
# Swap row must show 0
```

Verify the hostname is set:

```bash
hostname
# Should return: homelab-k3s
```

Check available resources:

```bash
nproc        # CPU cores
free -gh     # RAM in GB
df -h /      # Disk space
```

> `[SCREENSHOT]` — *Terminal showing nproc, free -gh, and df -h output before installation*

---

## Installing K3s

One command installs the entire K3s cluster:

```bash
curl -sfL https://get.k3s.io | sh -
```

What this does:
- Downloads the K3s binary
- Installs it as a systemd service (`k3s`)
- Starts the Kubernetes control plane (API server, scheduler, controller manager)
- Starts a single worker node (the same machine)
- Writes a kubeconfig to `/etc/rancher/k3s/k3s.yaml`

Wait about 30 seconds, then verify:

```bash
sudo k3s kubectl get nodes
```

Expected output:
```
NAME          STATUS   ROLES                  AGE   VERSION
homelab-k3s   Ready    control-plane,master   1m    v1.28.x+k3s1
```

> `[SCREENSHOT]` — *Terminal showing `k3s kubectl get nodes` with STATUS: Ready*

---

## What K3s Installs by Default

K3s comes with some components pre-installed that I'd have to add manually in a full kubeadm cluster:

| Component | What it is |
|-----------|-----------|
| CoreDNS | Internal DNS for the cluster (pods resolve each other by name) |
| Traefik | Default ingress controller (I'll replace this with Nginx) |
| Local Path Provisioner | Default storage class for persistent volumes |
| Klipper LB | Basic service load balancer (I'll replace with MetalLB) |

I'll disable Traefik and Klipper during installation because I'm using Nginx Ingress and MetalLB instead. Let me reinstall with those flags:

```bash
# First, uninstall the default installation
/usr/local/bin/k3s-uninstall.sh

# Reinstall with Traefik and Klipper disabled
curl -sfL https://get.k3s.io | sh -s - \
  --disable traefik \
  --disable servicelb
```

> `[SCREENSHOT]` — *K3s reinstall output with --disable flags, ending with service started successfully*

---

## Setting Up kubectl on the VM

`k3s kubectl` works but I prefer the standard `kubectl` command:

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Point kubectl at K3s config
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown homelab:homelab ~/.kube/config

# Test
kubectl get nodes
kubectl get pods -A           # all pods across all namespaces
```

> `[SCREENSHOT]` — *`kubectl get pods -A` showing CoreDNS, local-path-provisioner, and metrics-server pods running*

---

## Setting Up kubectl on Windows (Remote Access)

I want to run `kubectl` commands from my Windows machine without SSHing into the VM every time.

### Step 1 — Install kubectl on Windows

```powershell
# Using winget
winget install Kubernetes.kubectl

# Or download manually from kubernetes.io/releases
```

### Step 2 — Copy the kubeconfig from the VM

```bash
# On the VM — show the kubeconfig content
sudo cat /etc/rancher/k3s/k3s.yaml
```

Copy this content to your Windows machine at:
```
C:\Users\YourName\.kube\config
```

### Step 3 — Fix the server address

The kubeconfig file has `server: https://127.0.0.1:6443` — that's the loopback address, only valid inside the VM. Change it to the VM's actual IP:

```yaml
server: https://192.168.1.51:6443
```

### Step 4 — Test from Windows

```powershell
kubectl get nodes
kubectl get pods -A
```

> `[SCREENSHOT]` — *Windows Terminal running kubectl get nodes against the homelab K3s cluster*

---

## Installing Helm

Helm is the package manager for Kubernetes. Almost every application I'll deploy (Nginx Ingress, MetalLB, Longhorn, Jellyfin, Nextcloud) has a Helm chart. Instead of writing 200 lines of YAML from scratch, Helm lets me install with a few commands and a small values file.

```bash
# On the K3s VM
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

> `[SCREENSHOT]` — *`helm version` output showing version 3.x*

Add the common chart repositories I'll use:

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add metallb https://metallb.github.io/metallb
helm repo add longhorn https://charts.longhorn.io
helm repo add jetstack https://charts.jetstack.io       # cert-manager
helm repo update
```

---

## Understanding the Cluster Structure

Before moving on, here's how Kubernetes organises things:

```
Cluster
└── Namespaces (logical separation)
    ├── kube-system     → core Kubernetes components
    ├── kube-public     → cluster-wide public info
    ├── default         → where apps go if you don't specify
    ├── networking      → I'll put MetalLB + Nginx here
    ├── storage         → Longhorn goes here
    ├── media           → Jellyfin goes here
    └── cloud           → Nextcloud goes here
```

I create the namespaces I'll use:

```bash
kubectl create namespace networking
kubectl create namespace storage
kubectl create namespace media
kubectl create namespace cloud
kubectl get namespaces
```

> `[SCREENSHOT]` — *`kubectl get namespaces` showing all namespaces including the newly created ones*

---

## Useful kubectl Commands

Quick reference for navigating the cluster:

```bash
# Nodes
kubectl get nodes -o wide              # with IPs and OS info

# Pods
kubectl get pods -A                    # all namespaces
kubectl get pods -n media              # specific namespace
kubectl describe pod <name> -n media   # detailed info + events
kubectl logs <pod-name> -n media       # container logs
kubectl exec -it <pod-name> -n media -- bash  # shell into pod

# Services
kubectl get svc -A                     # all services with IPs/ports

# Events (useful for debugging)
kubectl get events -n media --sort-by='.lastTimestamp'

# Apply a manifest
kubectl apply -f manifest.yaml

# Delete a resource
kubectl delete -f manifest.yaml
```

---

## Where I Am Now

At the end of Part 4 I have:

- ✅ K3s running on `192.168.1.51` (Traefik and Klipper disabled)
- ✅ `kubectl` working both on the VM and from Windows
- ✅ Helm installed with all needed repos added
- ✅ Namespaces created for networking, storage, media, cloud
- ✅ Basic cluster navigation working

Next: MetalLB and Nginx Ingress — giving the cluster a real IP and routing traffic to the right services.

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
