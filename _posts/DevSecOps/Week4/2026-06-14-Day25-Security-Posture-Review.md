---
layout: post
title: "Week 4 — Day 25: Putting It All Together — Security Posture Review"
date: 2026-06-14 10:00:00 +0800
categories:
  - DevSecOps
  - Week4
tags:
  - DevSecOps
  - CloudSecurity
  - Architecture
  - Review
author: muhammed
description: A full review connecting all four weeks — drawing the complete security architecture, identifying your weakest areas, and writing a personal threat model for your home lab and blog infrastructure.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fredsiege.com%2Fwp-content%2Fuploads%2F2024%2F10%2FAsset-1spr.png&f=1&nofb=1&ipt=0ef99da3138f6209e680f91a6e4ca3be7a8e5a2a9a701332f809807f70699ba8
---

## What We Built Over 4 Weeks

This is the final day of the plan. Before moving forward, it's worth connecting all the pieces into a coherent security architecture picture. Real-world security isn't individual tools in isolation — it's layers that reinforce each other.

---

## The Full Security Stack

```
┌─────────────────────────────────────────────────────────────────┐
│                     IDENTITY LAYER                              │
│  IAM Identity Center → IAM Roles → SCPs → Permission Boundaries │
│  MFA enforced → Least privilege → No long-lived human keys       │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                     PERIMETER LAYER                             │
│  CloudFront → WAF (OWASP rules, rate limiting) → Shield         │
│  Route 53 → ALB → Zero Trust network (no implicit internal trust)│
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                     WORKLOAD LAYER                              │
│  EC2/ECS/EKS hardened → non-root containers → read-only FS      │
│  Trivy scans in CI → Falco at runtime → Inspector continuously   │
│  Secrets Manager / Vault → no hardcoded credentials             │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                     PIPELINE LAYER                              │
│  Semgrep (SAST) → Snyk (SCA) → Checkov/tfsec (IaC)             │
│  Trivy (image) → ZAP (DAST) → Branch protection gates           │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                     DATA LAYER                                  │
│  KMS encryption at rest → TLS in transit → S3 block public      │
│  RDS encrypted → EBS encrypted by default → Secrets in vault    │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                     DETECTION LAYER                             │
│  CloudTrail → AWS Config → GuardDuty → Security Hub             │
│  Falco → CloudWatch alarms → CIS benchmark monitoring            │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                     RESPONSE LAYER                              │
│  IR runbooks → Quarantine SG → Key revocation procedures        │
│  EBS snapshots → Athena forensics → EventBridge automation       │
└─────────────────────────────────────────────────────────────────┘
```

 — *The above architecture drawn cleanly in draw.io or on a whiteboard, with all 7 layers labeled and the key services/tools listed in each layer*

---

## How the Layers Reinforce Each Other

| If this fails... | This layer catches it |
|-----------------|----------------------|
| SAST misses a SQL injection | DAST finds it in the running app |
| A vulnerable dependency slips through | Inspector catches the CVE in the deployed container |
| A container escapes its sandbox | Falco detects the syscall anomaly |
| A developer accidentally pushes with `--no-verify` | GuardDuty detects the subsequent unusual API call |
| A secret gets leaked in a log | Secrets Manager + CloudTrail shows who retrieved it |
| An attacker gets an IAM key | SCP prevents them disabling CloudTrail; GuardDuty alerts on the anomaly |
| A misconfigured Terraform resource deploys | Config rule flags it and triggers auto-remediation |

Defense in depth means no single control failure = full compromise.

---

## Self-Assessment — Where Are Your Gaps?

Go through each layer and honestly rate your current coverage:

| Layer | Control | Status |
|-------|---------|--------|
| Identity | IAM Identity Center for all human access | ☐ Done / ☐ Not yet |
| Identity | MFA enforced via SCP | ☐ Done / ☐ Not yet |
| Identity | No root access keys | ☐ Done / ☐ Not yet |
| Identity | All service roles follow least privilege | ☐ Done / ☐ Not yet |
| Perimeter | WAF with OWASP managed rules on ALB | ☐ Done / ☐ Not yet |
| Perimeter | Rate limiting on auth endpoints | ☐ Done / ☐ Not yet |
| Workload | All containers run as non-root | ☐ Done / ☐ Not yet |
| Workload | Trivy integrated in CI pipeline | ☐ Done / ☐ Not yet |
| Workload | Falco deployed on K8s nodes | ☐ Done / ☐ Not yet |
| Workload | No hardcoded secrets anywhere | ☐ Done / ☐ Not yet |
| Pipeline | Semgrep running on every PR | ☐ Done / ☐ Not yet |
| Pipeline | Snyk scanning dependencies | ☐ Done / ☐ Not yet |
| Pipeline | Checkov scanning IaC | ☐ Done / ☐ Not yet |
| Pipeline | ZAP baseline on every deployment | ☐ Done / ☐ Not yet |
| Data | All S3 buckets block public access | ☐ Done / ☐ Not yet |
| Data | RDS and EBS encrypted | ☐ Done / ☐ Not yet |
| Data | VPC endpoints for AWS service calls | ☐ Done / ☐ Not yet |
| Detection | GuardDuty enabled all regions | ☐ Done / ☐ Not yet |
| Detection | Security Hub with CIS standard | ☐ Done / ☐ Not yet |
| Detection | CloudTrail multi-region with validation | ☐ Done / ☐ Not yet |
| Detection | CIS CloudWatch alarms configured | ☐ Done / ☐ Not yet |
| Response | IR runbook for compromised key | ☐ Done / ☐ Not yet |
| Response | IR runbook for compromised EC2 | ☐ Done / ☐ Not yet |
| Response | Quarantine SG pre-created | ☐ Done / ☐ Not yet |

