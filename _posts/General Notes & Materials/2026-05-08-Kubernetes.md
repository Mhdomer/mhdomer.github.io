---
layout: post
title: Kubernetes Full Walkthrough
date: 2026-05-08T11:00:00
categories:
  - General Notes & Materials
tags:
  - kubernetes
  - k8s
  - containers
  - devops
  - orchestration
author: muhammed
description: Full Kubernetes walkthrough — architecture, pods, deployments, services, ingress, config, storage, and kubectl from the ground up
toc: true
pin: false
math: false
mermaid: false
image: /assets/notes-img/kubernetes.png
Link: "[[]]"
Link1:
img:
---

# Kubernetes Full Walkthrough

---

## What is Kubernetes?

Kubernetes (K8s) is a container orchestration platform. Where Docker runs a single container, Kubernetes manages **thousands of containers across many machines** — scheduling, scaling, healing, and networking them automatically.

**Core job of Kubernetes:**
- Run your containers reliably across a cluster
- Restart them if they crash
- Scale up/down based on load
- Route traffic to healthy instances
- Roll out updates with zero downtime

---

## Architecture

### The Cluster

A Kubernetes cluster has two types of machines:

```
┌─────────────────────────────────────────────────┐
│                  CONTROL PLANE                   │
│  kube-apiserver  etcd  scheduler  controller-mgr │
└─────────────────────────────────────────────────┘
          │              │              │
   ┌──────┴──┐    ┌──────┴──┐   ┌──────┴──┐
   │  Node 1  │    │  Node 2  │   │  Node 3  │
   │  kubelet │    │  kubelet │   │  kubelet │
   │  kube-  │    │  kube-  │   │  kube-  │
   │  proxy   │    │  proxy   │   │  proxy   │
   └─────────┘    └─────────┘   └─────────┘
```

**Control Plane components:**

| Component | Role |
|---|---|
| `kube-apiserver` | The front door — all kubectl commands hit this REST API |
| `etcd` | Distributed key-value store — the cluster's source of truth |
| `kube-scheduler` | Decides which node a new pod runs on |
| `kube-controller-manager` | Runs control loops (ReplicaSet, Node, Endpoint controllers) |
| `cloud-controller-manager` | Integrates with cloud provider (AWS, GCP, Azure) |

**Worker Node components:**

| Component | Role |
|---|---|
| `kubelet` | Agent on each node — ensures containers are running |
| `kube-proxy` | Maintains network rules for Service routing |
| `container runtime` | Actually runs containers (containerd, CRI-O) |

---

## Installing kubectl and a Local Cluster

### kubectl — the CLI

```bash
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

### Local cluster options

| Tool | Best for |
|---|---|
| **minikube** | Learning, single-node |
| **kind** (K8s in Docker) | CI pipelines, testing |
| **k3s** | Lightweight, Raspberry Pi, edge |
| **Docker Desktop** | Mac/Windows dev |

```bash
# minikube
minikube start
minikube start --driver=docker --cpus=4 --memory=4g

# kind
kind create cluster
kind create cluster --name mylab

# k3s (single command install)
curl -sfL https://get.k3s.io | sh -
```

### kubectl context and config

```bash
kubectl config get-contexts                  # list all clusters/contexts
kubectl config current-context              # which cluster you're talking to
kubectl config use-context minikube         # switch cluster
kubectl config set-context --current --namespace=dev  # set default namespace
cat ~/.kube/config                          # raw kubeconfig file
```

---

## Core Objects (Resources)

Everything in Kubernetes is a **resource** — you describe the desired state in YAML and kubectl applies it.

### The basic pattern

```bash
kubectl apply -f resource.yaml     # create or update
kubectl delete -f resource.yaml    # delete
kubectl get pods                   # list
kubectl describe pod mypod         # detailed info + events
kubectl edit pod mypod             # live edit in $EDITOR
```

---

## Pods

A **Pod** is the smallest deployable unit — one or more containers that share a network and storage.

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  containers:
    - name: app
      image: nginx:alpine
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
      env:
        - name: ENV
          value: "production"
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
      livenessProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 15
        periodSeconds: 20
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl get pods -o wide           # show node and IP
kubectl logs myapp                 # container stdout
kubectl logs myapp -f              # follow
kubectl logs myapp -c sidecar      # specific container in multi-container pod
kubectl exec -it myapp -- bash     # shell inside pod
kubectl delete pod myapp
```

**You almost never create naked Pods in production** — use Deployments instead (they recreate pods if they die).

---

## Namespaces

Namespaces are virtual clusters within a cluster — used for isolation between teams/environments.

```bash
kubectl get namespaces
kubectl create namespace staging
kubectl delete namespace staging

# Run commands in a specific namespace
kubectl get pods -n staging
kubectl get pods --all-namespaces      # or -A
kubectl apply -f app.yaml -n staging
```

Default namespaces: `default`, `kube-system`, `kube-public`, `kube-node-lease`

---

## Deployments

A **Deployment** manages a set of identical Pods (replicas), handles rolling updates, and restarts crashed pods.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods                             # see the 3 replica pods
kubectl rollout status deployment/myapp     # watch rollout progress

