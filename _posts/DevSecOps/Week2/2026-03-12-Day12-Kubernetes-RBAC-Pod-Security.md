---
layout: post
title: "Week 2 — Day 12: Kubernetes RBAC & Pod Security"
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
description: A full walkthrough of Kubernetes RBAC, Pod Security Standards, Network Policies, and service account hardening — securing your cluster beyond just the application layer.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fmiro.medium.com%2Fv2%2Fresize%3Afit%3A1200%2F0*mHAuMdmOQwOa70Zf.jpg&f=1&nofb=1&ipt=f33090c83b448680b1963ab65855c8f88b346ecceb45c99b72b0a576524ae8d8
---

## Why Kubernetes Security Is Different

In Kubernetes, the attack surface isn't just your application — it's the entire cluster orchestration layer. Misconfigured RBAC, overly permissive pods, or unrestricted network traffic between pods can let an attacker move laterally across your entire infrastructure.

This day covers the three main layers of Kubernetes security: **identity (RBAC)**, **workload (Pod Security)**, and **network (Network Policies)**.

---

## Kubernetes RBAC

### Core Concepts

RBAC controls who can do what in a Kubernetes cluster.

| Object | Description |
|--------|-------------|
| `Role` | Permissions within a single namespace |
| `ClusterRole` | Permissions cluster-wide (or reusable across namespaces) |
| `RoleBinding` | Binds a Role to a user/group/service account within a namespace |
| `ClusterRoleBinding` | Binds a ClusterRole to a subject cluster-wide |

**Subjects (who):**
- `User` — human users (authenticated externally, e.g., via OIDC)
- `Group` — a set of users
- `ServiceAccount` — identity for a pod

---

### Roles and ClusterRoles

```yaml
# Role — namespace-scoped
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

```yaml
# ClusterRole — cluster-wide or reusable
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
```

**Verbs map to API operations:**

| Verb | HTTP | Operation |
|------|------|-----------|
| `get` | GET | Read a specific resource |
| `list` | GET | List resources |
| `watch` | GET | Stream changes |
| `create` | POST | Create |
| `update` | PUT | Replace |
| `patch` | PATCH | Partial update |
| `delete` | DELETE | Delete |

---

### RoleBindings

```yaml
# Bind pod-reader Role to a user in production namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# Bind to a ServiceAccount
subjects:
  - kind: ServiceAccount
    name: my-service-account
    namespace: production
```

> `[SCREENSHOT]` — *kubectl get rolebindings -n production -o wide showing the bindings with their roles and subjects listed*

---

### Service Account Hardening

Every pod gets a service account. By default it uses the `default` service account in its namespace — which often has more permissions than needed. Always create dedicated service accounts.

```yaml
# Dedicated service account for a deployment
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-service-account
  namespace: production
automountServiceAccountToken: false   # don't auto-mount if not needed
```

```yaml
# Reference it in the deployment
spec:
  serviceAccountName: api-service-account
  automountServiceAccountToken: false
```

> `[SCREENSHOT]` — *kubectl describe pod showing serviceAccount: api-service-account and the token mount either absent (if automount disabled) or present*

**Why disable token auto-mount?** By default, Kubernetes mounts a service account token at `/var/run/secrets/kubernetes.io/serviceaccount/token` in every pod. If your app doesn't call the Kubernetes API, this token is unnecessary attack surface — an attacker who gets into the pod can use it to query or modify cluster resources.

---

### Checking Permissions

```bash
# Can alice list pods in production?
kubectl auth can-i list pods --namespace production --as alice

# Can the default service account create deployments?
kubectl auth can-i create deployments \
  --namespace production \
  --as system:serviceaccount:production:default

# List what a service account can do
kubectl auth can-i --list \
  --namespace production \
  --as system:serviceaccount:production:api-service-account
```

> `[SCREENSHOT]` — *Terminal showing kubectl auth can-i --list output for a service account, listing all allowed verbs and resources*

---

### Common RBAC Mistakes

**Wildcard permissions:**
```yaml
# NEVER do this
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

**Cluster-admin binding for a namespace-scoped user:**
```yaml
# Avoid — gives full cluster control
roleRef:
  kind: ClusterRole
  name: cluster-admin
```

