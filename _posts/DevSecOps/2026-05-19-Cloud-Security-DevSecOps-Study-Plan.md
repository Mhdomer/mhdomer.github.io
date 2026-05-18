---
layout: post
title: Cloud Security & DevSecOps — Daily Study Plan
date: 2026-05-19 10:00:00 +0800
categories:
  - DevSecOps
tags:
  - DevSecOps
  - CloudSecurity
  - AWS
  - StudyPlan
author: muhammed
description: A structured daily study plan to fill the remaining gaps in cloud security and DevSecOps — covering IAM, AWS security services, secrets management, container hardening, pipeline security, and incident response.
toc: true
pin: true
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fdt-cdn.net%2Fimages%2Fdevsecops-image-2000-6557ba1b00.png&f=1&nofb=1&ipt=2c97bb8d4855a9ead057f0aeac1acf98499eb138df50b81d60d1583dde3a945e 
---

## Overview

This plan builds on top of existing Docker, Terraform, CI/CD, and Observability knowledge. It targets the gaps that matter most for real cloud security and DevSecOps roles.

| Week | Focus Area |
|------|-----------|
| Week 1 | IAM & AWS Security Services |
| Week 2 | Secrets Management & Container Security |
| Week 3 | Pipeline Security (SAST / DAST / SCA) |
| Week 4 | Zero Trust, IR & Threat Modeling |

---

## Week 1 — IAM & AWS Security Services
> May 20 – May 26

### Day 1 — AWS IAM Deep Dive
**Goal:** Understand how identity works in AWS before touching any security tooling.

- IAM Users, Groups, Roles, Policies (inline vs managed)
- Least privilege principle — how to apply it in practice
- Policy simulator — test policies before deploying
- Conditions in IAM policies (IP, MFA, time-based)
- **Lab:** Create a role with least-privilege S3 access and test with the simulator

**Resources:**
- TryHackMe: AWS IAM room
- AWS docs: IAM Policy Reference

---

### Day 2 — SCPs & Permission Boundaries
**Goal:** Understand org-level controls that override IAM.

- AWS Organizations & Service Control Policies (SCPs)
- Difference between SCPs and IAM policies
- Permission boundaries — what they prevent
- **Lab:** Write an SCP that denies disabling CloudTrail across all accounts

---

### Day 3 — CloudTrail & AWS Config
**Goal:** Know how to audit what happened and detect drift.

- CloudTrail: what it logs, multi-region setup, log integrity
- AWS Config: rules, conformance packs, remediation actions
- Detecting config drift with Config rules
- **Lab:** Create a Config rule that flags publicly accessible S3 buckets

---

### Day 4 — GuardDuty & Security Hub
**Goal:** Understand AWS native threat detection.

- GuardDuty: finding types, data sources (VPC Flow Logs, DNS, CloudTrail)
- Security Hub: aggregating findings, CSPM, CIS Benchmark checks
- How findings flow: GuardDuty → Security Hub → EventBridge → alert
- **Lab:** Enable GuardDuty in a test account, trigger a sample finding

---

### Day 5 — Amazon Inspector & Patch Management
**Goal:** Vulnerability scanning at the infrastructure level.

- Inspector v2: EC2 scanning, ECR image scanning, Lambda scanning
- Understanding CVE severity and EPSS scoring
- Integrating Inspector findings into Security Hub
- **Lab:** Scan an EC2 instance and review findings

---

### Day 6 — WAF & Shield
**Goal:** Application-layer and DDoS protection.

- AWS WAF: rules, rule groups, managed rules (OWASP core set)
- Rate-based rules and bot control
- Shield Standard vs Advanced
- **Lab:** Attach a WAF to an ALB with OWASP managed rules enabled

---

### Day 7 — Week 1 Review
- Write up notes on all 6 topics
- Map each service to a MITRE ATT&CK cloud technique it detects or prevents
- Rest

---

## Week 2 — Secrets Management & Container Security
> May 27 – Jun 2