# Scale
kubectl scale deployment myapp --replicas=5

# Update image (triggers rolling update)
kubectl set image deployment/myapp app=nginx:1.26

# Rollout history and rollback
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=2

# Delete
kubectl delete deployment myapp
```

---

## Services

A **Service** gives a stable DNS name and IP to a set of Pods (selected by label). Pods come and go, but the Service stays.

### Service types

| Type | Use case |
|---|---|
| `ClusterIP` | Internal only — default, reachable within cluster |
| `NodePort` | Exposes on a port on every node (30000–32767) |
| `LoadBalancer` | Cloud load balancer (AWS ELB, GCP LB) |
| `ExternalName` | DNS alias to an external hostname |

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp          # matches pods with this label
  ports:
    - protocol: TCP
      port: 80          # service port (what clients call)
      targetPort: 80    # container port (where traffic goes)
  type: ClusterIP
```

```bash
kubectl apply -f service.yaml
kubectl get services
kubectl describe service myapp-svc

# Test from inside the cluster
kubectl run test --rm -it --image=busybox -- wget -qO- http://myapp-svc

# NodePort — access via any node IP
kubectl expose deployment myapp --type=NodePort --port=80
minikube service myapp --url        # get the URL in minikube
```

---

## Ingress

An **Ingress** routes HTTP/HTTPS traffic from outside the cluster to Services based on hostname or path. Requires an **Ingress Controller** (nginx-ingress, Traefik, etc.).

```bash
# Install nginx ingress controller (minikube)
minikube addons enable ingress

# Install via Helm (production)
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-svc
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 8000
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress myapp-ingress
```

---

## ConfigMaps and Secrets

### ConfigMap — non-sensitive configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
```

```bash
kubectl create configmap app-config --from-literal=APP_ENV=production
kubectl create configmap app-config --from-file=config.yaml
kubectl get configmap app-config -o yaml
```

**Use in a Pod:**

```yaml
# As environment variables
envFrom:
  - configMapRef:
      name: app-config

# As a mounted file
volumes:
  - name: config-vol
    configMap:
      name: app-config
containers:
  - volumeMounts:
      - name: config-vol
        mountPath: /etc/app
```

### Secret — sensitive data (base64 encoded)

```bash
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=supersecret

kubectl get secret db-creds -o yaml
echo "c3VwZXJzZWNyZXQ=" | base64 -d    # decode a secret value
```

```yaml
# Use in a Pod as env vars
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-creds
        key: password

# Or mount as files
volumes:
  - name: secrets-vol
    secret:
      secretName: db-creds
```

**Secrets are base64, not encrypted by default** — use Sealed Secrets, Vault, or cloud KMS for real encryption at rest.

---

## Persistent Storage

### PersistentVolume (PV) and PersistentVolumeClaim (PVC)

```yaml
# PVC — claim storage (you rarely define PVs manually; cloud providers do it)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-storage
spec:
  accessModes:
    - ReadWriteOnce        # one node can read/write
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

Access modes:
- `ReadWriteOnce` (RWO) — one node, read/write
- `ReadOnlyMany` (ROX) — many nodes, read only
- `ReadWriteMany` (RWX) — many nodes, read/write (requires NFS or cloud FS)

```yaml
# Use in a Pod
volumes:
  - name: db-vol
    persistentVolumeClaim:
      claimName: db-storage
containers:
  - volumeMounts:
      - name: db-vol
        mountPath: /var/lib/postgresql/data
```

```bash
kubectl get pvc
kubectl get pv
kubectl describe pvc db-storage
```

---

## StatefulSets

StatefulSets are for stateful apps (databases, queues) that need:
- Stable, predictable pod names (`db-0`, `db-1`, `db-2`)
- Stable persistent storage per pod
- Ordered startup and shutdown

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-creds
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

---

## Resource Management

### Requests vs Limits

```yaml
resources:
  requests:
    cpu: "100m"       # minimum guaranteed (1000m = 1 CPU core)
    memory: "128Mi"   # minimum guaranteed
  limits:
    cpu: "500m"       # hard cap — throttled if exceeded
    memory: "256Mi"   # hard cap — OOMKilled if exceeded
```

- **Requests** — used by the scheduler to decide which node to place the pod on
- **Limits** — enforced at runtime; memory limit breach = pod killed

### LimitRange and ResourceQuota

```yaml
# LimitRange — default limits for a namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
```

```yaml
# ResourceQuota — total cap for a namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
```

---

## Horizontal Pod Autoscaler (HPA)

Automatically scales the number of pod replicas based on CPU/memory.

```bash
kubectl autoscale deployment myapp --cpu-percent=70 --min=2 --max=10
kubectl get hpa
kubectl describe hpa myapp
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

Requires the **metrics-server** to be running.

---

## Jobs and CronJobs

### Job — run a task to completion

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp:latest
          command: ["python", "manage.py", "migrate"]
      restartPolicy: OnFailure
  backoffLimit: 3
```

### CronJob — scheduled job

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"        # cron syntax
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: myapp:latest
              command: ["/scripts/backup.sh"]
          restartPolicy: OnFailure
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

