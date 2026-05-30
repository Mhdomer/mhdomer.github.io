---
layout: post
title: "Homelab Part 7 — Jellyfin Self-Hosted Media Server"
date: 2026-05-24 16:00:00 +0800
categories:
  - Homelab
tags:
  - Homelab
  - Kubernetes
  - Jellyfin
  - Media
  - Selfhosted
author: muhammed
description: Deploying Jellyfin on Kubernetes — mounting my local movie library, setting up the media server, configuring HTTPS access at jellyfin.home.lab, and streaming to every device on my network.
toc: true
pin: false
math: false
mermaid: false
---

## What Jellyfin Is

Jellyfin is a free, open-source media server. I point it at a folder of movies and TV shows, it scans the files, downloads metadata and cover art from the internet, and presents a Netflix-like interface I can access from any device on my home network — phone, iPad, laptop, TV.

No subscriptions. No sending data to a third party. My movies, my server, my data.

The alternative most people know is Plex. Plex requires a cloud account and sometimes a Plex Pass subscription for certain features. Jellyfin is completely self-contained — it runs offline if it needs to.

---

## Storage Architecture for Jellyfin

Jellyfin needs two kinds of storage:

1. **Config volume** (Longhorn) — user accounts, library database, watch history, transcoding cache. Small (5GB), but important. Needs to survive pod restarts.

2. **Media directory** (hostPath) — the actual `.mkv`, `.mp4`, `.avi` files. These live in `/media/movies` on the VM's filesystem. I don't put them in Longhorn because video files are huge and Longhorn is better suited for structured data. A `hostPath` mount gives the pod direct access to the VM's disk.

---

## Adding Movies to the VM

Before deploying Jellyfin, I copy my movies to the media directory on the K3s VM.

From Windows, I use SCP:

```powershell
# Copy a single movie
scp "D:\Movies\The.Matrix.1999.mkv" homelab@192.168.1.51:/media/movies/

# Copy an entire folder
scp -r "D:\Movies\" homelab@192.168.1.51:/media/movies/
```

Or I use a file manager that supports SFTP (like WinSCP or Filezilla) and drag-and-drop.

Once they're on the VM:

```bash
# Verify on the K3s VM
ls -lh /media/movies/
# Should show your movie files
```

> `[SCREENSHOT]` — *Terminal showing movie files listed in /media/movies with their sizes*

---

## Deploying Jellyfin

I create a single manifest file with all the Kubernetes resources Jellyfin needs:

```yaml
# jellyfin.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: media
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jellyfin-config
  namespace: media
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jellyfin
  namespace: media
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jellyfin
  template:
    metadata:
      labels:
        app: jellyfin
    spec:
      containers:
        - name: jellyfin
          image: jellyfin/jellyfin:latest
          ports:
            - containerPort: 8096
          env:
            - name: JELLYFIN_DATA_DIR
              value: /config
            - name: JELLYFIN_CACHE_DIR
              value: /config/cache
          volumeMounts:
            - name: config
              mountPath: /config
            - name: movies
              mountPath: /media/movies
              readOnly: true
            - name: shows
              mountPath: /media/shows
              readOnly: true
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: jellyfin-config
        - name: movies
          hostPath:
            path: /media/movies
            type: Directory
        - name: shows
          hostPath:
            path: /media/shows
            type: Directory
---
apiVersion: v1
kind: Service
metadata:
  name: jellyfin
  namespace: media
spec:
  selector:
    app: jellyfin
  ports:
    - port: 8096
      targetPort: 8096
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jellyfin
  namespace: media
  annotations:
    cert-manager.io/cluster-issuer: homelab-ca-issuer
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
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

Two things to note about the Ingress annotations:
- `proxy-body-size: "0"` removes the upload size limit — needed if you ever upload media through the UI
- `proxy-read-timeout: "600"` and `proxy-send-timeout: "600"` give 10-minute timeouts — important for video streaming

```bash
kubectl apply -f jellyfin.yaml
```

Watch the pod start:

```bash
kubectl get pods -n media -w
# jellyfin-xxx    0/1   ContainerCreating   ...
# jellyfin-xxx    1/1   Running             ...
```

> `[SCREENSHOT]` — *`kubectl get pods -n media` showing jellyfin pod Running*

---

## Verifying the PVC Bound

```bash
kubectl get pvc -n media
# NAME              STATUS   VOLUME             CAPACITY   ACCESS MODES
# jellyfin-config   Bound    pvc-xxx-xxx-xxx    5Gi        RWO
```

The STATUS must be `Bound` before Jellyfin can start. If it stays `Pending`, check Longhorn is healthy:

```bash
kubectl get pods -n storage
kubectl describe pvc jellyfin-config -n media
```

> `[SCREENSHOT]` — *`kubectl get pvc -n media` showing jellyfin-config as Bound*

---

## First-Time Jellyfin Setup

Open `https://jellyfin.home.lab` in a browser.