### Day 8 — AWS Secrets Manager & Parameter Store
**Goal:** Never hardcode credentials again.

- Secrets Manager vs SSM Parameter Store — when to use which
- Automatic secret rotation with Lambda
- Referencing secrets in ECS tasks, Lambda, and EC2 via IAM roles
- **Lab:** Store a DB password in Secrets Manager, retrieve it in a Python script using boto3

---

### Day 9 — HashiCorp Vault Basics
**Goal:** Understand Vault for multi-cloud or on-prem secrets.

- Vault architecture: secrets engines, auth methods, policies
- Dynamic secrets — generate short-lived DB credentials on demand
- Vault Agent for auto-renewal
- **Lab:** Run Vault in dev mode locally, create a KV secret, and retrieve it via CLI

---

### Day 10 — Docker Security Hardening
**Goal:** Secure containers from build to runtime.

- Non-root users in containers
- Read-only filesystems, dropped capabilities
- Dockerfile best practices (multi-stage builds, minimal base images)
- Docker Bench for Security
- **Lab:** Harden an existing Dockerfile and run Docker Bench against it

---

### Day 11 — Container Image Scanning with Trivy
**Goal:** Catch vulnerabilities before they reach production.

- Trivy: scanning images, filesystems, git repos, IaC
- Understanding CRITICAL vs HIGH CVEs in container context
- Integrating Trivy into a GitHub Actions pipeline
- **Lab:** Scan a public Docker image, fix or accept findings, integrate into CI

---

### Day 12 — Kubernetes RBAC & Pod Security
**Goal:** Secure the cluster not just the app.

- RBAC: Roles, ClusterRoles, RoleBindings
- Least privilege for service accounts
- Pod Security Standards (Restricted / Baseline / Privileged)
- Network Policies — isolate namespaces
- **Lab:** Create a restricted service account for a deployment, apply a network policy

---

### Day 13 — Runtime Security with Falco
**Goal:** Detect suspicious behavior inside running containers.

- Falco rules: syscall-based detection
- Default ruleset — what it catches (shell spawned in container, etc.)
- Alerting Falco events to Slack or a SIEM
- **Lab:** Install Falco on a Kubernetes node, trigger a rule by running a shell in a pod

---

### Day 14 — Week 2 Review
- Write up notes
- Build a mental model: secrets flow → container build → runtime protection
- Rest

---

## Week 3 — Pipeline Security (SAST / DAST / SCA)
> Jun 3 – Jun 9

### Day 15 — SAST with Semgrep
**Goal:** Catch code-level vulnerabilities before merge.

- What SAST is and what it can/can't catch
- Semgrep: writing custom rules, using community rulesets
- Integrating Semgrep into GitHub Actions as a PR check
- **Lab:** Run Semgrep on an intentionally vulnerable app (e.g., DVWA), review findings

---

### Day 16 — SCA with Snyk / Dependabot
**Goal:** Track vulnerable dependencies.

- SCA vs SAST — different problems
- Snyk: scanning `package.json`, `requirements.txt`, `go.mod`, Docker images
- Dependabot: auto PRs for dependency updates
- SBOM generation (Software Bill of Materials)
- **Lab:** Run `snyk test` on a Node.js project, fix or suppress findings

---

### Day 17 — IaC Scanning with Checkov & tfsec
**Goal:** Catch misconfigurations in Terraform before apply.

- Checkov: scanning Terraform, CloudFormation, Kubernetes manifests
- tfsec: focused on Terraform security checks
- Common findings: open security groups, unencrypted S3, public RDS
- **Lab:** Run Checkov on your MindCraft Terraform code, review findings

---

### Day 18 — DAST with OWASP ZAP
**Goal:** Test the running app from the outside.

- DAST vs SAST — runtime vs static
- ZAP: spidering, active scan, API scanning
- Running ZAP in CI against a staging environment
- **Lab:** Run ZAP baseline scan against a local vulnerable app (DVWA or Juice Shop)

---

### Day 19 — Full Secure Pipeline Design
**Goal:** Assemble all tools into one pipeline.

