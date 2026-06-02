---
layout: post
title: "Week 2 — Day 11: Container Image Scanning with Trivy"
date: 2026-03-11 10:00:00 +0800
categories:
  - DevSecOps
  - Week2
tags:
  - Trivy
  - ContainerSecurity
  - VulnerabilityScanning
  - DevSecOps
  - CI/CD
author: muhammed
description: A full walkthrough of Trivy — scanning container images, filesystems, Git repos, and IaC files for vulnerabilities and misconfigurations, plus integrating it into GitHub Actions.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fi.ytimg.com%2Fvi%2FrP6VpIzb1Jw%2Fmaxresdefault.jpg&f=1&nofb=1&ipt=145e85e4e47c053c6252f314a92129ac39f7670d8667e5799f14847cf5dc66c1
---

## What is Trivy?

Trivy is an open-source, all-in-one security scanner by Aqua Security. It scans for:

| Target | What it finds |
|--------|--------------|
| Container images | OS package CVEs, language library CVEs |
| Filesystems | Same as above, on local directories |
| Git repositories | Secrets, CVEs in dependency files |
| IaC files | Misconfigurations in Terraform, Dockerfile, K8s manifests |
| SBOM | Vulnerabilities in a Software Bill of Materials |

Trivy is fast, has no server component, and is the most widely used container scanner in CI/CD pipelines.

---

## Installation

```bash
# Windows (via Scoop)
scoop install trivy

# Or download the binary from GitHub releases
# https://github.com/aquasecurity/trivy/releases

# Verify
trivy --version
```

---

## Scanning Container Images

### Basic Scan

```bash
trivy image nginx:latest
```

> `[SCREENSHOT]` — *Terminal showing trivy image nginx:latest output — a table with columns: Library, Vulnerability ID, Severity, Installed Version, Fixed Version, Title*

Trivy scans:
1. The OS packages (apt/apk/yum)
2. Application libraries (pip, npm, gem, cargo, etc.)
3. For each: checks its vulnerability database (NVD, GitHub Advisory, OS vendor advisories)

---

### Filtering by Severity

```bash
# Only show Critical and High
trivy image --severity CRITICAL,HIGH nginx:latest

# Only Critical
trivy image --severity CRITICAL nginx:latest
```

> `[SCREENSHOT]` — *Terminal showing filtered output with only CRITICAL and HIGH rows, making the output much more manageable*

---

### Failing CI on Critical Findings

```bash
# Exit code 1 if any CRITICAL CVE is found — use this in CI to block the pipeline
trivy image --severity CRITICAL --exit-code 1 myapp:latest
```

If the scan finds a Critical CVE, the command exits with code 1, failing the pipeline step.

---

### Scanning a Specific Architecture

```bash
trivy image --platform linux/amd64 myapp:latest
```

Important when building multi-arch images on an ARM Mac — scan the architecture that will actually run in production.

---

### Output Formats

```bash
# JSON output (for parsing or feeding into other tools)
trivy image --format json --output results.json nginx:latest

# SARIF output (for GitHub Code Scanning)
trivy image --format sarif --output results.sarif nginx:latest

# Table (default)
trivy image --format table nginx:latest
```

> `[SCREENSHOT]` — *Terminal showing trivy image --format json output piped through jq showing the structured vulnerability data*

---

## Scanning Filesystems and Repos

### Scan a Local Directory

```bash
# Scan your project directory for vulnerabilities in dependency files
trivy fs .

# This picks up: package-lock.json, requirements.txt, Gemfile.lock, go.sum, etc.
```

> `[SCREENSHOT]` — *Terminal showing trivy fs . output scanning a Node.js project directory and finding CVEs in node_modules packages*

### Scan a Git Repository

```bash
# Scan a remote repo without cloning
trivy repo https://github.com/yourusername/yourrepo

# Scan and also check for secrets
trivy repo --scanners vuln,secret https://github.com/yourusername/yourrepo
```

> `[SCREENSHOT]` — *Terminal showing trivy repo scanning a GitHub repo URL and listing found vulnerabilities from the dependency files*

---

## Scanning IaC Files

Trivy checks Dockerfiles, Terraform, Kubernetes manifests, and Helm charts for misconfigurations.

```bash
# Scan a Dockerfile
trivy config Dockerfile

# Scan a Terraform directory
trivy config ./terraform/

# Scan a Kubernetes manifest
trivy config k8s-deployment.yaml
```

> `[SCREENSHOT]` — *Terminal showing trivy config Dockerfile output listing misconfigurations like "Specify at least 1 USER command" and "Do not use sudo" with their severity levels*

