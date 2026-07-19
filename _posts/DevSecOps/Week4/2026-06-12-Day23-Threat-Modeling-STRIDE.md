---
layout: post
title: "Week 4 — Day 23: Threat Modeling with STRIDE"
date: 2026-06-12 10:00:00 +0800
categories:
  - DevSecOps
  - Week4
tags:
  - ThreatModeling
  - STRIDE
  - AppSec
  - CloudSecurity
  - DevSecOps
author: muhammed
description: A full walkthrough of threat modeling using STRIDE — drawing data flow diagrams, identifying threats, rating risk, and applying the methodology to a real cloud-native application.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fprodsens.live%2Fwp-content%2Fuploads%2F2025%2F01%2F31009-understanding-the-stride-function-in-cybersecurity.jpg&f=1&nofb=1&ipt=69e6daa588ee4bb8b3ecaa97f34c53f4e9184ca3e131cb56ca6d89f2dbe6f8ac
---

## What is Threat Modeling?

Threat modeling is structured thinking about what can go wrong in a system — done **before** you build, not after a breach.

Instead of waiting for a pentest or a real attack to discover vulnerabilities, you proactively ask:
- What are we building?
- What can go wrong?
- What are we going to do about it?
- Did we do a good enough job?

The output is a prioritized list of threats and the controls needed to address them.

**When to do it:** During design, when adding significant new features, when the attack surface changes (new API endpoint, new data type, new user type).

---

## STRIDE Threat Categories

STRIDE is a threat classification model developed by Microsoft. Each letter maps to a category of threat:

| Letter | Threat | Violates | Example |
|--------|--------|----------|---------|
| **S** | Spoofing | Authentication | Attacker forges JWT token to impersonate another user |
| **T** | Tampering | Integrity | Attacker modifies a request in transit to change an order amount |
| **R** | Repudiation | Non-repudiation | User denies making a transaction, logs don't prove otherwise |
| **I** | Information Disclosure | Confidentiality | API returns full user object including password hash |
| **D** | Denial of Service | Availability | Attacker floods the API causing it to become unresponsive |
| **E** | Elevation of Privilege | Authorization | Regular user accesses admin endpoints by modifying their JWT role claim |

---

## Step 1 — Draw a Data Flow Diagram (DFD)

A DFD maps how data moves through your system. It uses four elements:

| Symbol | Meaning |
|--------|---------|
| Rectangle | External entity (user, external service) |
| Circle / rounded box | Process (your application component) |
| Two parallel lines | Data store (database, S3 bucket, cache) |
| Arrow | Data flow (HTTP request, DB query, message) |
| Dashed box | Trust boundary (crossing this boundary = higher scrutiny) |

**Example — MindCraft application DFD:**

```
[User Browser]
     |
     | HTTPS request
     ▼
  ─ ─ ─ ─ ─ ─ ─ (Trust Boundary: Internet → AWS) ─ ─ ─ ─ ─
     |
     ▼
[CloudFront + WAF]
     |
     | Filtered request
     ▼
[ALB]
     |
     ▼
  ─ ─ ─ ─ ─ ─ (Trust Boundary: Public → Private subnet) ─ ─
     |
     ▼
[API Service (ECS)]  ──────────────── [Redis Cache]
     |                                     (Data Store)
     | SQL query
     ▼
[RDS PostgreSQL]    ←── [S3 Bucket (file uploads)]
  (Data Store)               (Data Store)
     |
     ▼
[CloudTrail / CloudWatch]
  (Data Store — audit logs)
```

> `[SCREENSHOT]` — *A hand-drawn or tool-drawn DFD on paper or in draw.io/Miro showing the MindCraft application components, data flows, and trust boundaries clearly labeled*

**Tools for drawing DFDs:**
- Draw.io (free, web-based)
- Microsoft Threat Modeling Tool (free, Windows)
- OWASP Threat Dragon (open source)
- Miro or Lucidchart (collaborative)

---

## Step 2 — Apply STRIDE to Each Element

For every data flow, process, and data store — ask which STRIDE threats apply.

### Applying STRIDE to the MindCraft App

#### Data Flow: User → CloudFront

