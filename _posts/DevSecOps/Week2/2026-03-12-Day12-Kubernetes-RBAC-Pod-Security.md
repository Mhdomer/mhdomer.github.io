---
layout: post
title: "Day 12: Kubernetes RBAC & Pod Security - Hardening Cluster Workloads"
date: 2026-03-12 10:00:00 +0800
categories:
  - DevSecOps
  - Week2
tags:
  - Kubernetes
  - RBAC
  - PodSecurity
  - ContainerSecurity
  - DevSecOps
author: muhammed
description: A practical walkthrough of Kubernetes RBAC, Pod Security Standards, Network Policies, and service account hardening - securing cluster workloads beyond the container boundary.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fmiro.medium.com%2Fv2%2Fresize%3Afit%3A1200%2F0*mHAuMdmOQwOa70Zf.jpg&f=1&nofb=1&ipt=f33090c83b448680b1963ab65855c8f88b346ecceb45c99b72b0a576524ae8d8
---

## Why Kubernetes Security Overwhelms Beginners

When you transition from running single Docker containers to managing a Kubernetes cluster, security suddenly becomes multi-layered.

You are no longer just securing an operating system or a Dockerfile. You are now securing an entire distributed orchestration platform:

- Who is allowed to talk to the Kubernetes API server?
- What happens if an attacker exploits a vulnerable web app running inside a pod? Can they query the internal cluster API to compromise other microservices?
- Can a compromised pod in the `staging` namespace freely send HTTP requests to the `production` database?

By default, Kubernetes is designed for developer convenience, not defense. Pods can communicate with all other pods across all namespaces, and every pod automatically receives an authentication token to communicate with the cluster API.

Today, we are demystifying Kubernetes security into three clear layers: **Identity (RBAC)**, **Workload Guardrails (Pod Security Standards)**, and **Network Isolation (Network Policies)**.

---

## Layer 1: Kubernetes RBAC (Role-Based Access Control)

Kubernetes Role-Based Access Control answers three simple questions for every request sent to the cluster API:

1. **Who? (The Subject):** A human user, an external group, or an automated `ServiceAccount` running inside a pod.
2. **What? (The Rules):** Which API operations (verbs: `get`, `list`, `create`, `delete`) on which resources (`pods`, `services`, `secrets`, `configmaps`).
3. **Where? (The Scope):** Inside a single namespace, or across the entire cluster?

```
+-------------------------------------------------------------+
|                     Kubernetes RBAC Model                   |
+-------------------------------------------------------------+
   [ Subject ]  ----->  [ RoleBinding ]  ----->  [ Role ]
 (User / ServiceAccount)    (The Glue)         (Verbs + Resources)
```

### Namespace-Scoped vs Cluster-Scoped

| Scope | Role Definition | Binding Mechanism | Use Case |
| :--- | :--- | :--- | :--- |
| **Namespace-Scoped** | `Role` | `RoleBinding` | Scoped to a single namespace (e.g. read pods in `staging`) |
| **Cluster-Scoped** | `ClusterRole` | `ClusterRoleBinding` | Applies cluster-wide (e.g. manage nodes, read secrets across all namespaces) |

---

## Writing Clean, Least-Privilege RBAC Manifests

### 1. The Role: Defining Permissions

Here is a clean `Role` that allows reading pods in the `production` namespace only:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader-role
rules:
  - apiGroups: [""] # Core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
```

### 2. The RoleBinding: Connecting the Subject to the Role

A `RoleBinding` acts as the glue connecting an identity to a `Role`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: production
  name: read-pods-binding
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader-role
  apiGroup: rbac.authorization.k8s.io
```

### Verifying Permissions with `kubectl auth can-i`

Never guess whether permissions work. Kubernetes provides a built-in authorization testing command:

```bash
# Test if user Alice can list pods in production
kubectl auth can-i list pods --namespace production --as alice
# Output: yes

# Test if Alice can delete deployments
kubectl auth can-i delete deployments --namespace production --as alice
# Output: no
```

---

## The ServiceAccount Silent Danger: Token Automounting

This is one of the most critical security traps in Kubernetes:

Whenever you create a Pod without specifying a service account, Kubernetes assigns it the `default` service account of that namespace.

By default, Kubernetes **automatically mounts a JWT bearer token** into every single container at:

```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

### Why This Is Dangerous

If your container runs an application with a Local File Inclusion (LFI) or Server-Side Request Forgery (SSRF) vulnerability:

1. The attacker reads `/var/run/secrets/kubernetes.io/serviceaccount/token`.
2. The attacker uses that bearer token to query `https://kubernetes.default.svc`.
3. If an administrator lazily bound permissions to the `default` service account, the attacker can now enumerate pods, dump secrets, or create privileged pods to compromise the entire cluster!

### The Hardening Fix

If your application does not explicitly need to talk to the Kubernetes API (like Prometheus or an operator does), **disable token auto-mounting**:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app-sa
  namespace: production
automountServiceAccountToken: false
```

And in your Deployment spec:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: web-app-sa
      automountServiceAccountToken: false
      containers:
        - name: web
          image: myapp:v1.0
```

