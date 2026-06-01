---
layout: post
title: "Homelab Part 9 — Cloudflare Tunnel for Secure External Access"
date: 2026-05-24 18:00:00 +0800
categories:
  - Homelab
tags:
  - Homelab
  - Kubernetes
  - Cloudflare
  - Tunnel
  - Security
  - Networking
author: muhammed
description: Using Cloudflare Tunnel to securely expose Jellyfin and Nextcloud outside the home network — no port forwarding, no home IP exposure, just a clean public URL.
toc: true
pin: false
math: false
mermaid: false
---

## Why Not Just Port Forward

The obvious way to access your homelab from the internet is port forwarding — tell your router to forward port 80 and 443 to your K3s VM. Direct, simple, done.

The problems:

1. **Your home IP is exposed** — anyone who knows your URL can see your actual public IP address. With that, they can attempt brute-force attacks, scan for vulnerabilities, and know exactly where you live (roughly).
2. **IP changes** — most ISPs assign dynamic IPs. When it changes, your services go down and you need to update DNS.
3. **Firewall complexity** — you have to manage what's reachable and what isn't. One misconfiguration and you've opened a hole you didn't intend.

**Cloudflare Tunnel** solves all three:
- Your cluster makes an **outbound connection** to Cloudflare. No inbound ports open on your router.
- Your home IP is never exposed — traffic goes `User → Cloudflare → Tunnel → Your cluster`.
- Cloudflare's global network handles the TLS and DDoS protection in front of you.
- If your home IP changes, the tunnel reconnects automatically — DNS doesn't change.

The trade-off: Cloudflare can see your traffic (they terminate TLS on their edge before re-encrypting to your tunnel). If that's a concern, port forwarding with proper firewall rules is more private. For a homelab media server, I'm fine with it.

---

## Prerequisites

