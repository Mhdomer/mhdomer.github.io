---
layout: post
title: "Week 3 — Day 17: IaC Scanning with Checkov & tfsec"
date: 2026-06-05 10:00:00 +0800
categories:
  - DevSecOps
  - Week3
tags:
  - Checkov
  - tfsec
  - Terraform
  - IaC
  - DevSecOps
  - CloudSecurity
author: muhammed
description: A full walkthrough of Checkov and tfsec — scanning Terraform, CloudFormation, and Kubernetes manifests for security misconfigurations before they reach production.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fgo.snyk.io%2Frs%2F677-THP-415%2Fimages%2Flogo-vertical-black.png&f=1&nofb=1&ipt=edf0ec8b56bce70e32ec1ec1f7985cc6b702342ce79f8302afd5dc886846e283
---

## Why IaC Security Scanning?

Infrastructure as Code defines your cloud environment — security groups, IAM policies, S3 bucket settings, encryption configs. A misconfiguration in Terraform is a misconfiguration in production.

IaC scanning catches these **before `terraform apply`** — in the PR, in the pipeline, before any real infrastructure changes.

**Common IaC misconfigurations:**
- Security groups allowing `0.0.0.0/0` on port 22 or 3389
- S3 buckets with public access enabled
- RDS instances without encryption
- EC2 instances with public IPs
- IAM roles with wildcard permissions
- Unencrypted EBS volumes
- CloudTrail disabled
- Logging not enabled on load balancers

---

## Checkov

Checkov is a static analysis tool for IaC by Bridgecrew (Palo Alto). It supports Terraform, CloudFormation, Kubernetes, Helm, Dockerfile, GitHub Actions, and more.

### Installation

```bash
pip install checkov

# Verify
checkov --version
```

---

### Scanning Terraform

```bash
# Scan a Terraform directory
checkov -d ./terraform/

# Scan a single file
checkov -f main.tf

# Show only failed checks
checkov -d ./terraform/ --compact

# Output as JSON
checkov -d ./terraform/ --output json > results.json

# Output as JUnit XML (for CI test reports)
checkov -d ./terraform/ --output junitxml > results.xml
```

> `[SCREENSHOT]` — *Terminal showing checkov -d ./terraform/ output — a table of passed/failed checks with check IDs (CKV_AWS_*), resource names, and file paths. Summary at the bottom showing X passed, Y failed*

---

### Understanding Checkov Output

```
Check: CKV_AWS_18: "Ensure the S3 bucket has access logging enabled"
  FAILED for resource: aws_s3_bucket.data-bucket
  File: /terraform/s3.tf:12-28
  Guide: https://docs.bridgecrew.io/docs/s3_14

Check: CKV_AWS_21: "Ensure the S3 bucket has versioning enabled"
  PASSED for resource: aws_s3_bucket.data-bucket

Check: CKV_AWS_145: "Ensure that S3 buckets are encrypted with KMS by default"
  FAILED for resource: aws_s3_bucket.data-bucket
  File: /terraform/s3.tf:12-28
```

Each check has:
- **Check ID** — e.g., `CKV_AWS_18` — uniquely identifies the rule
- **Status** — PASSED / FAILED / SKIPPED
- **Resource** — the Terraform resource that was checked
- **File + line** — exactly where in your code

> `[SCREENSHOT]` — *Checkov output showing a mix of PASSED (green) and FAILED (red) checks on Terraform resources, with the check IDs and resource names visible*

---

### Key Checkov Checks for AWS

| Check ID | What it verifies |
|----------|-----------------|
| `CKV_AWS_18` | S3 access logging enabled |
| `CKV_AWS_19` | S3 server-side encryption enabled |
| `CKV_AWS_20` | S3 bucket not publicly readable |
| `CKV_AWS_21` | S3 versioning enabled |
| `CKV_AWS_23` | ALB access logs enabled |
| `CKV_AWS_25` | Security group no unrestricted SSH |
| `CKV_AWS_57` | S3 block public ACLs |
| `CKV_AWS_66` | CloudTrail log file validation |
| `CKV_AWS_76` | CloudTrail S3 bucket has access logging |
| `CKV_AWS_86` | ECS task definition logging enabled |
| `CKV_AWS_97` | S3 bucket encryption with KMS |
| `CKV_AWS_111` | IAM policy no wildcard resources |
| `CKV_AWS_130` | VPC subnets no public IP on launch |