```bash
kubectl get jobs
kubectl get cronjobs
kubectl logs job/db-migrate
```

---

## RBAC — Role-Based Access Control

### ServiceAccount

Every pod runs as a ServiceAccount (default: `default`). Create dedicated ones:

```bash
kubectl create serviceaccount myapp-sa
```

### Role and RoleBinding (namespace-scoped)

```yaml
# Role — what actions are allowed
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
```

```yaml
# RoleBinding — who gets the Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Use `ClusterRole` + `ClusterRoleBinding` for cluster-wide permissions.

---

## Helm — Kubernetes Package Manager

Helm is to Kubernetes what apt is to Ubuntu — it installs pre-packaged applications called **charts**.

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search and install
helm search repo bitnami/postgresql
helm install my-postgres bitnami/postgresql --set auth.postgresPassword=secret

# List installed releases
helm list
helm list -A               # all namespaces

# Upgrade and rollback
helm upgrade my-postgres bitnami/postgresql --set image.tag=16
helm rollback my-postgres 1

# Uninstall
helm uninstall my-postgres
```

---

## kubectl Cheat Sheet

```bash
# Get resources
kubectl get pods,svc,deploy                # multiple types at once
kubectl get pods -o wide                   # show node and IP
kubectl get pods -o yaml                   # full YAML output
kubectl get pods -l app=myapp              # filter by label
kubectl get pods --field-selector=status.phase=Running
kubectl get all -n staging                 # everything in a namespace

# Describe (events are at the bottom — check these first when debugging)
kubectl describe pod mypod
kubectl describe node worker-1

# Logs
kubectl logs mypod
kubectl logs -f mypod                      # follow
kubectl logs mypod --previous             # logs from crashed previous container
kubectl logs -l app=myapp --all-containers # all pods matching label

# Exec
kubectl exec -it mypod -- bash
kubectl exec -it mypod -c sidecar -- sh   # specific container

# Port forward (for local testing)
kubectl port-forward pod/mypod 8080:80
kubectl port-forward svc/myapp-svc 8080:80
kubectl port-forward deployment/myapp 8080:80

# Copy files
kubectl cp mypod:/etc/config.yaml ./config.yaml
kubectl cp ./local.txt mypod:/tmp/local.txt

# Apply, delete, diff
kubectl apply -f .                        # apply all YAML in current dir
kubectl apply -f https://example.com/manifest.yaml
kubectl delete -f deployment.yaml
kubectl diff -f deployment.yaml           # what would change

# Dry run
kubectl apply -f deployment.yaml --dry-run=client
kubectl apply -f deployment.yaml --dry-run=server

# Generate YAML skeletons
kubectl create deployment myapp --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl create service clusterip myapp --tcp=80:80 --dry-run=client -o yaml

# Labels and annotations
kubectl label pod mypod env=production
kubectl annotate pod mypod team=backend
kubectl get pods -l env=production

# Drain and cordon nodes (maintenance)
kubectl cordon node-1                     # stop scheduling new pods
kubectl drain node-1 --ignore-daemonsets # evict pods
kubectl uncordon node-1                  # re-enable scheduling

# Events — great for debugging
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n staging
```

---

## Debugging Patterns

**Pod won't start:**

```bash
kubectl describe pod mypod        # look at Events section at the bottom
kubectl logs mypod --previous     # logs before the crash
```

Common statuses:
- `Pending` — not scheduled yet (check node resources, taints, affinities)
- `CrashLoopBackOff` — container keeps crashing (check logs)
- `ImagePullBackOff` — can't pull image (check image name, registry credentials)
- `OOMKilled` — exceeded memory limit (increase limits)
- `Evicted` — node was under pressure (check node disk/memory)

**Service not reachable:**

```bash
kubectl get endpoints myapp-svc   # should show pod IPs — if empty, labels don't match
kubectl exec -it debug -- wget -qO- http://myapp-svc    # test from inside cluster
```

**Check node health:**

```bash
kubectl get nodes
kubectl describe node worker-1    # check Conditions and Events
kubectl top nodes                 # CPU/memory usage (needs metrics-server)
kubectl top pods
```

---

## Quick Reference

| Task | Command |
|---|---|
| Apply a manifest | `kubectl apply -f file.yaml` |
| Get pods | `kubectl get pods -o wide` |
| Shell into pod | `kubectl exec -it pod -- bash` |
| View logs | `kubectl logs -f pod` |
| Port forward | `kubectl port-forward svc/name 8080:80` |
| Scale deployment | `kubectl scale deploy/name --replicas=5` |
| Update image | `kubectl set image deploy/name app=img:tag` |
| Rollback | `kubectl rollout undo deploy/name` |
| Watch rollout | `kubectl rollout status deploy/name` |
| Get all resources | `kubectl get all -n namespace` |
| Debug events | `kubectl get events --sort-by='.lastTimestamp'` |
| Switch context | `kubectl config use-context name` |
| Switch namespace | `kubectl config set-context --current --namespace=ns` |
| Install Helm chart | `helm install name repo/chart` |
| List Helm releases | `helm list -A` |
