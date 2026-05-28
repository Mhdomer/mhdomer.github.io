---
layout: post
title: "Homelab Part 5 — MetalLB and Nginx Ingress"
date: 2026-05-24 14:00:00 +0800
categories:
  - Homelab
tags:
  - Homelab
  - Kubernetes
  - MetalLB
  - Nginx
  - Ingress
  - Networking
author: muhammed
description: Installing MetalLB to give Kubernetes real local IPs, and Nginx Ingress to route traffic to the right service based on hostname — the networking layer the entire homelab runs on.
toc: true
pin: false
math: false
mermaid: false
---

## The Problem This Solves

My K3s cluster is running but I can't access anything from my home network yet. Here's why:

Kubernetes services use virtual IPs that only exist inside the cluster. When I create a service for Jellyfin, it gets something like `10.43.87.23` — a cluster-internal IP that nothing outside the VM can reach.

In cloud Kubernetes (EKS, GKE), when you create a `LoadBalancer` service, the cloud automatically provisions a real IP and a load balancer. On bare metal, that cloud integration doesn't exist.

**MetalLB** solves the LoadBalancer problem — it gives Kubernetes the ability to assign real IPs from my home network to services.

**Nginx Ingress** solves the routing problem — instead of a separate IP/port for every service, one IP handles all traffic and routes based on the hostname.

---

## IP Planning

I need to reserve a small IP range for MetalLB. These IPs must be:
- On the same subnet as my home network (`192.168.1.0/24`)
- Outside my router's DHCP range (so the router won't assign them to other devices)

My router's DHCP range is typically `192.168.1.100–192.168.1.200`. I'll tell MetalLB to use `192.168.1.200–192.168.1.210` — 11 addresses, more than enough.

I also update my router's DHCP range to stop at `192.168.1.199` so it doesn't conflict.

> `[SCREENSHOT]` — *Router DHCP settings showing range ending at .199, leaving .200+ free for MetalLB*

---

## Installing MetalLB

```bash
helm install metallb metallb/metallb \
  --namespace networking \
  --wait
```

Wait for pods to be ready:

```bash
kubectl get pods -n networking
# NAME                                  READY   STATUS    
# metallb-controller-xxx                1/1     Running   
# metallb-speaker-xxx                   1/1     Running
```

> `[SCREENSHOT]` — *`kubectl get pods -n networking` showing MetalLB controller and speaker both Running*

### Configure the IP Pool

MetalLB needs to know which IP range it can hand out. I create two manifests:

```yaml
# metallb-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: networking
spec:
  addresses:
    - 192.168.1.200-192.168.1.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: networking
spec:
  ipAddressPools:
    - homelab-pool
```

```bash
kubectl apply -f metallb-pool.yaml
```

`L2Advertisement` tells MetalLB to use Layer 2 (ARP) to advertise IPs — meaning it responds to ARP requests on my network for those IPs, making them reachable from any device on the LAN.

> `[SCREENSHOT]` — *`kubectl apply -f metallb-pool.yaml` output showing both resources created*

### Verify MetalLB Works

I create a quick test service:

```yaml
# test-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: test-lb
  namespace: default
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: nonexistent
```

```bash
kubectl apply -f test-lb.yaml
kubectl get svc test-lb
# EXTERNAL-IP should show 192.168.1.200
kubectl delete -f test-lb.yaml
```

> `[SCREENSHOT]` — *`kubectl get svc test-lb` showing EXTERNAL-IP: 192.168.1.200 assigned by MetalLB*

---

## Installing Nginx Ingress

Nginx Ingress is the traffic router. One `LoadBalancer` service gets `192.168.1.200` from MetalLB, and Nginx Ingress reads the hostname on each request to decide which Kubernetes service to forward it to.

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace networking \
  --set controller.service.loadBalancerIP=192.168.1.200