---

### Scanning Your MindCraft Terraform

```bash
# Navigate to the MindCraft Terraform directory
checkov -d _posts/MindCraft\ Cloud\ Deployment/ --filter-regex ".*\.tf"

# Or scan any specific Terraform directory
checkov -d ./terraform/ --check CKV_AWS_20,CKV_AWS_19,CKV_AWS_25
```

> `[SCREENSHOT]` — *Checkov running on the MindCraft Terraform code showing the specific failures — e.g., S3 bucket without encryption, security group open to 0.0.0.0/0, with the exact Terraform file and line numbers*

---

### Skipping False Positives

Add a skip comment directly in Terraform:

```hcl
resource "aws_s3_bucket" "public-assets" {
  bucket = "my-public-website-assets"

  # checkov:skip=CKV_AWS_20:Public assets bucket intentionally public for website
  # checkov:skip=CKV_AWS_57:Public ACLs required for static website hosting
}
```

Or skip via CLI:

```bash
checkov -d . --skip-check CKV_AWS_20,CKV_AWS_57
```

---

### Scanning Kubernetes Manifests

```bash
checkov -d ./k8s/ --framework kubernetes
```

Key Kubernetes checks:

| Check ID | What it verifies |
|----------|-----------------|
| `CKV_K8S_6` | Do not admit root containers |
| `CKV_K8S_8` | Liveness probe configured |
| `CKV_K8S_14` | Image tag is not latest |
| `CKV_K8S_20` | Containers do not run with allowPrivilegeEscalation |
| `CKV_K8S_28` | Do not admit containers with added capabilities |
| `CKV_K8S_30` | Do not admit containers with NET_RAW capability |
| `CKV_K8S_37` | Minimize the admission of containers with added capability |

> `[SCREENSHOT]` — *Checkov scanning a Kubernetes deployment manifest and showing failures like "Do not admit root containers" and "Image tag is not latest" with the manifest file and line numbers*

---

### Scanning Dockerfiles

```bash
checkov -f Dockerfile --framework dockerfile
```

> `[SCREENSHOT]` — *Checkov scanning a Dockerfile and flagging issues like "Ensure that a user for the container has been created" (non-root user) and "Ensure that COPY is used instead of ADD"*

---

## tfsec

tfsec is a Terraform-specific security scanner by Aqua Security. It's faster than Checkov for pure Terraform use cases and integrates natively with the Terraform ecosystem.

### Installation

```bash
# Windows via Scoop
scoop install tfsec

# Or download from GitHub releases
# https://github.com/aquasecurity/tfsec/releases

tfsec --version
```

---

### Scanning with tfsec

```bash
# Scan current directory
tfsec .

# Scan specific directory
tfsec ./terraform/

# Only show specific severities
tfsec . --minimum-severity HIGH

# Output as JSON
tfsec . --format json --out results.json

# Output as SARIF
tfsec . --format sarif --out results.sarif
```

> `[SCREENSHOT]` — *Terminal showing tfsec . output — colored blocks for each finding with the check ID, description, severity (CRITICAL/HIGH/MEDIUM/LOW), file path, line range, and a code snippet of the offending Terraform resource*

---

### tfsec Output Format

```
Result #1 HIGH Security group rule allows ingress from public internet.
───────────────────────────────────────────────
  ID        aws-ec2-no-public-ingress-sgr
  Impact    Your port exposed to the internet
  Resolution Remove the public CIDR block from the rule
  More Info https://aquasecurity.github.io/tfsec/...
───────────────────────────────────────────────
  /terraform/security-groups.tf:15-22
  via /terraform/security-groups.tf:12-25
───────────────────────────────────────────────
   12    resource "aws_security_group_rule" "ssh" {
   13      type        = "ingress"
   14      from_port   = 22
   15 [    cidr_blocks = ["0.0.0.0/0"]
   16      to_port     = 22
   17      protocol    = "tcp"
   18    }
```

> `[SCREENSHOT]` — *tfsec output showing a security group with 0.0.0.0/0 on port 22, with the code snippet highlighted showing exactly which line is the problem*