**Secrets access for pods that don't need it:**
```yaml
# Only add if your app actually reads Kubernetes secrets
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
```

---

## Pod Security Standards

Kubernetes Pod Security Standards (PSS) replace the deprecated PodSecurityPolicy. They define three policy levels:

| Level | Description |
|-------|-------------|
| `privileged` | No restrictions — for trusted system workloads |
| `baseline` | Prevents known privilege escalations |
| `restricted` | Hardened — enforces security best practices |

Applied via labels on namespaces.

---

### Enforcing Pod Security Standards

```bash
# Label a namespace to enforce the restricted profile
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

- `enforce` — rejects non-compliant pods
- `warn` — allows but prints a warning
- `audit` — logs to the audit log

> `[SCREENSHOT]` — *kubectl get namespace production --show-labels showing the pod-security labels applied*

### What "Restricted" Enforces

```yaml
# A pod compliant with "restricted" profile
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

Trying to deploy a pod with `privileged: true` or `runAsUser: 0` in a restricted namespace will be rejected.

> `[SCREENSHOT]` — *Terminal showing kubectl apply failing with "Error from server: pods ... is forbidden: violates PodSecurity restricted" when trying to deploy a privileged container in a restricted namespace*

---

## Network Policies

By default, all pods in a Kubernetes cluster can communicate with all other pods — across namespaces. Network Policies restrict this.

**Default deny all ingress for a namespace:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}      # applies to all pods in the namespace
  policyTypes:
    - Ingress
```

After applying this, no pod in `production` can receive traffic unless explicitly allowed.

---

### Allow Specific Traffic

```yaml
# Allow only the frontend pods to talk to the API pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

> `[SCREENSHOT]` — *kubectl get networkpolicy -n production showing the two policies (default-deny-ingress and allow-frontend-to-api) listed*

**Allow egress to DNS only (lock down outbound):**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - protocol: UDP
          port: 53    # Allow DNS resolution
```

---

## Lab — Restrict a Deployment with RBAC + Network Policy

**Objective:** Create a dedicated service account for a deployment and apply a network policy isolating it from other pods.

1. Create the namespace and service account:
```bash
kubectl create namespace lab
kubectl create serviceaccount api-sa -n lab
```

2. Create a Role for the service account:
```yaml
# api-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: lab
  name: api-role
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
```
```bash
kubectl apply -f api-role.yaml
kubectl create rolebinding api-binding \
  --role=api-role \
  --serviceaccount=lab:api-sa \
  -n lab
```

3. Deploy a pod using the service account:
```yaml
# api-deployment.yaml
spec:
  serviceAccountName: api-sa
  automountServiceAccountToken: false
  containers:
    - name: api
      image: nginx:alpine
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
```

> `[SCREENSHOT]` — *kubectl describe pod showing serviceAccountName: api-sa and the security context fields as configured*

4. Apply default deny network policy:
```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: lab
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
EOF
```

5. Verify isolation — from another pod, try to curl the API pod — it should time out:
```bash
kubectl run test --image=busybox -n lab --rm -it -- wget -T 3 api-pod-ip
```

> `[SCREENSHOT]` — *Terminal showing wget timing out when trying to reach the isolated pod — confirming the network policy is working*

---

## Key Takeaways

- Always use dedicated service accounts — never the `default` one
- Disable `automountServiceAccountToken` for pods that don't call the Kubernetes API
- Use `kubectl auth can-i --list` to audit what permissions a service account has
- Apply Pod Security Standards at the `restricted` level for production namespaces
- Network Policies default-deny is the only way to achieve true pod isolation — without it, all pods talk to all pods
- Restrict both Ingress AND Egress — egress restrictions prevent data exfiltration from a compromised pod

---

## References

<div class="references">
<ul>
  <li><a href="https://kubernetes.io/docs/reference/access-authn-authz/rbac/" target="_blank">Kubernetes RBAC Documentation</a></li>
  <li><a href="https://kubernetes.io/docs/concepts/security/pod-security-standards/" target="_blank">Pod Security Standards</a></li>
  <li><a href="https://kubernetes.io/docs/concepts/services-networking/network-policies/" target="_blank">Kubernetes Network Policies</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
