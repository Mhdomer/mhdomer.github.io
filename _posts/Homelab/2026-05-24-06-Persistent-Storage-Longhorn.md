---
layout: post
title: "Homelab Part 6 — Persistent Storage with Longhorn"
date: 2026-05-24 15:00:00 +0800
categories:
  - Homelab
tags:
  - Homelab
  - Kubernetes
  - Longhorn
  - Storage
  - PersistentVolume
author: muhammed
description: Installing Longhorn to give Kubernetes pods real persistent storage — so Jellyfin remembers its library and Nextcloud keeps its files when pods restart.
toc: true
pin: false
math: false
mermaid: false
---

## The Problem With Stateless Pods

By default, Kubernetes pods are stateless. When a pod restarts — because of an update, a crash, or a node reboot — it starts fresh with no memory of what it had before. For a web server serving static HTML, that's fine. For Jellyfin and Nextcloud, it's a disaster.

Jellyfin needs to remember its media library index, user accounts, and watch history. Nextcloud needs to keep every file you've uploaded. Both need storage that survives pod restarts.

**Persistent Volumes (PVs)** solve this in Kubernetes — they're storage resources that exist independently of any pod. A pod claims a volume (via a PersistentVolumeClaim), uses it, and when the pod restarts it mounts the same volume again.

K3s comes with a built-in storage class called `local-path` — it creates directories on the node's filesystem. That works, but it has no snapshots, no replication, and no backup support.

**Longhorn** is a proper distributed block storage system for Kubernetes. It runs inside the cluster, creates replicated volumes backed by the node's disks, and supports snapshots and backups. This is what I'm using for Jellyfin and Nextcloud.

---

## Longhorn Prerequisites

Longhorn needs `open-iscsi` on the host node. SSH into the K3s VM:

```bash
ssh homelab@192.168.1.51

# Install the requirement
sudo apt install -y open-iscsi
sudo systemctl enable iscsid
sudo systemctl start iscsid
```

Verify:

```bash
sudo systemctl status iscsid
# Should show: active (running)
```

> `[SCREENSHOT]` — *Terminal showing iscsid status: active (running)*

---

## Installing Longhorn

```bash
helm install longhorn longhorn/longhorn \
  --namespace storage \
  --set defaultSettings.defaultDataPath="/var/lib/longhorn"
```

This takes 2–3 minutes. Wait for all pods to be ready:

```bash
kubectl get pods -n storage
```

Expected output (lots of pods — Longhorn is a full storage system):

```
NAME                                        READY   STATUS    
longhorn-manager-xxx                        1/1     Running   
longhorn-driver-deployer-xxx                1/1     Running   
instance-manager-xxx                        1/1     Running   
engine-image-xxx                            1/1     Running   
longhorn-ui-xxx                             1/1     Running   
csi-attacher-xxx                            1/1     Running   
csi-provisioner-xxx                         1/1     Running   
csi-resizer-xxx                             1/1     Running   
csi-snapshotter-xxx                         1/1     Running   
longhorn-csi-plugin-xxx                     2/2     Running   
```

> `[SCREENSHOT]` — *`kubectl get pods -n storage` showing all Longhorn pods Running*

---

## Setting Longhorn as Default Storage Class

K3s installs `local-path` as the default storage class. I want Longhorn to be the default so that any PVC without an explicit `storageClassName` gets a Longhorn volume:

```bash
# Remove default from local-path
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Set Longhorn as default
kubectl patch storageclass longhorn \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify
kubectl get storageclass
# longhorn (default)   driver.longhorn.io   ...
# local-path           rancher.io/local-path ...
```

> `[SCREENSHOT]` — *`kubectl get storageclass` showing longhorn as (default)*

---

## Accessing the Longhorn UI

Longhorn has a web dashboard. I expose it through the Ingress I already have:

```yaml
# longhorn-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn
  namespace: storage
  annotations:
    cert-manager.io/cluster-issuer: homelab-ca-issuer
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - longhorn.home.lab
      secretName: longhorn-tls
  rules:
    - host: longhorn.home.lab
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: longhorn-frontend
                port:
                  number: 80
```

```bash
kubectl apply -f longhorn-ingress.yaml
```