The setup wizard runs:

### 1. Create Admin Account

Set a username and strong password. This is your admin account for managing the server.

> `[SCREENSHOT]` — *Jellyfin initial setup — create admin user screen*

### 2. Set Up Media Libraries

Click **Add Media Library**:

**For Movies:**
- Content type: Movies
- Display name: Movies
- Folder: click **+** → type `/media/movies`
- Language: your preference
- Country: your preference

**For TV Shows:**
- Content type: TV Shows
- Display name: Shows
- Folder: `/media/shows`

> `[SCREENSHOT]` — *Jellyfin Add Library screen with /media/movies folder selected*

### 3. Metadata Language

Set your preferred language for titles, descriptions, and metadata.

### 4. Finish Setup

Click through the remaining screens (allow remote access: yes, streaming port: keep default). Log in with your admin account.

---

## The Jellyfin Dashboard

After setup, Jellyfin scans the library. For a few hundred movies, this takes 2–5 minutes. It downloads:
- Movie posters and backdrops
- Cast and crew information
- Synopsis, ratings, genres
- Trailers (if enabled)

Once done, the dashboard shows your library with full artwork — exactly like Netflix or Apple TV.

> `[SCREENSHOT]` — *Jellyfin dashboard showing movie library with poster art*

---

## Streaming to Devices

Jellyfin has clients for everything:

| Device | How to Access |
|--------|--------------|
| Browser | `https://jellyfin.home.lab` |
| iPhone/iPad | Jellyfin app (App Store) → Add Server → `https://jellyfin.home.lab` |
| Android | Jellyfin app (Play Store) |
| Apple TV | Infuse app → Add Jellyfin server |
| Smart TV | Jellyfin app (some Samsung/LG TVs) |
| Windows | Jellyfin Media Player app |

For iPhone/iPad — since I'm using a self-signed certificate (the homelab CA), I need the CA cert installed first. I did that in Part 5. Without it, the app will refuse the HTTPS connection.

> `[SCREENSHOT]` — *Jellyfin app on iPhone showing the movie library and a movie detail page*

---

## Transcoding

Jellyfin can transcode video on-the-fly — converting from a format your device can't play (like some `.mkv` H.265 files) to something it can. This is CPU-intensive.

With 8 cores allocated to the K3s VM, software transcoding works fine. If you have issues with high CPU during playback, check the Jellyfin admin panel:

**Dashboard → Playback → Transcoding:**
- Check which codec is being used
- Consider using Direct Play/Direct Stream when possible (no transcoding needed)

For most modern devices (iPhone, iPad, Chromebook), H.264/AAC in an MP4 container plays natively without transcoding. H.265 files will transcode on iOS unless you use an app like Infuse, which has its own hardware decoder.

---

## Useful Jellyfin Admin Operations

```bash
# Check Jellyfin logs for errors
kubectl logs -n media deployment/jellyfin

# Restart Jellyfin (preserves config — it's in Longhorn)
kubectl rollout restart deployment/jellyfin -n media

# Scale down (stop Jellyfin temporarily)
kubectl scale deployment jellyfin -n media --replicas=0

# Scale back up
kubectl scale deployment jellyfin -n media --replicas=1
```

In the Jellyfin web UI, **Dashboard → Library** lets you:
- Scan for new media (after adding movies)
- Refresh metadata for a specific movie
- View recent activity and active streams

---

## Where I Am Now

At the end of Part 7 I have:

- ✅ Jellyfin deployed in the `media` namespace
- ✅ Config stored in a Longhorn PVC (survives pod restarts)
- ✅ Movies and shows mounted directly from the VM's `/media/` directory
- ✅ HTTPS access at `https://jellyfin.home.lab`
- ✅ Media library scanning and streaming working
- ✅ iPhone/iPad app connected and streaming

Next: Nextcloud — my personal cloud storage, so I can sync and access files from anywhere on the network.

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