- Pipeline stages: SAST → SCA → Build → Image Scan → Deploy → DAST
- Fail fast vs warn — when to block the pipeline
- Security gates: what thresholds to set for CRITICAL findings
- **Lab:** Write a GitHub Actions workflow that runs Semgrep + Trivy + Checkov in sequence

---

### Day 20 — Week 3 Review
- Write up the full pipeline design as a post
- Rest

---

## Week 4 — Zero Trust, Incident Response & Threat Modeling
> Jun 10 – Jun 16

### Day 21 — Zero Trust Architecture
**Goal:** Understand the model and how AWS implements it.

- Zero Trust principles: verify explicitly, least privilege, assume breach
- AWS implementation: IAM Identity Center, VPC Lattice, PrivateLink
- Network micro-segmentation vs perimeter security
- **Read:** NIST SP 800-207 (Zero Trust Architecture) — summary only

---

### Day 22 — Cloud Incident Response
**Goal:** Know what to do when something goes wrong.

- IR phases in cloud context: Detect → Contain → Eradicate → Recover
- AWS tools for IR: CloudTrail forensics, VPC Flow Logs, GuardDuty findings
- Isolating a compromised EC2: snapshot, quarantine SG, revoke IAM keys
- **Lab:** Simulate a compromised access key, practice the containment steps

---

### Day 23 — Threat Modeling with STRIDE
**Goal:** Think like an attacker before building.

- STRIDE: Spoofing, Tampering, Repudiation, Info Disclosure, DoS, Elevation
- Applying STRIDE to a cloud-native app (API Gateway → Lambda → RDS)
- Drawing data flow diagrams (DFDs) for threat modeling
- **Lab:** Threat model your MindCraft application using STRIDE

---

### Day 24 — CIS Benchmarks & Compliance Basics
**Goal:** Understand how compliance maps to technical controls.

- CIS AWS Foundations Benchmark — key controls
- Security Hub: CIS Benchmark automated checks
- Mapping controls to SOC2 trust principles (brief overview)
- **Lab:** Run Security Hub CIS check, document failed controls and remediation

---

### Day 25 — Putting It All Together
**Goal:** Connect everything into a unified security posture.

- Draw your full security architecture: identity → network → workload → data → detection
- Identify your top 3 weakest areas from the past 4 weeks
- Write a one-page personal threat model for your home lab / blog infrastructure

---

### Day 26-27 — Final Review & Blog Posts
- Publish notes from each week as blog posts
- Update your CV/LinkedIn with the new skills
- Rest

---

## Quick Reference — Tools Covered

| Tool | Category | When to Use |
|------|----------|-------------|
| Trivy | Image/IaC scanning | Every image build |
| Checkov | IaC scanning | Every Terraform change |
| tfsec | IaC scanning | Terraform security focus |
| Semgrep | SAST | Every PR |
| Snyk | SCA | Dependency PRs |
| OWASP ZAP | DAST | Staging deployments |
| Falco | Runtime detection | Kubernetes clusters |
| GuardDuty | Cloud threat detection | Always-on in AWS |
| Security Hub | CSPM aggregation | Compliance dashboard |
| Vault | Secrets management | Multi-cloud or on-prem |

---

## References

<div class="references">
<ul>
  <li><a href="https://tryhackme.com/module/devsecops" target="_blank">TryHackMe: DevSecOps Path</a></li>
  <li><a href="https://docs.aws.amazon.com/security/" target="_blank">AWS Security Documentation</a></li>
  <li><a href="https://www.checkov.io/" target="_blank">Checkov by Bridgecrew</a></li>
  <li><a href="https://trivy.dev/" target="_blank">Trivy by Aqua Security</a></li>
  <li><a href="https://semgrep.dev/" target="_blank">Semgrep</a></li>
  <li><a href="https://falco.org/" target="_blank">Falco Runtime Security</a></li>
  <li><a href="https://csrc.nist.gov/publications/detail/sp/800-207/final" target="_blank">NIST SP 800-207 Zero Trust</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