With this single flag set to `false`, the token directory is not mounted, closing an entire attack vector.

---

## Layer 2: Pod Security Standards (PSS)

In older versions of Kubernetes, teams used `PodSecurityPolicy` (PSP). PSP was complex and deprecated. Modern Kubernetes uses **Pod Security Standards (PSS)**, built directly into the Kubernetes API.

PSS defines three baseline security profiles:

| Profile | Purpose | Enforcement Rules |
| :--- | :--- | :--- |
| **Privileged** | Unrestricted | For system daemons, CNI networking plugins, and storage drivers |
| **Baseline** | Default safety | Prevents known privilege escalations; blocks host namespaces and host ports |
| **Restricted** | Hardened production | Strictly enforces non-root execution, drops all capabilities, and requires read-only root filesystems |

### Enforcing Standards via Namespace Labels

You enforce Pod Security Standards simply by applying labels to a namespace:

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

If a developer attempts to deploy a pod running as root (`runAsUser: 0`) or requesting `privileged: true` into the `production` namespace, the Kubernetes admission controller rejects the deployment immediately!

---

## Layer 3: Kubernetes Network Policies

By default in Kubernetes, the internal network is completely flat. **Any pod in any namespace can communicate with any other pod.**

If an attacker compromises a frontend blog pod in the `staging` namespace, they can directly port-scan and query the production database pod in the `production` namespace over internal cluster IPs!

### The Solution: Default Deny All Ingress

The foundational rule of Kubernetes network security is applying a **Default Deny** policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {} # Matches ALL pods in this namespace
  policyTypes:
    - Ingress
```

Once applied, all incoming traffic to pods in `production` is blocked unless an explicit allow rule exists.

### Allowing Specific Microservice Traffic

Now, explicitly permit only the frontend microservice to communicate with the backend API:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend-api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend-web
      ports:
        - protocol: TCP
          port: 8080
```

Now, even if a threat actor compromises another pod in the same namespace, network packets destined for `backend-api` on port 8080 are dropped at the CNI layer.

---

## Hands-on Lab: Hardening a Production Namespace

Let's test this end-to-end:

### Step 1: Create Namespace with Restricted PSS

```bash
kubectl create namespace secure-lab

kubectl label namespace secure-lab \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest
```

### Step 2: Test Non-Compliant Pod Rejection

Attempt to launch an insecure pod running as root:

```bash
kubectl run insecure-pod --image=nginx -n secure-lab
```

The Kubernetes API server immediately rejects the request:

```
Error from server (Forbidden): pods "insecure-pod" is forbidden: violates PodSecurity "restricted:latest": 
allowPrivilegeEscalation != false, unrestricted capabilities, runAsNonRoot != true, runAsUser=0
```

### Step 3: Deploy a Hardened Compliant Pod

Deploy a pod that complies with the restricted profile:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
  namespace: secure-lab
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: web
      image: nginxinc/nginx-unprivileged:alpine
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

The pod deploys cleanly and passes all admission checks!

---

## Junior Pitfalls to Avoid

1. **The `cluster-admin` Laziness Trap:** Binding `cluster-admin` to a developer or service account to bypass an RBAC error is the most common cause of cluster compromise. Scope permissions to namespaces using `Role` and `RoleBinding`.
2. **Leaving Token Automount Enabled:** Always set `automountServiceAccountToken: false` on pods that do not interact with the Kubernetes API.
3. **Assuming Network Policies Work Without a Supporting CNI:** Kubernetes Network Policies require a network plugin that supports them (e.g. Calico, Cilium, or AWS VPC CNI with network policy support enabled). In standard default Minikube or basic cloud clusters without a policy controller, Network Policy manifests apply without errors but traffic is NOT blocked!
4. **Granting Wildcard Verbs:** Never write `verbs: ["*"]` or `resources: ["*"]` in production RBAC rules. Always specify explicit actions.

---

## Key Takeaways

- Kubernetes RBAC controls who can interact with the cluster API server; always scope to namespaces using `Role` rather than `ClusterRole`.
- Disable `automountServiceAccountToken` on workloads to prevent token theft during container escapes or SSRF attacks.
- Enforce Pod Security Standards at the `restricted` level on production namespaces.
- Apply Default Deny Network Policies to isolate pods and stop lateral movement across namespaces.
- Use `kubectl auth can-i` to audit and verify effective permissions.

---

## References

<div class="references">
<ul>
  <li><a href="https://kubernetes.io/docs/reference/access-authn-authz/rbac/" target="_blank">Kubernetes RBAC Reference Documentation</a></li>
  <li><a href="https://kubernetes.io/docs/concepts/security/pod-security-standards/" target="_blank">Kubernetes Pod Security Standards (PSS)</a></li>
  <li><a href="https://kubernetes.io/docs/concepts/services-networking/network-policies/" target="_blank">Kubernetes Network Policies Guide</a></li>
  <li><a href="https://github.com/cyberark/kubernetes-rbac-audit" target="_blank">Kubernetes RBAC Audit Toolkit</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
