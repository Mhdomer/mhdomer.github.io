---
layout: post
title: "Homelab Part 8 — Nextcloud Personal Cloud Storage"
date: 2026-05-24 17:00:00 +0800
categories:
  - Homelab
tags:
  - Homelab
  - Kubernetes
  - Nextcloud
  - Storage
  - Selfhosted
author: muhammed
description: Deploying Nextcloud on Kubernetes with a MariaDB database and Longhorn storage — giving myself a self-hosted Google Drive accessible from any device on the network.
toc: true
pin: false
math: false
mermaid: false
---

## What Nextcloud Is

Nextcloud is a self-hosted cloud storage platform — like Google Drive or iCloud, but running entirely on my hardware. I can:

- Sync files between devices automatically (phone, laptop, iPad)
- Access files via a browser or the Nextcloud mobile app
- Store photos, documents, notes — anything
- Share files with links (within my network, or via Cloudflare Tunnel later)
- Add apps inside Nextcloud: calendar, contacts, tasks, notes

The data never leaves my home network. No cloud company has a copy.

---

## What Nextcloud Needs

Nextcloud requires more than just a container — it needs a **database**. Nextcloud stores all its metadata (file names, user accounts, sharing permissions, settings) in a relational database. I'm using MariaDB.

So I'm deploying two pods:
1. **MariaDB** — the database backend
2. **Nextcloud** — the application itself

Both get Longhorn volumes for their data.

---

## The Storage Plan

| Resource | Size | What It Stores |
|----------|------|---------------|
| MariaDB PVC | 10Gi | The database (file metadata, users, sharing) |
| Nextcloud Data PVC | 50Gi | The actual files uploaded to Nextcloud |

50GB is my starting point. Longhorn volumes can be expanded later without downtime.

---

## Deploying MariaDB

I create the database first, then Nextcloud connects to it.

```yaml
# mariadb.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-data
  namespace: cloud
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-secret
  namespace: cloud
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: "your-root-password-here"
  MYSQL_DATABASE: nextcloud
  MYSQL_USER: nextcloud
  MYSQL_PASSWORD: "your-nextcloud-db-password-here"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb
  namespace: cloud
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
        - name: mariadb
          image: mariadb:10.11
          envFrom:
            - secretRef:
                name: mariadb-secret
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: mariadb-data
---
apiVersion: v1
kind: Service
metadata:
  name: mariadb
  namespace: cloud
spec:
  selector:
    app: mariadb
  ports:
    - port: 3306
      targetPort: 3306
```

Replace `your-root-password-here` and `your-nextcloud-db-password-here` with actual strong passwords before applying.

```bash
kubectl apply -f mariadb.yaml
kubectl get pods -n cloud
# mariadb-xxx    1/1   Running
```

> `[SCREENSHOT]` — *`kubectl get pods -n cloud` showing mariadb pod Running*

---

## Deploying Nextcloud

```yaml
# nextcloud.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-data
  namespace: cloud
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud
  namespace: cloud
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nextcloud
  template:
    metadata:
      labels:
        app: nextcloud
    spec:
      containers:
        - name: nextcloud
          image: nextcloud:28-apache
          ports:
            - containerPort: 80
          env:
            - name: NEXTCLOUD_ADMIN_USER
              value: admin
            - name: NEXTCLOUD_ADMIN_PASSWORD
              value: "your-admin-password-here"
            - name: NEXTCLOUD_TRUSTED_DOMAINS
              value: nextcloud.home.lab
            - name: MYSQL_HOST
              value: mariadb
            - name: MYSQL_DATABASE
              value: nextcloud
            - name: MYSQL_USER
              value: nextcloud
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mariadb-secret
                  key: MYSQL_PASSWORD
          volumeMounts:
            - name: data
              mountPath: /var/www/html
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: nextcloud-data
---
apiVersion: v1
kind: Service
metadata:
  name: nextcloud
  namespace: cloud
spec:
  selector:
    app: nextcloud
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nextcloud
  namespace: cloud
  annotations:
    cert-manager.io/cluster-issuer: homelab-ca-issuer
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - nextcloud.home.lab
      secretName: nextcloud-tls
  rules:
    - host: nextcloud.home.lab
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nextcloud
                port:
                  number: 80
```

```bash
kubectl apply -f nextcloud.yaml
```

Watch the pod start — Nextcloud takes 1–2 minutes on first boot because it initialises the database:

```bash
kubectl get pods -n cloud -w
# nextcloud-xxx    0/1   ContainerCreating ...
# nextcloud-xxx    1/1   Running           ...
```