- A domain you own (I'm using a personal domain registered via Cloudflare)
- Cloudflare account with the domain added (free plan works)
- `cloudflared` CLI installed

If you don't have a domain, Cloudflare sells them cheaply and you can use a `.com` for around $10/year. The tunnel itself is free.

---

## Installing cloudflared

On the K3s VM:

```bash
# Download cloudflared
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 \
  -o /tmp/cloudflared

chmod +x /tmp/cloudflared
sudo mv /tmp/cloudflared /usr/local/bin/cloudflared

# Verify
cloudflared --version
```

> `[SCREENSHOT]` — *Terminal showing cloudflared version output*

---

## Authenticating with Cloudflare

```bash
cloudflared tunnel login
```

This prints a URL. Open it in a browser, log in to your Cloudflare account, and authorise the domain you want to use. `cloudflared` stores the certificate at `~/.cloudflared/cert.pem`.

> `[SCREENSHOT]` — *Browser showing the Cloudflare authorization page for the tunnel*

---

## Creating the Tunnel

```bash
cloudflared tunnel create homelab
```

This creates a tunnel with a unique ID. You'll see output like:

```
Created tunnel homelab with id abc123def456-...
Tunnel credentials written to /home/homelab/.cloudflared/abc123def456-....json
```

Note the tunnel ID — you'll need it.

> `[SCREENSHOT]` — *Terminal showing tunnel creation with the UUID*

---

## Configuring the Tunnel

Create the tunnel configuration file:

```bash
mkdir -p ~/.cloudflared
cat > ~/.cloudflared/config.yaml << 'EOF'
tunnel: abc123def456-...     # your tunnel ID
credentials-file: /home/homelab/.cloudflared/abc123def456-....json

ingress:
  - hostname: jellyfin.yourdomain.com
    service: http://192.168.1.200:80
    originRequest:
      httpHostHeader: jellyfin.home.lab

  - hostname: nextcloud.yourdomain.com
    service: http://192.168.1.200:80
    originRequest:
      httpHostHeader: nextcloud.home.lab

  - service: http_status:404
EOF
```

What's happening here:
- `jellyfin.yourdomain.com` (public) → traffic goes to `192.168.1.200` (Nginx Ingress) with the header `jellyfin.home.lab` so Nginx routes it correctly
- `nextcloud.yourdomain.com` (public) → same IP, different host header
- The final `service: http_status:404` is a catch-all — any other hostname returns 404

Replace `yourdomain.com` with your actual domain and the tunnel ID with yours.

---

## Creating DNS Records via Cloudflare

Instead of a manual DNS record, I use `cloudflared` to create CNAME records pointing to the tunnel:

```bash
cloudflared tunnel route dns homelab jellyfin.yourdomain.com
cloudflared tunnel route dns homelab nextcloud.yourdomain.com
```

This creates CNAME records in Cloudflare's DNS:
```
jellyfin.yourdomain.com  CNAME  abc123def456.cfargotunnel.com
nextcloud.yourdomain.com CNAME  abc123def456.cfargotunnel.com
```

> `[SCREENSHOT]` — *Cloudflare DNS dashboard showing the CNAME records for jellyfin and nextcloud*

---

## Running the Tunnel as a Kubernetes Deployment

Instead of running `cloudflared` as a systemd service on the VM, I run it inside Kubernetes — consistent with how everything else in the homelab is managed.

First, create a secret from the tunnel credentials file:

```bash
kubectl create secret generic cloudflared-credentials \
  --from-file=credentials.json=/home/homelab/.cloudflared/abc123def456-....json \
  --namespace networking
```

Then create a ConfigMap for the config:

```yaml
# cloudflared-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflared-config
  namespace: networking
data:
  config.yaml: |
    tunnel: abc123def456-...
    credentials-file: /etc/cloudflared/credentials.json
    ingress:
      - hostname: jellyfin.yourdomain.com
        service: http://ingress-nginx-controller.networking.svc.cluster.local:80
        originRequest:
          httpHostHeader: jellyfin.home.lab
      - hostname: nextcloud.yourdomain.com
        service: http://ingress-nginx-controller.networking.svc.cluster.local:80
        originRequest:
          httpHostHeader: nextcloud.home.lab
      - service: http_status:404
```

Notice the service URL changed — inside the cluster, I use the internal service name `ingress-nginx-controller.networking.svc.cluster.local` instead of the external IP. This is cleaner and works even if MetalLB IP changes.

```yaml
# cloudflared-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: networking
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          args:
            - tunnel
            - --config
            - /etc/cloudflared/config.yaml
            - run
          volumeMounts:
            - name: config
              mountPath: /etc/cloudflared/config.yaml
              subPath: config.yaml
            - name: credentials
              mountPath: /etc/cloudflared/credentials.json
              subPath: credentials.json
      volumes:
        - name: config
          configMap:
            name: cloudflared-config
        - name: credentials
          secret:
            secretName: cloudflared-credentials
```

I use `replicas: 2` for the tunnel — Cloudflare load-balances across both connections, giving me redundancy.

```bash
kubectl apply -f cloudflared-config.yaml
kubectl apply -f cloudflared-deployment.yaml

kubectl get pods -n networking
# cloudflared-xxx    1/1   Running
# cloudflared-yyy    1/1   Running
```

> `[SCREENSHOT]` — *`kubectl get pods -n networking` showing two cloudflared pods Running*

---

## Verifying the Tunnel

```bash
cloudflared tunnel info homelab
```

Should show `CONNECTIONS: 2` — one for each pod.

Now open `https://jellyfin.yourdomain.com` in a browser from your phone on mobile data (not WiFi — if you're on WiFi you're still on the local network). If Jellyfin loads, the tunnel is working.

> `[SCREENSHOT]` — *Jellyfin loading at jellyfin.yourdomain.com on a mobile browser with LTE (not WiFi)*

---

## TLS — Internal vs External

This is an important detail. My homelab now has two TLS layers:

| Where | Certificate | Issued By |
|-------|-------------|-----------|
| Browser → Cloudflare | Valid Let's Encrypt cert | Cloudflare (automatic) |
| Cloudflare → My tunnel | Encrypted tunnel (mTLS) | Cloudflare internal |
| Nginx → Pod | Self-signed cert | My homelab CA |

The public-facing certificate at `yourdomain.com` is a real, browser-trusted Let's Encrypt certificate that Cloudflare manages automatically. No browser warnings for external users.

The internal `.home.lab` certs (self-signed) are only used inside the cluster and on local network access — and I've already installed the homelab CA on my devices so those don't show warnings either.

---

## Locking Down External Access with Cloudflare Access (Optional)

Right now, anyone who knows my URLs could try to access Jellyfin and Nextcloud from the internet. Both have login screens, but I want an extra layer.

**Cloudflare Access** (Zero Trust) lets me put an authentication gate in front of any tunnel — before the request even reaches my server.

In the Cloudflare dashboard:
1. **Zero Trust → Access → Applications → Add an Application**
2. Choose: Self-hosted
3. Application domain: `jellyfin.yourdomain.com`
4. Policy: **Allow → Emails** → add my email address

Now when someone visits `jellyfin.yourdomain.com`, Cloudflare shows a login page first. Only my email gets access. Jellyfin's own login page is a second layer after that.

This is free for up to 50 users on the Cloudflare Zero Trust free plan.

> `[SCREENSHOT]` — *Cloudflare Access policy screen showing email-based authentication configured for jellyfin.yourdomain.com*

---

## Where I Am Now

At the end of Part 9 I have:

- ✅ Cloudflare Tunnel running as 2 pods in the `networking` namespace
- ✅ `jellyfin.yourdomain.com` and `nextcloud.yourdomain.com` publicly accessible
- ✅ No ports open on my router — all traffic is outbound-only
- ✅ Home IP never exposed to the internet
- ✅ Valid browser-trusted TLS certificate via Cloudflare
- ✅ Cloudflare Access protecting both services with email authentication

The homelab is complete.

---

## The Full Picture

Looking back at the complete stack that's now running:

```
Internet
    │
    ▼
Cloudflare Edge (TLS termination, DDoS protection, Access auth)
    │
    ▼
Cloudflare Tunnel (encrypted outbound connection from my cluster)
    │
    ▼
cloudflared pods (namespace: networking)
    │
    ▼
Nginx Ingress (192.168.1.200 — hostname-based routing)
    │
    ├── jellyfin.home.lab / jellyfin.yourdomain.com
    │       └── Jellyfin pod (namespace: media)
    │               └── /media/movies (hostPath) + Longhorn PVC (config)
    │
    └── nextcloud.home.lab / nextcloud.yourdomain.com
            └── Nextcloud pod (namespace: cloud)
                    └── Longhorn PVC (data) + MariaDB pod + Longhorn PVC (db)

AdGuard Home (192.168.1.50) → internal DNS for *.home.lab
MetalLB → 192.168.1.200-210 IP pool
Cert-Manager → auto-TLS for all internal services
Longhorn → persistent volumes for all stateful pods
```

All of this runs on a single laptop. Two VMs. One router. No cloud services for the actual data — just Cloudflare as a secure proxy for external access.

---

## What I Learned Building This

The goal wasn't just to have Jellyfin and Nextcloud running. It was to understand how each piece of modern cloud infrastructure fits together — and I now have hands-on experience with:

- **Hypervisors and VM networking** (VMware Workstation, Bridged mode)
- **DNS** (AdGuard Home, split-horizon DNS with internal rewrites)
- **Container orchestration** (Kubernetes, pods, deployments, services, namespaces)
- **Load balancing** (MetalLB, Layer 2 ARP advertisement)
- **Ingress and reverse proxying** (Nginx Ingress, hostname-based routing)
- **TLS certificate management** (Cert-Manager, ClusterIssuers, self-signed CAs)
- **Persistent storage** (Longhorn, PVCs, storage classes, snapshots)
- **Zero-trust networking** (Cloudflare Tunnel, no open inbound ports)

Every one of these maps directly to how AWS, GCP, and Azure build their infrastructure — just with managed services instead of self-hosted ones. The concepts are identical.

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
