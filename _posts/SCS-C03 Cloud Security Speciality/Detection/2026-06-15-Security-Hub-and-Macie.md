---
layout: post
title: AWS Security Hub and Amazon Macie — CSPM and Data Classification
date: 2026-06-15T10:00:00
categories:
  - SCS-C03 Cloud Security Speciality
  - Detection
tags:
  - aws
  - security-hub
  - macie
  - cspm
  - cloud-security
  - scs-c03
author: muhammed
description: SCS-C03 Domain 1 — Security Hub finding aggregation, security standards, ASFF, Macie sensitive data discovery, and multi-account setup
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## AWS Security Hub

**Security Hub** is a Cloud Security Posture Management (CSPM) service.
It aggregates security findings from GuardDuty, Inspector, Macie, IAM Access Analyzer, and dozens of third-party tools into a single, normalised view.
It also runs its own **security checks** against AWS best practices and compliance standards.

---

## How Security Hub Works

```
GuardDuty findings ──────┐
Inspector findings ───────┤
Macie findings ───────────┼──► Security Hub ──► EventBridge ──► SNS / Lambda / SIEM
IAM Access Analyzer ─────┤   (normalises to ASFF)
Third-party tools ────────┘
Config rules (optional) ──┘
```

All findings are normalised into **ASFF (Amazon Security Finding Format)** — a standard JSON schema that makes findings comparable across sources.

```bash
# Enable Security Hub
aws securityhub enable-security-hub \
  --enable-default-standards

# Enable in organisation (from delegated admin)
aws securityhub update-organization-configuration \
  --auto-enable \
  --auto-enable-standards NONE

# List enabled standards
aws securityhub get-enabled-standards

# List findings
aws securityhub get-findings \
  --filters '{"SeverityLabel": [{"Value": "CRITICAL", "Comparison": "EQUALS"}]}' \
  --output table
```

> 📸 **SCREENSHOT:** Security Hub → Summary page.
> Show the security score, findings by severity chart, and the enabled security standards with their compliance percentages.

---

## Security Standards

Security Hub evaluates your resources against built-in security standards.
Each standard runs a set of **controls** — individual checks — and scores your compliance percentage.

| Standard | Focus |
|---|---|
| **AWS Foundational Security Best Practices (FSBP)** | AWS-specific best practices across all services |
| **CIS AWS Foundations Benchmark** | Center for Internet Security prescriptive rules |
| **PCI DSS** | Payment Card Industry Data Security Standard |
| **NIST SP 800-53** | US government security framework |
| **SOC 2** | Service Organisation Control 2 |

**Exam tip:** FSBP is the most comprehensive and AWS-native — it covers over 200 controls.
CIS Benchmark is commonly used alongside it for compliance audit evidence.

### Suppressing Controls

