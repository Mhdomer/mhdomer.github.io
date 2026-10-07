---
layout: post
title: "DevSecOps Labs: Index & Practical Roadmap"
date: 2026-06-14T10:00:00+03:00
categories:
  - DevSecOps
  - Labs
tags:
  - DevSecOps
  - Labs
  - CloudSecurity
  - HandsOn
author: muhammed
description: Practical hands-on lab index mapping real-world cloud security and DevSecOps exercises to each week of the 4-week study plan. Includes lab objectives, tools, success criteria, and direct links to full walkthroughs.
toc: true
pin: false
math: false
mermaid: true
image: https://veritis.com/wp-content/uploads/2022/06/all-you-need-to-know-about-devsecops-and-its-implementation.jpg
permalink: /posts/DevSecOps-Labs-Index/
---

## Overview

When I started my DevSecOps journey, I realized quickly that reading theory and passing multiple-choice quizzes is only 20% of the game. Real confidence comes from building the architecture, breaking it intentionally, and fixing it in code.

This page serves as the master index for all the hands-on security labs I built throughout the 4-week [Cloud Security & DevSecOps Study Plan]({% post_url DevSecOps/2026-05-19-Cloud-Security-DevSecOps-Study-Plan %}).

Every lab below is structured around a clear workflow:
1. **Objective:** What specific security problem we are solving.
2. **Tools & Stack:** The exact services and open-source utilities used.
3. **Success Criteria:** The verifiable proof that the fix or control works.
4. **Walkthrough Link:** Direct link to the complete step-by-step documentation.

---

## Week 1: AWS Security Services & Identity

### Lab 1.1: IAM Least Privilege Role & Policy Simulator
- **Objective:** Create a role with minimal S3 access and verify no unintended permissions exist.
- **Tools:** AWS IAM, Policy Simulator, AWS CLI
- **Success Criteria:** `s3:GetObject` allowed, `s3:DeleteObject` denied, `ec2:*` denied in simulator.
- **Walkthrough:** [Day 1: Demystifying AWS IAM - Roles, Policies, and Least Privilege]({% post_url DevSecOps/Week1/2026-03-01-Day1-AWS-IAM %})

---

### Lab 1.2: Trigger and Investigate a GuardDuty Finding
- **Objective:** Generate real threat detections, trace findings into Security Hub, and automate alerts.
- **Tools:** AWS GuardDuty, Security Hub, Amazon EventBridge
- **Success Criteria:** Sample findings populate Security Hub with severity ratings, EventBridge rule triggers on High severity finding.
- **Walkthrough:** [Day 4: AWS GuardDuty & Security Hub - Threat Detection & Centralized Posture]({% post_url DevSecOps/Week1/2026-03-04-Day4-GuardDuty-Security-Hub %})

---

### Lab 1.3: AWS Config Rule & Auto-Remediation for S3
- **Objective:** Build an AWS Config rule that detects publicly accessible S3 buckets and automatically reverts them.
- **Tools:** AWS Config, S3, SSM Automation Documents
- **Success Criteria:** Public bucket triggers a NON_COMPLIANT status within minutes and auto-remediates.
- **Walkthrough:** [Day 3: AWS CloudTrail & AWS Config - Forensic Auditing & Compliance Posture]({% post_url DevSecOps/Week1/2026-03-03-Day3-CloudTrail-AWS-Config %})

---

### Lab 1.4: CloudTrail Forensics with Athena
- **Objective:** Log all management events and run SQL queries in Athena to hunt suspicious API calls.
- **Tools:** AWS CloudTrail, Amazon Athena, S3
- **Success Criteria:** Query isolates the exact assumed role session, source IP, and parameters of simulated malicious actions.
- **Walkthrough:** [Day 3: AWS CloudTrail & AWS Config - Forensic Auditing & Compliance Posture]({% post_url DevSecOps/Week1/2026-03-03-Day3-CloudTrail-AWS-Config %})

---

### Lab 1.5: WAF Setup with OWASP Core Rules on an ALB
- **Objective:** Attach AWS WAF Web ACL to an Application Load Balancer and block SQL injection and cross-site scripting attempts.
- **Tools:** AWS WAF v2, ALB, curl
- **Success Criteria:** Clean HTTP GET requests return 200 OK, payloads containing `' OR 1=1 --` trigger immediate 403 Forbidden responses.
- **Walkthrough:** [Day 6: AWS WAF & Shield - Protecting Web Apps from Layer 7 Attacks]({% post_url DevSecOps/Week1/2026-03-06-Day6-WAF-Shield %})