Add `longhorn.home.lab → 192.168.1.200` to AdGuard DNS rewrites, then open `https://longhorn.home.lab`.

> `[SCREENSHOT]` — *Longhorn dashboard showing the node, available storage, and volumes list*

---

## Understanding How Storage Works End-to-End

Before deploying applications, it helps to understand the flow:

```
Application (Deployment)
    │
    └── PersistentVolumeClaim (PVC)   ← "I need 10GB of storage"
            │
            └── PersistentVolume (PV)  ← actual storage, auto-created by Longhorn
                    │
                    └── Longhorn Volume ← replicated data on node disk
```

The application doesn't know or care where the storage comes from. It just mounts a path. Longhorn handles the actual disk management underneath.

---

## Creating a Test Volume

I'll verify Longhorn works before depending on it for Jellyfin and Nextcloud:

```yaml
# test-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f test-pvc.yaml
kubectl get pvc test-pvc
# STATUS should change from Pending to Bound within 30 seconds
```

> `[SCREENSHOT]` — *`kubectl get pvc test-pvc` showing STATUS: Bound and VOLUME pointing to a longhorn-xxx volume*

Clean up:

```bash
kubectl delete -f test-pvc.yaml
```

---

## Storage for Jellyfin and Nextcloud — What They Need

Planning the volumes before deploying the apps:

| Application | What Needs Storage | Size | Access Mode |
|-------------|-------------------|------|-------------|
| Jellyfin | Config + metadata | 5GB | ReadWriteOnce |
| Jellyfin | Media library | 100GB+ | ReadWriteOnce |
| Nextcloud | App data + files | 50GB | ReadWriteOnce |
| Nextcloud | Database (MariaDB) | 10GB | ReadWriteOnce |

The media library volume for Jellyfin is big — how much space you allocate depends on how many movies you have. I'll start with 100GB and Longhorn can resize it later.

**Important note about media files:** Longhorn volumes store data inside the VM's 200GB disk. My movies aren't going to live in a Longhorn volume — they'll be in a directory I mount directly into the Jellyfin pod. Longhorn only stores Jellyfin's config and database (the metadata, watch history, user accounts). The actual video files live in a separate directory I control directly.

---

## The Media Directory

I create the directory where my movies will actually live:

```bash
# On the K3s VM
sudo mkdir -p /media/movies
sudo mkdir -p /media/shows
sudo chown -R homelab:homelab /media
```

I'll copy movies into these directories using SCP or a network share. When Jellyfin is deployed in Part 7, I'll mount this directory into the container using a `hostPath` volume — a direct path from the VM's filesystem into the pod.

> `[SCREENSHOT]` — *Terminal showing the /media/movies directory created and disk space with `df -h /media`*

---

## Snapshot Configuration

One of the main reasons I chose Longhorn over `local-path` is snapshots. I configure automatic recurring snapshots so I don't lose data:

In the Longhorn UI:
- **Setting → Recurring Job** → Create
  - Name: `daily-snapshot`
  - Task: Snapshot
  - Cron: `0 2 * * *` (2am every night)
  - Retain: 7 snapshots

This means I always have a week's worth of daily snapshots for every volume. If Nextcloud's database gets corrupted, I can roll back.

> `[SCREENSHOT]` — *Longhorn recurring job settings showing the daily-snapshot job configured*

---

## Monitoring Storage Health

Quick commands to check storage status:

```bash
# List all persistent volumes in the cluster
kubectl get pv

# List all claims by application
kubectl get pvc -A

# Describe a specific volume (events, capacity, binding)
kubectl describe pvc <name> -n <namespace>
```

In the Longhorn UI, the **Node** tab shows disk usage and health. A volume in **Healthy** state with **2 replicas** means the data is intact.

---

## Where I Am Now

At the end of Part 6 I have:

- ✅ Longhorn installed in the `storage` namespace
- ✅ Longhorn set as the default StorageClass (replacing `local-path`)
- ✅ Longhorn dashboard accessible at `https://longhorn.home.lab`
- ✅ Test PVC confirmed working — volumes bind within seconds
- ✅ `/media/movies` and `/media/shows` directories created for actual video files
- ✅ Recurring daily snapshots configured with 7-day retention

Next: Jellyfin — deploy the media server, connect it to my movie library, and access it from any device on the network.

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
