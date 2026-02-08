---
layout: post
title: DevSecOps Labs — Index & Ideas
date: 2026-06-14 10:00:00 +0800
categories:
  - DevSecOps
  - Labs
tags:
  - DevSecOps
  - Labs
  - CloudSecurity
  - HandsOn
author: muhammed
description: A personal lab index — hands-on exercises mapped to each week of the DevSecOps study plan. Each lab has a clear objective, tools needed, and success criteria.
toc: true
pin: false
math: false
mermaid: false
image: https://veritis.com/wp-content/uploads/2022/06/all-you-need-to-know-about-devsecops-and-its-implementation.jpg
---

## How to Use This Index

Each lab maps to a week in the [study plan]({% post_url DevSecOps/2026-05-19-Cloud-Security-DevSecOps-Study-Plan %})


When you complete a lab, write it up in its own post inside this folder and link it here.

**Lab writeup format:** What you did → what you found → what you fixed → what you learned.
Screenshots are the evidence — every step should have one.

---

## Week 1 — AWS Security Services

### Lab 1.1 — IAM Least Privilege Role + Policy Simulator
**Objective:** Create a role with minimal S3 access and verify no other permissions work.
**Tools:** AWS Console, IAM Policy Simulator, AWS CLI
**Success criteria:** `s3:GetObject` allowed, `s3:DeleteObject` denied, `ec2:*` denied — all confirmed in simulator
**Writeup:** _(done)_

---

### Lab 1.2 — Trigger and Investigate a GuardDuty Finding
**Objective:** Generate sample findings, trace one through to Security Hub, and simulate a response.
**Tools:** GuardDuty, Security Hub, EventBridge
**Success criteria:** Sample finding visible in Security Hub with correct severity, EventBridge rule fires on High finding
**Writeup:** _(not yet done)_

---

### Lab 1.3 — Config Rule + Auto-Remediation for Public S3
**Objective:** Create a Config rule that detects a public S3 bucket and auto-remediates it.
**Tools:** AWS Config, S3, SSM Automation
**Success criteria:** Make a bucket public → Config flags it within 5 min → auto-remediation reverts it
**Writeup:** _(not yet done)_

---

### Lab 1.4 — CloudTrail Forensics with Athena
**Objective:** Simulate a suspicious API call and trace it using Athena queries on CloudTrail logs.
**Tools:** CloudTrail, Athena, S3
**Success criteria:** Query returns the exact API call, source IP, and user identity of the simulated action
**Writeup:** _(not yet done)_

---

### Lab 1.5 — WAF Setup with OWASP Rules on an ALB
**Objective:** Attach a WAF Web ACL to an ALB with OWASP managed rules and test SQLi blocking.
**Tools:** AWS WAF, ALB, curl
**Success criteria:** Normal requests pass, SQLi payload in query string returns 403
**Writeup:** _(not yet done)_

---

## Week 2 — Secrets Management & Container Security

### Lab 2.1 — Secrets Manager Rotation with RDS
**Objective:** Store RDS credentials in Secrets Manager and trigger automatic rotation.
**Tools:** AWS Secrets Manager, RDS, Lambda, Python boto3
**Success criteria:** Application retrieves credentials via SDK (no hardcoding), rotation runs successfully, new password works on DB
**Writeup:** _(not yet done)_

---

### Lab 2.2 — HashiCorp Vault Dynamic AWS Credentials
**Objective:** Use Vault to generate temporary IAM credentials on demand and verify they expire.
**Tools:** Vault (dev mode), AWS IAM
**Success criteria:** `vault read aws/creds/s3-reader` returns a temporary key, key expires after lease, IAM user auto-deleted
**Writeup:** _(not yet done)_

---

### Lab 2.3 — Dockerfile Hardening Before and After
**Objective:** Take an insecure Dockerfile, apply all hardening principles, and run Docker Bench before and after.
**Tools:** Docker, Docker Bench for Security
**Success criteria:** Docker Bench WARN count drops by at least 5 after hardening, container runs as non-root
**Writeup:** _(not yet done)_

---