---

## Week 2: Secrets Management & Container Security

### Lab 2.1: Secrets Manager Automatic Rotation
- **Objective:** Centralize database credentials and execute automatic rotation using Lambda.
- **Tools:** AWS Secrets Manager, RDS, Lambda, Python boto3
- **Success Criteria:** Application pulls credentials via SDK without hardcoding, rotation lambda updates DB credentials seamlessly.
- **Walkthrough:** [Day 8: AWS Secrets Manager & Systems Manager Parameter Store - Dynamic Secrets]({% post_url DevSecOps/Week2/2026-03-08-Day8-Secrets-Manager-Parameter-Store %})

---

### Lab 2.2: HashiCorp Vault Dynamic AWS Credentials
- **Objective:** Configure Vault to generate short-lived, leased IAM credentials on demand.
- **Tools:** HashiCorp Vault, AWS IAM
- **Success Criteria:** `vault read aws/creds/s3-reader` issues temporary keys that automatically expire and self-destruct upon lease expiry.
- **Walkthrough:** [Day 9: HashiCorp Vault Basics]({% post_url DevSecOps/Week2/2026-03-09-Day9-HashiCorp-Vault %})

---

### Lab 2.3: Dockerfile Hardening Before and After
- **Objective:** Refactor an insecure root-based Dockerfile into a minimal, non-root, read-only container image.
- **Tools:** Docker, Docker Bench for Security
- **Success Criteria:** Container runs as unprivileged UID 10001, root filesystem mounted read-only, image size reduced by over 80%.
- **Walkthrough:** [Day 10: Docker Security Hardening]({% post_url DevSecOps/Week2/2026-03-10-Day10-Docker-Security-Hardening %})

---

### Lab 2.4: Container Image Scanning with Trivy
- **Objective:** Scan container images in CI to filter out Critical and High CVEs before deployment.
- **Tools:** Trivy, Docker, GitHub Actions
- **Success Criteria:** Scans return structured tables of CVEs, pipeline exits with error code 1 when Critical vulnerabilities are detected.
- **Walkthrough:** [Day 11: Container Image Scanning with Trivy]({% post_url DevSecOps/Week2/2026-03-11-Day11-Trivy-Container-Scanning %})

---

### Lab 2.5: Kubernetes RBAC & Network Isolation
- **Objective:** Implement dedicated ServiceAccounts and restrictive NetworkPolicies to stop lateral movement.
- **Tools:** kubectl, Kubernetes (K3s/Minikube), NetworkPolicy manifests
- **Success Criteria:** Pods cannot access unauthorized namespaces; default cluster-admin service account tokens are disabled.
- **Walkthrough:** [Day 12: Kubernetes RBAC & Pod Security]({% post_url DevSecOps/Week2/2026-03-12-Day12-Kubernetes-RBAC-Pod-Security %})

---

### Lab 2.6: Runtime Threat Detection with Falco
- **Objective:** Monitor Linux kernel syscalls inside containers and trigger alerts on shell execution or sensitive file reads.
- **Tools:** Falco, Helm, Kubernetes
- **Success Criteria:** Spawning `/bin/bash` inside a running pod immediately generates a Notice alert in Falco logs.
- **Walkthrough:** [Day 13: Runtime Security with Falco]({% post_url DevSecOps/Week2/2026-03-13-Day13-Falco-Runtime-Security %})

---

## Week 3: Pipeline Security & Automated Testing

### Lab 3.1: SAST Scanning with Semgrep
- **Objective:** Audit source code on pull requests using open-source Semgrep rules and custom regex patterns.
- **Tools:** Semgrep CLI, GitHub Actions
- **Success Criteria:** Vulnerable code patterns (like unparameterized SQL queries) fail automated CI checks with actionable guidance.
- **Walkthrough:** [Day 15: SAST with Semgrep]({% post_url DevSecOps/Week3/2026-06-03-Day15-SAST-Semgrep %})

---

### Lab 3.2: Dependency Vulnerability Scanning with Snyk
- **Objective:** Detect outdated and vulnerable third-party packages in `package.json` and `requirements.txt`.
- **Tools:** Snyk CLI, Dependabot
- **Success Criteria:** High-severity dependency CVEs flagged, automated PRs generated for patched minor versions.
- **Walkthrough:** [Day 16: SCA with Snyk & Dependabot]({% post_url DevSecOps/Week3/2026-06-04-Day16-SCA-Snyk-Dependabot %})

---