---

### Ignoring Findings in tfsec

```hcl
resource "aws_security_group_rule" "bastion-ssh" {
  type        = "ingress"
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]   # tfsec:ignore:aws-ec2-no-public-ingress-sgr - bastion host, intentional
}
```

---

## Checkov vs tfsec

| | Checkov | tfsec |
|--|---------|-------|
| Supports | Terraform, CF, K8s, Dockerfile, GH Actions | Terraform only |
| Speed | Moderate | Fast |
| Custom rules | Python or YAML | YAML |
| Output formats | JSON, JUnit, SARIF, CLI | JSON, SARIF, CLI |
| Best for | Multi-framework pipelines | Pure Terraform-heavy teams |

**Use both in the pipeline** — they have overlapping but not identical rule sets and catch different things.

---

## GitHub Actions Integration

```yaml
# .github/workflows/iac-scan.yml
name: IaC Security Scan

on:
  push:
    paths:
      - 'terraform/**'
      - '*.tf'
  pull_request:
    paths:
      - 'terraform/**'

jobs:
  checkov:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: terraform/
          framework: terraform
          output_format: sarif
          output_file_path: checkov.sarif
          soft_fail: false        # fail the pipeline on findings

      - name: Upload Checkov SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: checkov.sarif

  tfsec:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          working_directory: terraform/
          minimum_severity: HIGH
          format: sarif
          sarif_file: tfsec.sarif

      - name: Upload tfsec SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: tfsec.sarif
```

> `[SCREENSHOT]` — *GitHub Actions run showing both Checkov and tfsec jobs — one passing (green) and one failing (red) due to a misconfigured security group, with the finding details in the job output*

> `[SCREENSHOT]` — *GitHub PR showing a "Security scanning / checkov" check failing with a link to the details, blocking the merge*

---

## Lab — Scan MindCraft Terraform Code

**Objective:** Run Checkov and tfsec on your existing MindCraft infrastructure code and review findings.

1. Navigate to your Terraform directory:
```bash
cd "d:/Mhdomer_logs/mhdomer.github.io/_posts/MindCraft Cloud Deployment"
```

2. Run Checkov:
```bash
checkov -d . --compact --framework terraform
```

> `[SCREENSHOT]` — *Checkov output showing failed checks on the MindCraft Terraform code with check IDs and resource names*

3. Run tfsec:
```bash
tfsec . --minimum-severity MEDIUM
```

> `[SCREENSHOT]` — *tfsec output on the MindCraft Terraform code showing findings with code snippets*

4. Pick 2-3 failing checks → open the Terraform file → apply the fix:

**Example fix — enable S3 bucket encryption:**
```hcl
# Before
resource "aws_s3_bucket" "app-data" {
  bucket = "mindcraft-app-data"
}

# After
resource "aws_s3_bucket" "app-data" {
  bucket = "mindcraft-app-data"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "app-data" {
  bucket = aws_s3_bucket.app-data.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}
```

5. Re-run Checkov — confirm the fixed checks now pass

> `[SCREENSHOT]` — *Checkov output after the fix showing the previously failed S3 encryption check now marked as PASSED*

---

## Key Takeaways

- IaC scanning runs before `terraform apply` — fix misconfigs before they become real infrastructure
- Checkov covers Terraform + Kubernetes + Dockerfile + CloudFormation — use it as the default
- tfsec is faster and Terraform-focused — run both for maximum coverage
- Inline skip comments with justifications for intentional exceptions — don't just ignore findings
- Trigger IaC scans only on changes to `.tf` files to keep the pipeline fast
- SARIF output + GitHub Security tab = persistent tracking of IaC findings across PRs

---

## References

<div class="references">
<ul>
  <li><a href="https://www.checkov.io/1.Welcome/What%20is%20Checkov.html" target="_blank">Checkov Documentation</a></li>
  <li><a href="https://aquasecurity.github.io/tfsec/" target="_blank">tfsec Documentation</a></li>
  <li><a href="https://github.com/bridgecrewio/checkov-action" target="_blank">Checkov GitHub Action</a></li>
  <li><a href="https://github.com/aquasecurity/tfsec-action" target="_blank">tfsec GitHub Action</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