Some controls may not apply to your environment (e.g. a sandbox account doesn't need MFA on all users).
Suppress them rather than disabling the standard — suppressed controls don't affect your score but keep the check in place for audit trail purposes.

```bash
# Disable a specific control (suppress from scoring)
aws securityhub update-standards-control \
  --standards-control-arn arn:aws:securityhub:eu-west-1:123456789012:control/cis-aws-foundations-benchmark/v/1.2.0/1.4 \
  --control-status DISABLED \
  --disabled-reason "Root account MFA managed by org policy — not applicable"
```

---

## Finding Workflow

Security Hub findings have a **workflow status** you can use to track investigation:

| Status | Meaning |
|---|---|
| `NEW` | Not yet reviewed |
| `NOTIFIED` | Team notified |
| `INVESTIGATING` | Being investigated |
| `RESOLVED` | Fixed |
| `SUPPRESSED` | Intentionally ignored |

```bash
# Update finding workflow status
aws securityhub batch-update-findings \
  --finding-identifiers '[{"Id": "finding-id", "ProductArn": "arn:aws:securityhub:..."}]' \
  --workflow '{"Status": "RESOLVED"}' \
  --note '{"Text": "Fixed — S3 block public access enabled", "UpdatedBy": "alice"}'
```

---

## Automated Response with EventBridge

```bash
# EventBridge rule: act on CRITICAL Security Hub findings
aws events put-rule \
  --name "security-hub-critical" \
  --event-pattern '{
    "source": ["aws.securityhub"],
    "detail-type": ["Security Hub Findings - Imported"],
    "detail": {
      "findings": {
        "Severity": {"Label": ["CRITICAL"]},
        "Workflow": {"Status": ["NEW"]}
      }
    }
  }'
```

---

## Cross-Account Aggregation

In a multi-account setup, designate a **delegated administrator** account.
Member accounts automatically send findings to the admin account.
Use a **finding aggregator** to pull findings from multiple regions into one region.

```bash
# Create a finding aggregator (centralise all regions into eu-west-1)
aws securityhub create-finding-aggregator \
  --region-linking-mode ALL_REGIONS

# List aggregated regions
aws securityhub get-finding-aggregator \
  --finding-aggregator-arn arn:aws:securityhub:eu-west-1:123456789012:finding-aggregator/abc
```

---

## Amazon Macie

**Amazon Macie** is a data security service that uses machine learning to automatically discover, classify, and protect sensitive data in Amazon S3.
It detects PII (Personally Identifiable Information), financial data, credentials, and other sensitive content — at scale, across all your S3 buckets.

---

## How Macie Works

Macie analyses S3 objects by sampling their content and metadata.
It classifies objects using built-in managed data identifiers and custom patterns you define.

```
S3 Bucket ──► Macie (samples objects) ──► Findings ──► Security Hub / EventBridge
```

---

## Sensitive Data Discovery

### Automated Discovery

Automated sensitive data discovery continuously evaluates all S3 buckets in scope, sampling objects to identify sensitive data.
It produces a **sensitivity score** (0–100) for each bucket.

```bash
# Enable Macie
aws macie2 enable-macie

# Enable automated sensitive data discovery
aws macie2 update-automated-discovery-configuration \
  --status ENABLED
```

### Sensitive Data Discovery Jobs

For targeted analysis, create a classification job on specific buckets or object prefixes.

```bash
aws macie2 create-classification-job \
  --name "scan-customer-data" \
  --job-type ONE_TIME \
  --s3-job-definition '{
    "BucketDefinitions": [{
      "AccountId": "123456789012",
      "Buckets": ["customer-uploads", "user-documents"]
    }]
  }' \
  --managed-data-identifier-selector ALL
```

> 📸 **SCREENSHOT:** Macie → S3 buckets → select a bucket → Sensitive data.
> Show the sensitivity score, the sensitive data types found (e.g. CREDIT_CARD_NUMBER, EMAIL_ADDRESS), and the object count.

---

## Macie Finding Types

| Finding Type | Description |
|---|---|
| `SensitiveData:S3Object/Credentials` | Access keys, passwords, private keys found in an object |
| `SensitiveData:S3Object/Financial` | Credit card numbers, bank account numbers |
| `SensitiveData:S3Object/Personal` | PII — names, addresses, passport numbers, national IDs |
| `SensitiveData:S3Object/Multiple` | Multiple categories of sensitive data in one object |
| `Policy:IAMUser/S3BlockPublicAccessDisabled` | Block Public Access disabled on a bucket |
| `Policy:IAMUser/S3BucketEncryptionDisabled` | Bucket encryption disabled |
| `Policy:IAMUser/S3BucketPublic` | Bucket is publicly accessible |
| `Policy:IAMUser/S3BucketSharedExternally` | Bucket shared with an external account |

---

## Managed Data Identifiers

Macie has built-in detectors for over 100 sensitive data types across multiple countries.

Key categories:
- **Credentials** — AWS access keys, GitHub tokens, private keys
- **Financial** — Credit card numbers (Visa, Mastercard, Amex), IBAN, routing numbers
- **Personal** — Passports, driving licences, SSNs, NHS numbers, national IDs
- **Healthcare** — US HIPAA-regulated health information (PHI)

### Custom Data Identifiers

Define your own regex patterns for organisation-specific sensitive data formats (internal employee IDs, contract numbers, etc.).

```bash
aws macie2 create-custom-data-identifier \
  --name "Employee-ID" \
  --regex "EMP-[0-9]{6}" \
  --description "Internal employee ID format"
```

---

## Multi-Account Setup

Same pattern as GuardDuty — delegated administrator in the security tooling account.

```bash
# Enable Macie org-wide (run from delegated admin)
aws macie2 enable-organization-admin-account \
  --admin-account-id 111122223333

# Auto-enrol member accounts
aws macie2 update-organization-configuration \
  --auto-enable
```

---

## Macie + Security Hub Integration

Enable the integration and all Macie policy findings flow into Security Hub.
Sensitive data findings do NOT flow to Security Hub by default (they contain actual sensitive data fragments — keeping them in Macie protects that data).

---

## Exam Key Points

**Security Hub:**
- Aggregates findings in **ASFF format** — normalises across all sources
- Runs **security checks** against CIS Benchmark, FSBP, PCI DSS etc.
- Use **finding aggregator** to centralise multi-region findings
- **Suppressing** a control is better than disabling a standard
- Security Hub **does not remediate** — it detects and alerts

**Macie:**
- Only works with **S3** — not EBS, EFS, RDS, DynamoDB
- Uses **managed data identifiers** (built-in) and **custom data identifiers** (your regex)
- **Automated discovery** = continuous background sampling
- **Classification jobs** = on-demand targeted scans
- Policy findings → Security Hub. Sensitive data findings → stay in Macie
- Macie findings trigger **EventBridge** for automated response

---

## Quick Reference

```bash
# Security Hub
aws securityhub get-findings --filters '{"WorkflowStatus": [{"Value": "NEW", "Comparison": "EQUALS"}]}'
aws securityhub get-insights
aws securityhub describe-standards
aws securityhub list-members

# Macie
aws macie2 get-macie-session
aws macie2 list-classification-jobs
aws macie2 list-findings
aws macie2 get-findings --finding-ids finding-id-here
aws macie2 list-managed-data-identifiers
aws macie2 get-bucket-statistics
```