### Lab 2.4 — Trivy — Old Image vs Updated Image CVE Comparison
**Objective:** Scan an old base image, update it, re-scan, and document the CVE reduction.
**Tools:** Trivy, Docker
**Success criteria:** Before/after table showing Critical/High CVE count before update vs after, at least 50% reduction
**Writeup:** _(not yet done)_

---

### Lab 2.5 — Kubernetes RBAC Lockdown + Network Policy Isolation
**Objective:** Deploy two pods, lock down RBAC to dedicated service accounts, apply network policy, verify isolation.
**Tools:** kubectl, Kubernetes, a local cluster (Kind or Minikube)
**Success criteria:** Pod A cannot reach Pod B (connection timeout), Pod B can be reached only from allowed source
**Writeup:** _(not yet done)_

---

### Lab 2.6 — Falco — Trigger and Tune Rules
**Objective:** Install Falco, trigger 3 different default rules, then write one custom rule for your environment.
**Tools:** Falco, Helm, kubectl
**Success criteria:** 3 alert types appear in Falco logs, custom rule fires correctly on the target behavior
**Writeup:** _(not yet done)_

---

## Week 3 — Pipeline Security

### Lab 3.1 — Semgrep on Juice Shop
**Objective:** Scan OWASP Juice Shop with Semgrep, identify 3 real vulnerabilities, write fixes.
**Tools:** Semgrep, OWASP Juice Shop
**Success criteria:** 3 findings documented with file/line/description, at least 1 fixed and re-scanned to confirm resolution
**Writeup:** _(not yet done)_

---

### Lab 3.2 — Snyk on a Vulnerable Node Project
**Objective:** Scan a known-vulnerable Node.js app, trace a transitive vulnerability, apply snyk fix.
**Tools:** Snyk CLI, Node.js
**Success criteria:** Full dependency chain documented for 1 High CVE, snyk fix applied, re-scan shows reduced count
**Writeup:** _(not yet done)_

---

### Lab 3.3 — Checkov on MindCraft Terraform
**Objective:** Run Checkov on the MindCraft Terraform code, pick 5 failures, fix them.
**Tools:** Checkov, Terraform
**Success criteria:** Before count vs after count documented, 5 checks moved from FAILED to PASSED
**Writeup:** _(not yet done)_

---

### Lab 3.4 — ZAP Baseline Scan on Juice Shop
**Objective:** Run a ZAP baseline scan against Juice Shop, identify missing headers, add them, re-scan.
**Tools:** OWASP ZAP (Docker), Juice Shop (Docker)
**Success criteria:** HTML report saved, at least 2 Medium findings fixed and confirmed gone in second scan
**Writeup:** _(not yet done)_

---

### Lab 3.5 — Full Secure Pipeline on a Real Repo
**Objective:** Implement the Day 19 pipeline (Semgrep + Snyk + Checkov + Trivy + ZAP) on an actual GitHub repo with branch protection.
**Tools:** GitHub Actions, all pipeline tools
**Success criteria:** All 5 jobs run on PR, branch protection enforces all checks, a deliberate finding causes a blocked merge
**Writeup:** _(not yet done)_

---

## Week 4 — Advanced

### Lab 4.1 — Simulated IR: Compromised IAM Key Exercise
**Objective:** Use a test IAM key, simulate a compromise, execute the full IR runbook end-to-end.
**Tools:** AWS CLI, CloudTrail, Athena, IAM
**Success criteria:** Key deactivated within 5 min, all attacker API calls identified via Athena, any persistence cleaned up, runbook completed with timestamps
**Writeup:** _(not yet done)_

---

### Lab 4.2 — Threat Model MindCraft with OWASP Threat Dragon
**Objective:** Draw the full MindCraft DFD in Threat Dragon and generate a complete STRIDE threat list.
**Tools:** OWASP Threat Dragon, draw.io
**Success criteria:** DFD covers all components, at least 10 STRIDE threats identified, top 3 have mitigations defined
**Writeup:** _(not yet done)_

---

### Lab 4.3 — CIS Benchmark Remediation Sprint
**Objective:** Get Security Hub CIS Level 1 compliance score from current baseline to 80%+.
**Tools:** AWS Security Hub, AWS CLI, Terraform
**Success criteria:** Before/after score documented, all fixed controls listed with the CLI/Terraform command used
**Writeup:** _(not yet done)_