```

Pinning the IP to `192.168.1.200` ensures Nginx Ingress always gets that specific IP from MetalLB — important because my DNS records point there.

Wait for it to be ready:

```bash
kubectl get pods -n networking
kubectl get svc -n networking
# ingress-nginx-controller   LoadBalancer   192.168.1.200   80:xxx/TCP,443:xxx/TCP
```

> `[SCREENSHOT]` — *`kubectl get svc -n networking` showing ingress-nginx-controller with EXTERNAL-IP 192.168.1.200*

---

## Testing the Ingress

I deploy a simple test app to verify routing works:

```yaml
# hello-world.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: nginxdemos/hello
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: default
spec:
  selector:
    app: hello
  ports:
    - port: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: hello.home.lab
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello
                port:
                  number: 80
```

```bash
kubectl apply -f hello-world.yaml
```

Add `hello.home.lab` to AdGuard DNS rewrites pointing to `192.168.1.200`, then open `http://hello.home.lab` in my browser.

> `[SCREENSHOT]` — *Browser showing the Nginx hello world page at hello.home.lab*

Clean up:

```bash
kubectl delete -f hello-world.yaml
```

---

## Installing Cert-Manager (TLS)

I want `https://jellyfin.home.lab` — not just `http://`. Cert-Manager automates TLS certificate issuance and renewal.

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace networking \
  --set installCRDs=true
```

Wait for pods:

```bash
kubectl get pods -n networking | grep cert-manager
```

### Create a Self-Signed CA for Internal Services

For my internal `.home.lab` services I'll use a self-signed certificate authority. Let's Encrypt only works for public domains — I'll use that in Part 9 when I set up Cloudflare Tunnel.

```yaml
# internal-ca.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: homelab-ca
  namespace: networking
spec:
  isCA: true
  commonName: homelab-ca
  secretName: homelab-ca-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: homelab-ca-issuer
spec:
  ca:
    secretName: homelab-ca-secret
```

```bash
kubectl apply -f internal-ca.yaml
kubectl get clusterissuers
```

> `[SCREENSHOT]` — *`kubectl get clusterissuers` showing selfsigned-issuer and homelab-ca-issuer both Ready*

### Trust the CA on My Devices

Since it's self-signed, browsers will show a warning. To fix this I export the CA cert and install it as a trusted CA on my devices:

```bash
kubectl get secret homelab-ca-secret -n networking \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > homelab-ca.crt
```

Copy `homelab-ca.crt` to Windows and install it:
- Double-click the file → Install Certificate → Local Machine → Trusted Root CAs

On iPhone/iPad: AirDrop the cert file → tap it → Settings → downloaded profile → install.

> `[SCREENSHOT]` — *Windows Certificate Import Wizard showing the homelab CA being installed to Trusted Root CAs*

---

## How Ingress + Cert-Manager Work Together

When I deploy Jellyfin, my Ingress manifest will look like this:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jellyfin
  namespace: media
  annotations:
    cert-manager.io/cluster-issuer: homelab-ca-issuer
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - jellyfin.home.lab
      secretName: jellyfin-tls
  rules:
    - host: jellyfin.home.lab
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: jellyfin
                port:
                  number: 8096
```

The annotation `cert-manager.io/cluster-issuer: homelab-ca-issuer` tells Cert-Manager to automatically issue a TLS cert for `jellyfin.home.lab` and store it in the secret `jellyfin-tls`. Nginx Ingress picks up that secret and serves HTTPS automatically.

---

## The Traffic Flow So Far

```
Browser: https://jellyfin.home.lab
    │
    ▼
AdGuard DNS: jellyfin.home.lab → 192.168.1.200
    │
    ▼
MetalLB: 192.168.1.200 → Nginx Ingress pod
    │
    ▼
Nginx Ingress: hostname = jellyfin.home.lab → Jellyfin service
    │
    ▼
Jellyfin pod
```

Everything from DNS to TLS is now automated.

---

## Where I Am Now

At the end of Part 5 I have:

- ✅ MetalLB handing out IPs from `192.168.1.200–192.168.1.210`
- ✅ Nginx Ingress running at `192.168.1.200`
- ✅ Cert-Manager issuing TLS certificates automatically
- ✅ Homelab CA installed as trusted on my devices
- ✅ Traffic routing by hostname working and tested

Next: persistent storage with Longhorn — so Jellyfin and Nextcloud don't lose their data when pods restart.

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