### Lab 3.3: Infrastructure as Code (IaC) Scanning with Checkov
- **Objective:** Scan Terraform plans and HCL code for misconfigurations before infrastructure is provisioned.
- **Tools:** Checkov, tfsec, Terraform
- **Success Criteria:** Flagged violations (like open ingress `0.0.0.0/0` on port 22 or unencrypted EBS volumes) block the merge.
- **Walkthrough:** [Day 17: IaC Scanning with Checkov & tfsec]({% post_url DevSecOps/Week3/2026-06-05-Day17-IaC-Scanning-Checkov-tfsec %})

---

### Lab 3.4: DAST Scanning with OWASP ZAP
- **Objective:** Perform automated dynamic application security testing against running web application staging endpoints.
- **Tools:** OWASP ZAP (Docker), curl
- **Success Criteria:** Identifies missing security headers (CSP, HSTS), weak cookie flags, and exposed administrative endpoints.
- **Walkthrough:** [Day 18: DAST with OWASP ZAP]({% post_url DevSecOps/Week3/2026-06-06-Day18-DAST-OWASP-ZAP %})

---

### Lab 3.5: Full Secure CI/CD Pipeline
- **Objective:** Integrate SAST, SCA, IaC scanning, image linting, and branch protection into a unified GitHub Actions workflow.
- **Tools:** GitHub Actions, Semgrep, Snyk, Checkov, Trivy
- **Success Criteria:** All automated checks run concurrently on PRs; non-compliant builds fail before reaching the main branch.
- **Walkthrough:** [Day 19: Building a Full Secure CI/CD Pipeline]({% post_url DevSecOps/Week3/2026-06-07-Day19-Full-Secure-Pipeline %})

---

## Week 4: Cloud Incident Response, Threat Modeling & Posture

### Lab 4.1: Simulated Incident Response on Compromised IAM Keys
- **Objective:** Walk through the complete 4-phase incident response cycle when an AWS access key is leaked.
- **Tools:** AWS CLI, CloudTrail, Athena, IAM
- **Success Criteria:** Deactivate compromised key, revoke active STS sessions, isolate affected resources, and extract blast radius logs.
- **Walkthrough:** [Day 22: Cloud Incident Response]({% post_url DevSecOps/Week4/2026-06-11-Day22-Cloud-Incident-Response %})

---

### Lab 4.2: Threat Modeling with STRIDE
- **Objective:** Deconstruct an application architecture, draw data flow diagrams, and map threats against STRIDE categories.
- **Tools:** STRIDE Framework, Data Flow Diagrams
- **Success Criteria:** Detailed threat breakdown covering Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, and Elevation of Privilege.
- **Walkthrough:** [Day 23: Threat Modeling with STRIDE]({% post_url DevSecOps/Week4/2026-06-12-Day23-Threat-Modeling-STRIDE %})

---

### Lab 4.3: CIS Benchmark Remediation Sprint
- **Objective:** Audit an AWS account against the CIS AWS Foundations Benchmark v1.4 and remediate failed controls.
- **Tools:** AWS Security Hub, AWS CLI, Terraform
- **Success Criteria:** Remediate account baseline controls (enforce MFA on root, rotate stale keys, enable CloudTrail multi-region).
- **Walkthrough:** [Day 24: CIS Benchmarks & Compliance Basics]({% post_url DevSecOps/Week4/2026-06-13-Day24-CIS-Benchmarks-Compliance %})

---

### Lab 4.4: Zero Trust Network Micro-Segmentation
- **Objective:** Replace flat subnet security group rules with explicit security group-to-security group references.
- **Tools:** AWS VPC, Security Groups, EC2
- **Success Criteria:** Web tier can only reach the app tier on designated ports; lateral movement from web directly to database is strictly blocked.
- **Walkthrough:** [Day 21: Zero Trust Architecture]({% post_url DevSecOps/Week4/2026-06-10-Day21-Zero-Trust-Architecture %})

---

### Lab 4.5: Multi-Account Guardrails with SCPs
- **Objective:** Write and attach Service Control Policies across an AWS Organization to prevent disabling security services.
- **Tools:** AWS Organizations, Service Control Policies (SCPs)
- **Success Criteria:** Local account administrators cannot disable CloudTrail or delete security log buckets, even with full `AdministratorAccess`.
- **Walkthrough:** [Day 2: SCPs & Permission Boundaries]({% post_url DevSecOps/Week1/2026-03-02-Day2-SCPs-Permission-Boundaries %})

---

## Progress & Completion Matrix