> `[SCREENSHOT]` — *`kubectl get pods -n cloud` showing both mariadb and nextcloud pods Running*

---

## First Login

Open `https://nextcloud.home.lab` in a browser.

Log in with:
- Username: `admin`
- Password: whatever you set in `NEXTCLOUD_ADMIN_PASSWORD`

> `[SCREENSHOT]` — *Nextcloud login screen at nextcloud.home.lab*

Nextcloud shows a welcome screen and offers to install recommended apps. I install:
- **Files** (always on)
- **Photos**
- **Calendar**
- **Contacts**

> `[SCREENSHOT]` — *Nextcloud files view — the main dashboard after login*

---

## Creating User Accounts

The admin account is for administration. I create a regular user for daily use:

**Top-right menu → Administration → Users → New user:**
- Username: `muhammed`
- Password: set a strong one
- Quota: `unlimited` (or set a limit if you share with family)

> `[SCREENSHOT]` — *Nextcloud user management screen showing the new user created*

---

## Setting Up File Sync on Devices

### Windows/Mac — Nextcloud Desktop Client

Download from `nextcloud.com/install/#install-clients` and install. 

Set up:
- Server address: `https://nextcloud.home.lab`
- Login with the user account (not admin)
- Choose which folders to sync

Files in the sync folder appear in Windows Explorer automatically.

> `[SCREENSHOT]` — *Nextcloud desktop client showing sync status and file list*

### iPhone/iPad — Nextcloud Mobile App

Download from the App Store. 

Setup:
- Add account → Server: `https://nextcloud.home.lab`
- Log in

The app syncs photos automatically if you enable camera upload. All files are accessible offline if you mark them as favorites.

> `[SCREENSHOT]` — *Nextcloud mobile app showing files and camera upload settings*

---

## Fixing the Nextcloud Warnings

After the first login, Nextcloud's admin panel often shows warnings. Common ones and how to fix them:

### Warning: Trusted Domain Not Set

If Nextcloud shows "You are accessing the server from an untrusted domain":

```bash
# Run occ command inside the pod
kubectl exec -n cloud deployment/nextcloud -- \
  php occ config:system:set trusted_domains 0 --value="nextcloud.home.lab"
```

This is already handled by the `NEXTCLOUD_TRUSTED_DOMAINS` env var, but if it still appears, the `occ` command above fixes it permanently.

### Warning: Security & Setup Warnings

In the admin panel (**Administration → Overview**), Nextcloud checks for configuration issues. Most can be resolved:

```bash
# Fix missing background jobs (use system cron instead of AJAX)
kubectl exec -n cloud deployment/nextcloud -- \
  php occ background:cron
```

For the memory cache warning, add to Nextcloud's config:

```bash
kubectl exec -n cloud deployment/nextcloud -- \
  php occ config:system:set memcache.local --value="\\OC\\Memcache\\APCu"
```

> `[SCREENSHOT]` — *Nextcloud admin overview showing green checkmarks — no critical warnings*

---

## Uploading and Accessing Files

The Nextcloud web interface is straightforward — drag and drop files in the browser, create folders, and share links.

The real power is the desktop sync client. Once configured, any file I put in the Nextcloud folder on my laptop is available instantly on my phone and iPad. Documents, PDFs, photos — all synced automatically, no manual steps.

---

## Nextcloud vs Google Drive — Key Differences

| Feature | Google Drive | Nextcloud |
|---------|-------------|-----------|
| Storage cost | 15GB free, then paid | Limited by your disk |
| Privacy | Google sees your files | Only you see your files |
| Offline access | Limited | Full (with sync) |
| Custom apps | No | Calendar, Contacts, Notes, etc. |
| External access | Always on | Needs Cloudflare Tunnel (Part 9) |
| Setup complexity | Zero | What we just did |

The trade-off is real — Google Drive just works everywhere from the start. Nextcloud takes setup, but what I build here I own completely.

---

## Where I Am Now

At the end of Part 8 I have:

- ✅ MariaDB running in the `cloud` namespace with a Longhorn PVC
- ✅ Nextcloud running and connected to MariaDB
- ✅ File storage with 50GB Longhorn volume (expandable)
- ✅ HTTPS access at `https://nextcloud.home.lab`
- ✅ Desktop sync client configured on Windows
- ✅ Mobile app connected on iPhone/iPad

Both Jellyfin and Nextcloud are now running. The homelab is functional on the home network.

Next: Cloudflare Tunnel — the optional last piece, for accessing Jellyfin and Nextcloud securely from anywhere outside the home network.

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