| Threat | Question | Finding |
|--------|----------|---------|
| **S** Spoofing | Can a user forge another user's identity? | JWT tokens — are they validated server-side? |
| **T** Tampering | Can a request be modified in transit? | HTTPS enforced? HTTP redirected? |
| **R** Repudiation | Can users deny their actions? | Are user actions logged with identity? |
| **I** Info Disclosure | Does the response leak sensitive data? | API error messages — do they expose stack traces? |
| **D** DoS | Can this flow be flooded? | WAF rate limiting in place? |
| **E** Elevation | Can a user access resources they shouldn't? | Authorization checks on every API endpoint? |

#### Process: API Service (ECS)

| Threat | Finding | Control |
|--------|---------|---------|
| **S** Spoofing | Service calls other internal services — are they authenticated? | mTLS or IAM auth between services |
| **T** Tampering | Can environment variables or config be modified by an attacker? | Read-only filesystem, no SSRF to metadata service |
| **R** Repudiation | Are all business operations logged with user ID and timestamp? | Structured logging with user context |
| **I** Info Disclosure | Does the service log sensitive data (passwords, PII)? | Log scrubbing for PII |
| **D** DoS | Can a single user's request trigger expensive DB queries? | Query timeouts, rate limiting per user |
| **E** Elevation | Can a regular user call admin endpoints? | RBAC enforced, roles validated server-side |

#### Data Store: RDS PostgreSQL

| Threat | Finding | Control |
|--------|---------|---------|
| **S** Spoofing | Who can connect to the DB? | Only app tier SG, IAM DB auth |
| **T** Tampering | Can data be modified outside the application? | No direct internet access, audit logging |
| **R** Repudiation | Are DB writes auditable? | RDS audit logs enabled |
| **I** Info Disclosure | Is data encrypted at rest and in transit? | RDS encryption, SSL connections enforced |
| **D** DoS | Can the DB be overwhelmed? | Connection pooling, RDS instance sizing |
| **E** Elevation | Can the app user perform DDL operations? | App DB user has DML only (SELECT/INSERT/UPDATE/DELETE) |

---

## Step 3 — Rate Each Threat (DREAD)

After identifying threats, prioritize them using DREAD scoring:

| Factor | Score 1-3 | Meaning |
|--------|-----------|---------|
| **D** Damage | 3 = Critical data loss or full compromise |
| **R** Reproducibility | 3 = Reproducible every time |
| **E** Exploitability | 3 = No skill required, tools available |
| **A** Affected users | 3 = All users affected |
| **D** Discoverability | 3 = Publicly known, easy to find |

**Example rating — JWT token not validated server-side:**

| Factor | Score | Reason |
|--------|-------|--------|
| Damage | 3 | Full account takeover |
| Reproducibility | 3 | Consistent exploit |
| Exploitability | 2 | Requires JWT knowledge |
| Affected users | 3 | Every user |
| Discoverability | 2 | Requires probing |
| **Total** | **13/15** | **Critical** |

**Example rating — verbose error messages in API:**

| Factor | Score | Reason |
|--------|-------|--------|
| Damage | 1 | Info disclosure only |
| Reproducibility | 3 | Always present |
| Exploitability | 1 | Just read the response |
| Affected users | 1 | Attacker only sees their own errors |
| Discoverability | 3 | Trivially found |
| **Total** | **9/15** | **Medium** |

---

## Step 4 — Define Mitigations

For each identified threat, define a specific control:

| Threat | STRIDE | DREAD | Mitigation |
|--------|--------|-------|-----------|
| JWT not validated | Spoofing | 13 | Validate signature + expiry + claims server-side on every request |
| SQL injection via API params | Tampering | 12 | Use parameterized queries, run Semgrep in pipeline |
| No audit logging for payments | Repudiation | 10 | Log all payment events with user ID, timestamp, amount |
| Stack trace in API errors | Info Disclosure | 9 | Generic error messages in prod, structured errors in logs only |
| No rate limiting on login | DoS / Brute force | 11 | Rate-based WAF rule + account lockout after N failures |
| User can set own role in JWT | Elevation | 14 | Never trust client-provided role claims — look up role from DB |

---

## Step 5 — Threat Model for the MindCraft App

Let's apply this to MindCraft specifically.

**Architecture:** React frontend → API Gateway → ECS backend → RDS + S3 + ElastiCache → deployed on AWS via Terraform