| Lab # | Topic / Exercise | Domain | Status | Full Walkthrough |
|:---:|---|---|:---:|:---:|
| **1.1** | IAM Least Privilege & Policy Simulator | Identity | Completed | [View Guide]({% post_url DevSecOps/Week1/2026-03-01-Day1-AWS-IAM %}) |
| **1.2** | GuardDuty Threat Finding & EventBridge | Detection | Completed | [View Guide]({% post_url DevSecOps/Week1/2026-03-04-Day4-GuardDuty-Security-Hub %}) |
| **1.3** | AWS Config Rule & Auto-Remediation | Compliance | Completed | [View Guide]({% post_url DevSecOps/Week1/2026-03-03-Day3-CloudTrail-AWS-Config %}) |
| **1.4** | CloudTrail Forensic Analysis with Athena | Forensics | Completed | [View Guide]({% post_url DevSecOps/Week1/2026-03-03-Day3-CloudTrail-AWS-Config %}) |
| **1.5** | AWS WAF & OWASP Managed Rules on ALB | Perimeter | Completed | [View Guide]({% post_url DevSecOps/Week1/2026-03-06-Day6-WAF-Shield %}) |
| **2.1** | Secrets Manager & Lambda Rotation | Secrets | Completed | [View Guide]({% post_url DevSecOps/Week2/2026-03-08-Day8-Secrets-Manager-Parameter-Store %}) |
| **2.2** | HashiCorp Vault Dynamic AWS IAM Credentials | Secrets | Completed | [View Guide]({% post_url DevSecOps/Week2/2026-03-09-Day9-HashiCorp-Vault %}) |
| **2.3** | Dockerfile Hardening & Docker Bench | Containers | Completed | [View Guide]({% post_url DevSecOps/Week2/2026-03-10-Day10-Docker-Security-Hardening %}) |
| **2.4** | Trivy Container CVE Scanning in CI | Containers | Completed | [View Guide]({% post_url DevSecOps/Week2/2026-03-11-Day11-Trivy-Container-Scanning %}) |
| **2.5** | Kubernetes RBAC & Pod Security Isolation | Kubernetes | Completed | [View Guide]({% post_url DevSecOps/Week2/2026-03-12-Day12-Kubernetes-RBAC-Pod-Security %}) |
| **2.6** | Falco Kernel Syscall Threat Detection | Runtime | Completed | [View Guide]({% post_url DevSecOps/Week2/2026-03-13-Day13-Falco-Runtime-Security %}) |
| **3.1** | Semgrep SAST Static Code Analysis | Pipeline | Completed | [View Guide]({% post_url DevSecOps/Week3/2026-06-03-Day15-SAST-Semgrep %}) |
| **3.2** | Snyk & Dependabot SCA Dependency Scans | Pipeline | Completed | [View Guide]({% post_url DevSecOps/Week3/2026-06-04-Day16-SCA-Snyk-Dependabot %}) |
| **3.3** | Checkov & tfsec IaC Static Security Audits | Pipeline | Completed | [View Guide]({% post_url DevSecOps/Week3/2026-06-05-Day17-IaC-Scanning-Checkov-tfsec %}) |
| **3.4** | OWASP ZAP Dynamic Application Security | Pipeline | Completed | [View Guide]({% post_url DevSecOps/Week3/2026-06-06-Day18-DAST-OWASP-ZAP %}) |
| **3.5** | Full Secure GitHub Actions CI/CD Pipeline | Pipeline | Completed | [View Guide]({% post_url DevSecOps/Week3/2026-06-07-Day19-Full-Secure-Pipeline %}) |
| **4.1** | Compromised IAM Key Containment & IR | Incident Response | Completed | [View Guide]({% post_url DevSecOps/Week4/2026-06-11-Day22-Cloud-Incident-Response %}) |
| **4.2** | STRIDE Threat Model Decomposition | Modeling | Completed | [View Guide]({% post_url DevSecOps/Week4/2026-06-12-Day23-Threat-Modeling-STRIDE %}) |
| **4.3** | CIS AWS Foundations Benchmark Remediation | Compliance | Completed | [View Guide]({% post_url DevSecOps/Week4/2026-06-13-Day24-CIS-Benchmarks-Compliance %}) |
| **4.4** | Zero Trust Network Micro-Segmentation | Architecture | Completed | [View Guide]({% post_url DevSecOps/Week4/2026-06-10-Day21-Zero-Trust-Architecture %}) |
| **4.5** | Service Control Policies (SCPs) Guardrails | Governance | Completed | [View Guide]({% post_url DevSecOps/Week1/2026-03-02-Day2-SCPs-Permission-Boundaries %}) |

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