---

### Lab 4.4 — Zero Trust Micro-Segmentation on a 3-Tier App
**Objective:** Deploy a 3-tier app (web, app, DB), replace all CIDR-based SG rules with SG references, verify lateral movement is blocked.
**Tools:** Terraform, AWS EC2, Security Groups
**Success criteria:** Web tier → App tier works, Web tier → DB direct connection times out, Terraform plan shows no CIDR rules
**Writeup:** _(not yet done)_

---

### Lab 4.5 — SCP Write and Test on a Test Account
**Objective:** Write a multi-control SCP (block CloudTrail disable, block leaving org, restrict regions), attach to a test OU, verify it blocks as expected.
**Tools:** AWS Organizations, SCPs, AWS CLI
**Success criteria:** Each denied action returns `AccessDenied` from within the test account, management account is unaffected
**Writeup:** _(not yet done)_

---

## Bonus Labs

### Bonus 1 — Prowler Full Account Scan
**Objective:** Run Prowler (open-source security tool) against your AWS account and compare findings to Security Hub.
**Tools:** Prowler, AWS
**Why:** Prowler covers checks Security Hub misses, good for audit prep
**Writeup:** _(not yet done)_

---

### Bonus 2 — EC2 Instance Compromise Simulation
**Objective:** Intentionally "compromise" an isolated EC2 instance, practice the full forensic procedure — snapshot, quarantine SG, SSM analysis.
**Tools:** AWS EC2, SSM, CloudTrail
**Why:** Practice IR under no pressure before doing it under real pressure
**Writeup:** _(not yet done)_

---

### Bonus 3 — Build and Scan a Vulnerable Docker Image
**Objective:** Build a container with an old OS and known-vulnerable packages, scan with Trivy, fix layer by layer, document CVE reduction at each step.
**Tools:** Docker, Trivy
**Why:** Understand how image layers accumulate CVEs and how fixing the base image differs from fixing app deps
**Writeup:** _(not yet done)_

---

### Bonus 4 — Secret Scanning a Git History
**Objective:** Use Trufflehog and Gitleaks to scan a git repo history (not just current code) for secrets ever committed.
**Tools:** Trufflehog, Gitleaks
**Why:** Secrets committed and later deleted are still in git history — this is how many real breaches happen
**Writeup:** _(not yet done)_

---

## Progress Tracker

| Lab | Status | Date Completed |
|-----|--------|----------------|
| 1.1 IAM Policy Simulator | ☐ | |
| 1.2 GuardDuty Finding | ☐ | |
| 1.3 Config Auto-Remediation | ☐ | |
| 1.4 CloudTrail Athena Forensics | ☐ | |
| 1.5 WAF + ALB | ☐ | |
| 2.1 Secrets Manager Rotation | ☐ | |
| 2.2 Vault Dynamic Credentials | ☐ | |
| 2.3 Docker Hardening | ☐ | |
| 2.4 Trivy CVE Comparison | ☐ | |
| 2.5 K8s RBAC + Network Policy | ☐ | |
| 2.6 Falco Rules | ☐ | |
| 3.1 Semgrep on Juice Shop | ☐ | |
| 3.2 Snyk Vulnerable Node | ☐ | |
| 3.3 Checkov on MindCraft | ☐ | |
| 3.4 ZAP on Juice Shop | ☐ | |
| 3.5 Full Secure Pipeline | ☐ | |
| 4.1 IR Simulation | ☐ | |
| 4.2 Threat Model MindCraft | ☐ | |
| 4.3 CIS Remediation Sprint | ☐ | |
| 4.4 Zero Trust Micro-Segmentation | ☐ | |
| 4.5 SCP Write and Test | ☐ | |
| Bonus 1 Prowler | ☐ | |
| Bonus 2 EC2 Compromise Sim | ☐ | |
| Bonus 3 Vulnerable Image Layers | ☐ | |
| Bonus 4 Git History Secret Scan | ☐ | |

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)


- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