**Trust boundaries identified:**
1. Internet → CloudFront/WAF
2. Public subnet (ALB) → Private subnet (ECS)
3. ECS → RDS (DB subnet)
4. ECS → S3/ElastiCache/Secrets Manager (via VPC endpoints)

**Top 5 threats identified:**

| # | Threat | Category | Priority | Mitigation |
|---|--------|----------|----------|-----------|
| 1 | IDOR — user accesses another user's resources by guessing IDs | Elevation | Critical | Enforce ownership check on every resource lookup |
| 2 | Insecure file upload — user uploads malicious file to S3 | Tampering | High | Validate file type + size, scan with Lambda post-upload, serve via CloudFront not S3 direct |
| 3 | Secrets in environment variables | Info Disclosure | High | Move all secrets to Secrets Manager, use ECS secrets injection |
| 4 | ECS task with overly broad IAM role | Elevation | High | Least privilege IAM role scoped to specific S3 bucket and secret ARN only |
| 5 | No rate limiting on password reset endpoint | DoS / Enumeration | Medium | WAF rate rule + CAPTCHA on password reset |

> `[SCREENSHOT]` — *A completed threat model table in a spreadsheet or Notion showing the threat name, STRIDE category, DREAD score, and mitigation for the MindCraft application — at least 8-10 rows*

---

## Threat Modeling Tools

### OWASP Threat Dragon

Free, open-source web-based tool:
```bash
docker run -it -p 3000:3000 owasp/threat-dragon
# Open http://localhost:3000
```

> `[SCREENSHOT]` — *OWASP Threat Dragon UI in browser showing a DFD diagram being built with the threat list panel on the right showing identified STRIDE threats for a selected component*

### Microsoft Threat Modeling Tool

Free Windows tool with automatic STRIDE threat generation from DFDs:
- Draw components using the templates
- The tool automatically generates STRIDE threats for each element
- Export to Word/Excel for documentation

---

## Lab — Threat Model Your Blog Infrastructure

**Objective:** Apply STRIDE to the mhdomer.github.io Jekyll blog + GitHub Actions pipeline.

**Components to model:**
- GitHub repo (source code + posts)
- GitHub Actions (CI/CD pipeline)
- GitHub Pages (hosting)

**Draw the DFD:**
1. You (author) → GitHub repo
2. GitHub Actions → builds Jekyll site
3. GitHub Pages → serves to readers
4. Reader browser → GitHub Pages

**Apply STRIDE to each flow and component:**

| Component | Threat | Finding |
|-----------|--------|---------|
| GitHub repo | S | Can someone push to main without review? — branch protection enabled? |
| GitHub repo | T | Can pipeline files be modified to exfiltrate secrets? — workflow permissions scoped? |
| GitHub Actions | I | Are secrets printed in logs? — use masked secrets only |
| GitHub Actions | E | Can a PR modify a workflow and run it with elevated permissions? — require approval for fork PRs |
| GitHub Pages | D | Can someone take down the site? — GitHub Pages uptime dependent on GitHub |
| GitHub Pages | I | Is any sensitive info in the published posts? — review before publishing |

> `[SCREENSHOT]` — *A simple DFD drawn for the blog pipeline showing the four components with data flow arrows and trust boundaries, annotated with at least 3 STRIDE threats*

---

## Key Takeaways

- Threat modeling is the only proactive security activity — everything else is reactive
- STRIDE gives you a systematic way to not miss threat categories — work through each letter for each component
- The DFD is the foundation — if the diagram is wrong, the threat model is wrong
- Prioritize with DREAD — not all threats are equal; fix Critical ones first
- A threat model is a living document — update it when the architecture changes
- Even a simple threat model (30 minutes on a whiteboard) catches more than no threat model

---

## References

<div class="references">
<ul>
  <li><a href="https://owasp.org/www-community/Threat_Modeling" target="_blank">OWASP Threat Modeling</a></li>
  <li><a href="https://github.com/OWASP/threat-dragon" target="_blank">OWASP Threat Dragon</a></li>
  <li><a href="https://learn.microsoft.com/en-us/azure/security/develop/threat-modeling-tool" target="_blank">Microsoft Threat Modeling Tool</a></li>
  <li><a href="https://shostack.org/books/threat-modeling-book" target="_blank">Threat Modeling — Designing for Security (Adam Shostack)</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