> `[SCREENSHOT]` — *The above checklist printed or filled in, showing which items are done and which are still gaps — your personal security posture snapshot*

---

## Personal Threat Model — Blog Infrastructure

Applying the Week 4 threat modeling methodology to your actual infrastructure.

### Components

| Component | Description |
|-----------|-------------|
| GitHub repo | Source for all blog posts and notes |
| GitHub Actions | Builds and deploys via Jekyll |
| GitHub Pages | Hosts the public site |
| Obsidian vault | Local note-taking (on Windows) |
| Local machine | Windows 11, development environment |

### DFD

```
[You — Windows 11]
      |
      | git push (HTTPS/SSH)
      ▼
[GitHub Repo]
      |
      | triggers on push
      ▼
[GitHub Actions] ──── reads secrets (SNYK_TOKEN, etc.)
      |
      | deploys built site
      ▼
[GitHub Pages]
      |
      | serves via HTTPS
      ▼
[Public Readers]

[Obsidian Vault]
      |
      | files in git-ignored dirs
      ▼
[Local filesystem only — not published]
```

### STRIDE Analysis

| Component / Flow | Threat | Risk | Mitigation |
|-----------------|--------|------|-----------|
| GitHub repo | **T** — malicious PR modifying workflow file to exfiltrate secrets | High | Require review for workflow file changes, restrict fork PR permissions |
| GitHub repo | **I** — accidental commit of local notes or `.env` | Medium | `.gitignore` correct, no secrets in committed files |
| GitHub Actions | **I** — secrets printed in logs | Medium | Never echo secrets, use masked variables |
| GitHub Actions | **E** — workflow with `write` permissions to all repo contents | Medium | Use minimal GITHUB_TOKEN permissions per job |
| GitHub Pages | **D** — site taken down (GitHub outage) | Low | Accept — no mitigation needed (free tier) |
| GitHub Pages | **I** — sensitive personal info accidentally published in a post | Medium | Review all posts before publish, don't include PII |
| Local machine | **S** — Windows Defender deletes legitimate security notes (happened already) | Medium | Add Obsidian/Jekyll folder to Defender exclusions |
| Local machine | **I** — Obsidian vault contains sensitive course notes | Low | Keep in gitignored dirs, don't sync to cloud without encryption |

### Mitigations to Implement

| Priority | Action | Effort |
|----------|--------|--------|
| High | Set `permissions: read-all` as default in workflow files, override per job | 15 min |
| High | Add branch protection requiring review for `/.github/workflows/` changes | 5 min |
| Medium | Add `.github/workflows/semgrep.yml` to scan blog posts for accidentally committed secrets | 30 min |
| Medium | Add Windows Defender exclusion for the Jekyll folder permanently | 5 min |
| Low | Review all published posts for PII once | 1 hour |

---

## Minimal Workflow Hardening

Apply this to your existing GitHub Actions workflows:

```yaml
# Add to the top of every workflow file
permissions:
  contents: read     # default: read-only

jobs:
  deploy:
    permissions:
      contents: write    # override only for jobs that need it
      pages: write
      id-token: write    # only if using OIDC
```

```yaml
# Scope GITHUB_TOKEN to minimum per job
jobs:
  semgrep:
    permissions:
      contents: read
      security-events: write   # only for uploading SARIF
```

> `[SCREENSHOT]` — *GitHub workflow file in editor showing the top-level permissions block set to read-all and individual job-level overrides for specific permissions*

---

## What to Build Next

With all four weeks complete, the natural next steps:

**Immediate (this month):**
- Apply the self-assessment checklist to your actual AWS environment
- Set up the full CI pipeline (Day 19) on a real project
- Write and test your first IR runbook

**Next 3 months:**
- Get hands-on with AWS Security Hub CIS remediation (score to 80%+)
- Deploy Falco on a Kubernetes cluster and tune the rules
- Do a real threat model session on the MindCraft application
- Practice a simulated IR exercise with a test compromised key

**Certifications to pursue:**
| Cert | Relevance |
|------|-----------|
| AWS Security Specialty | Validates everything in Week 1 + 4 |
| CKS (Certified Kubernetes Security Specialist) | Validates Week 2 container security |
| OSCP (in progress) | Attacker mindset complements the defender skills here |

> `[SCREENSHOT]` — *AWS Certification roadmap or a hand-written personal roadmap showing the planned certifications and timeline*

---

## Key Takeaways

- Security is a stack of layers, not a single tool — knowing where each layer fits is as important as knowing how to use each tool
- Your weakest layer defines your security posture — a perfect pipeline with no detection is just as bad as no pipeline with great detection
- Write your IR runbooks now, before an incident — practice them quarterly
- Threat modeling your own infrastructure (even a blog) builds the habit — make it a reflex before any new build
- The tools in this plan map directly to job requirements for DevSecOps Engineer, Cloud Security Engineer, and Platform Security roles

---

## References

<div class="references">
<ul>
  <li><a href="https://aws.amazon.com/architecture/security-identity-compliance/" target="_blank">AWS Security Reference Architecture</a></li>
  <li><a href="https://docs.aws.amazon.com/whitepapers/latest/aws-security-incident-response-guide/welcome.html" target="_blank">AWS Security Incident Response Guide</a></li>
  <li><a href="https://owasp.org/www-project-devsecops-guideline/" target="_blank">OWASP DevSecOps Guideline</a></li>
  <li><a href="https://aws.amazon.com/certification/certified-security-specialty/" target="_blank">AWS Certified Security Specialty</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