```bash
# Scan your MindCraft Terraform code
trivy config ./terraform/ --severity MEDIUM,HIGH,CRITICAL
```

> `[SCREENSHOT]` — *Terminal showing trivy config scanning the Terraform directory and finding misconfigurations like open security group rules or unencrypted S3 buckets*

---

## Secret Scanning

Trivy can scan for hardcoded secrets (API keys, passwords, tokens) in your codebase:

```bash
trivy fs --scanners secret .
```

> `[SCREENSHOT]` — *Terminal showing trivy fs --scanners secret finding a hardcoded AWS access key in a config file, with the file path and line number shown*

Trivy detects secrets like:
- AWS access keys
- GitHub tokens
- Private keys (RSA, EC)
- Generic API key patterns

---

## Understanding CVE Output

Each finding shows:

| Field | Meaning |
|-------|---------|
| Library | The package with the vulnerability |
| Vulnerability ID | CVE identifier |
| Severity | CRITICAL / HIGH / MEDIUM / LOW |
| Installed Version | What you have |
| Fixed Version | What you need to upgrade to |
| Title | Short description of the vulnerability |

**When "Fixed Version" is empty:** No fix is available yet. Your options are:
1. Accept the risk (document it)
2. Switch to a different package
3. Use a different base image that doesn't include this package

---

## Ignoring False Positives

Create a `.trivyignore` file in your project to suppress known false positives or accepted risks:

```
# .trivyignore

# Accepted risk — no fix available, low exploitability in our context
CVE-2023-12345

# This CVE is in a test-only dependency, not in production
CVE-2023-67890
```

```bash
trivy image --ignorefile .trivyignore myapp:latest
```

> `[SCREENSHOT]` — *Terminal showing trivy image with --ignorefile applied, and the previously shown CVE now absent from the output with a note "1 vulnerability suppressed by .trivyignore"*

---

## Integrating Trivy into GitHub Actions

### Basic Image Scan

```yaml
# .github/workflows/trivy.yml
name: Trivy Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy image scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Upload results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif
```

> `[SCREENSHOT]` — *GitHub Actions run showing the Trivy scan step completing — either green (no critical CVEs) or red with the CVEs listed in the step output*

The SARIF upload makes findings appear in the **GitHub Security tab → Code scanning alerts**.

> `[SCREENSHOT]` — *GitHub repository → Security tab → Code scanning showing Trivy findings listed with severity badges and the affected file/package*

---

### Full Pipeline with IaC + Image Scan

```yaml
name: Full Security Scan

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Trivy IaC scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: .
          severity: HIGH,CRITICAL
          exit-code: 0   # don't fail on IaC findings (informational for now)

      - name: Build image
        run: docker build -t myapp:test .

      - name: Trivy image scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:test
          severity: CRITICAL
          exit-code: 1   # fail on Critical image CVEs
```

---

## Lab — Scan a Public Image and Fix It

**Objective:** Scan an old image, understand the findings, update the base image, re-scan.

1. Scan an old Python image:
```bash
trivy image --severity CRITICAL,HIGH python:3.9
```

> `[SCREENSHOT]` — *Terminal showing trivy image python:3.9 output with multiple CRITICAL and HIGH CVEs listed*

2. Count the Critical findings:
```bash
trivy image --severity CRITICAL --format json python:3.9 | jq '.Results[].Vulnerabilities | length'
```

3. Scan the latest slim version:
```bash
trivy image --severity CRITICAL,HIGH python:3.12-slim
```

> `[SCREENSHOT]` — *Terminal showing trivy image python:3.12-slim with significantly fewer or zero CRITICAL findings — demonstrating the impact of keeping the base image updated*

4. Update your Dockerfile `FROM` line and rebuild
5. Re-scan your built image — verify CVE count dropped

---

## Key Takeaways

- Trivy is a single binary that scans images, filesystems, repos, IaC, and secrets — use it everywhere
- Add `--exit-code 1 --severity CRITICAL` to your CI pipeline to block on critical CVEs
- Upload SARIF results to GitHub Security tab for persistent tracking
- Keep base images updated — most CVEs are in outdated OS packages
- Use `.trivyignore` for accepted risks, not as a way to silence everything
- Scan both the image AND the IaC in the same pipeline

---

## References

<div class="references">
<ul>
  <li><a href="https://trivy.dev/latest/docs/" target="_blank">Trivy Documentation</a></li>
  <li><a href="https://github.com/aquasecurity/trivy-action" target="_blank">Trivy GitHub Action</a></li>
  <li><a href="https://aquasecurity.github.io/trivy/latest/docs/scanner/vulnerability/" target="_blank">Trivy Vulnerability Scanner</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
