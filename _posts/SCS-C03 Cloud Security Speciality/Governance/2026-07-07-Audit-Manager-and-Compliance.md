---
layout: post
title: AWS Audit Manager, Artifact, and Compliance — Evidence and Assurance
date: 2026-07-07T10:00:00
categories:
  - SCS-C03 Cloud Security Speciality
  - Governance
tags:
  - aws
  - audit-manager
  - artifact
  - compliance
  - config
  - governance
  - cloud-security
  - scs-c03
author: muhammed
description: SCS-C03 Domain 6 — Audit Manager automated evidence collection, AWS Artifact compliance reports, Config conformance packs, Security Hub CSPM, and Well-Architected Tool
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## Compliance Posture Management Overview

The SCS-C03 exam distinguishes between three related compliance capabilities:

| Service | Purpose |
|---|---|
| **AWS Config + conformance packs** | Continuous compliance checking — are resources configured correctly right now? |
| **Security Hub** | Cloud Security Posture Management (CSPM) — aggregate findings, standards scoring, cross-account |
| **AWS Audit Manager** | Audit evidence collection — continuously collect evidence mapped to compliance frameworks |
| **AWS Artifact** | On-demand access to AWS compliance reports (SOC, PCI, ISO, etc.) |
| **Well-Architected Tool** | Self-assessment against AWS best practices and security pillars |

---

## AWS Audit Manager

**AWS Audit Manager** automates the collection of evidence for compliance audits.
Instead of manually gathering screenshots, logs, and config snapshots before an audit, Audit Manager continuously collects and organises evidence as resources are created and used.

### Key Concepts

| Term | Meaning |
|---|---|
| **Framework** | A set of controls mapped to a compliance standard (SOC 2, PCI DSS, HIPAA, GDPR, CIS) |
| **Control** | A specific requirement (e.g. "MFA is enabled for all IAM users") |
| **Assessment** | An active evaluation run against a framework for specific AWS accounts and services |
| **Evidence** | The data collected to prove a control is met — Config snapshots, CloudTrail events, AWS API calls |
| **Evidence folder** | Evidence organised by control within an assessment |

### Evidence Sources

Audit Manager collects evidence from three sources:

| Source | What it collects |
|---|---|
| **AWS Config rules** | Configuration state — resource settings at a point in time |
| **CloudTrail** | API call history — who did what |
| **AWS Security Hub** | Security findings from multiple services |
| **Manual** | PDF/screenshots uploaded manually for controls that cannot be automated |

```bash
# Get available pre-built frameworks
aws auditmanager list-assessment-frameworks \
  --framework-type Standard

# Create an assessment (SOC 2 example)
aws auditmanager create-assessment \
  --name SOC2-2026-Q3 \
  --assessment-reports-destination DestinationType=S3,Destination=s3://my-audit-reports \
  --scope '{
    "awsAccounts": [
      {"id": "111122223333", "emailAddress": "team@company.com", "name": "prod-account"}
    ],
    "awsServices": [
      {"serviceName": "S3"},
      {"serviceName": "EC2"},
      {"serviceName": "IAM"},
      {"serviceName": "KMS"}
    ]
  }' \
  --roles '[{
    "roleArn": "arn:aws:iam::123456789012:role/audit-manager-role",
    "roleType": "PROCESS_OWNER"
  }]' \
  --framework-id framework-soc2-id-here

# Get assessment status
aws auditmanager get-assessment \
  --assessment-id assessment-id-here

# List evidence for a specific control
aws auditmanager get-evidence-by-evidence-folder \
  --assessment-id assessment-id-here \
  --control-set-id control-set-id \
  --evidence-folder-id folder-id

# Generate an assessment report (for auditors)
aws auditmanager create-assessment-report \
  --name SOC2-Q3-2026-Report \
  --description "Quarterly SOC 2 evidence package" \
  --assessment-id assessment-id-here
```

### Custom Frameworks

You can create custom frameworks for internal controls not covered by AWS pre-built frameworks.

```bash
# Create a custom control (check that CloudTrail is enabled)
aws auditmanager create-control \
  --name cloudtrail-enabled \
  --control-mapping-sources '[{
    "sourceName": "cloudtrail-enabled-config-rule",
    "sourceSetUpOption": "Procedural_Controls_Mapping",
    "sourceType": "AWS_Config",
    "sourceKeyword": {
      "keywordInputType": "SELECT_FROM_LIST",
      "keywordValue": "CLOUD_TRAIL_ENABLED"
    }
  }]'
```

---

## AWS Artifact

**AWS Artifact** provides on-demand access to AWS compliance documentation — audit reports, certifications, and agreements.
Use Artifact when:
- You need to share AWS's SOC 2 or ISO 27001 reports with your auditors to demonstrate the underlying infrastructure is certified
- You need to sign compliance agreements (HIPAA BAA, GDPR Data Processing Addendum)

### Types of Documents in Artifact

| Type | Examples |
|---|---|
| **Reports** | SOC 1, SOC 2, SOC 3, ISO 27001, PCI DSS AOC, FedRAMP |
| **Agreements** | HIPAA Business Associate Addendum (BAA), GDPR DPA |

```bash
# List available agreements
aws artifact list-agreements \
  --output table

# Get agreement details (e.g. HIPAA BAA)
aws artifact get-agreement-terms \
  --agreement-type HIPAA_BAA

# Accept an agreement (e.g. to enable HIPAA-covered services)
# Done via console: Artifact → Agreements → Accept
```

**Exam tip:** AWS Artifact does not evaluate your own environment — it provides AWS's compliance documentation.
For evaluating your own environment, use Audit Manager or Config conformance packs.

---

## AWS Config Conformance Packs — Compliance at Scale

A **conformance pack** is a collection of Config rules and remediation actions packaged together.
AWS provides pre-built conformance packs aligned to compliance frameworks.

```bash
# List available sample conformance pack templates
aws configservice list-conformance-pack-compliance-scores \
  --conformance-pack-names operational-best-practices-for-pci-dss

# Deploy a conformance pack from a sample template
aws configservice put-conformance-pack \
  --conformance-pack-name pci-dss-conformance-pack \
  --template-s3-uri s3://aws-conformance-packs-us-east-1/Operational-Best-Practices-for-PCI-DSS.yaml \
  --delivery-s3-bucket my-config-bucket

# Get compliance status
aws configservice describe-conformance-pack-compliance \
  --conformance-pack-name pci-dss-conformance-pack

# Get non-compliant resources across the pack
aws configservice get-conformance-pack-compliance-details \
  --conformance-pack-name pci-dss-conformance-pack \
  --filters ComplianceType=NON_COMPLIANT
```

### AWS Config Aggregator — Cross-Account Visibility

An **aggregator** collects Config data from all accounts in your organization into a single view.

```bash
# Create an org-wide aggregator
aws configservice put-configuration-aggregator \
  --configuration-aggregator-name org-aggregator \
  --organization-aggregation-source '{
    "RoleArn": "arn:aws:iam::123456789012:role/ConfigAggregatorRole",
    "AllAwsRegions": true
  }'

# Query resource state across all accounts
aws configservice select-aggregate-resource-config \
  --configuration-aggregator-name org-aggregator \
  --expression "SELECT accountId, resourceType, resourceId, configuration WHERE configuration.publiclyAccessible = 'true'"
```

---

## Security Hub — CSPM and Standards

**Security Hub** aggregates findings and scores your environment against security standards.
The standards map to compliance frameworks and give an overall **security score** (0–100%) for each.

### Security Hub Standards

| Standard | Controls | Use When |
|---|---|---|
| **AWS Foundational Security Best Practices (FSBP)** | 300+ | Baseline for all AWS accounts |
| **CIS AWS Foundations Benchmark** | Level 1 and Level 2 | CIS hardening requirements |
| **PCI DSS** | Cardholder data environment | Payment card processing |
| **NIST SP 800-53 Rev 5** | NIST controls | US government / FedRAMP-aligned |

```bash
# Enable Security Hub and all standards
aws securityhub enable-security-hub \
  --enable-default-standards

# Check standards compliance
aws securityhub get-standards-control-associations \
  --security-control-id EC2.2  # check status of "VPCs should have default security group closed" control

# Disable a control that doesn't apply (suppress false positives)
aws securityhub update-standards-control \
  --standards-control-arn arn:aws:securityhub:eu-west-1:123456789012:control/cis-aws-foundations-benchmark/v/1.2.0/1.20 \
  --control-status DISABLED \
  --disabled-reason "CloudShell access is restricted via SCP — not applicable"
```

---

## Well-Architected Tool

The **Well-Architected Framework** tool guides you through a structured self-assessment of your workload against six pillars — operational excellence, security, reliability, performance efficiency, cost optimization, and sustainability.

For SCS-C03, the **Security Pillar** is most relevant.
Security pillar focus areas:
- Identity and access management
- Detection
- Infrastructure protection
- Data protection
- Incident response

```bash
# Create a workload in the tool
aws wellarchitected create-workload \
  --workload-name prod-ecommerce \
  --description "Production e-commerce platform" \
  --environment PRODUCTION \
  --aws-regions eu-west-1 us-east-1 \
  --lenses '["wellarchitected", "serverless"]' \
  --account-ids '["123456789012"]'

# List milestones (saved assessments over time)
aws wellarchitected list-milestones \
  --workload-id workload-id-here \
  --lens-alias wellarchitected

# Get improvement plan for the security lens
aws wellarchitected get-lens-review \
  --workload-id workload-id-here \
  --lens-alias wellarchitected \
  --milestone-number 1
```

---

## Compliance Mapping Summary

| When the exam mentions... | Use this service |
|---|---|
| "Prove we are compliant with SOC 2 to our auditors" | AWS Artifact (AWS's own reports) + Audit Manager (your evidence) |
| "Continuously check if resources are misconfigured against PCI DSS" | Config conformance pack |
| "Aggregate security findings across 50 accounts" | Security Hub finding aggregator |
| "Which Config rules are non-compliant across the entire org?" | Config aggregator |
| "Sign the HIPAA BAA" | AWS Artifact → Agreements |
| "Review our workload against security best practices" | Well-Architected Tool — Security Pillar |
| "Automatically collect evidence for the quarterly audit" | Audit Manager |

---

## Exam Key Points

- **Audit Manager**: evidence collection for audits — continuous, automated, mapped to frameworks; generates reports for auditors
- **AWS Artifact**: AWS's own compliance reports (SOC, PCI, ISO) — not your environment's compliance
- **Config conformance packs**: batch deployment of Config rules aligned to a compliance standard; cross-account via org
- **Security Hub standards**: FSBP is the default; CIS, PCI DSS, NIST available — disable controls that don't apply with a reason
- **Config aggregator**: org-wide resource state query; use `select-aggregate-resource-config` for ad-hoc queries
- **Well-Architected Tool**: structured self-assessment, not automated scanning — generates improvement plan

---

## Quick Reference

```bash
# Audit Manager
aws auditmanager list-assessments
aws auditmanager get-assessment --assessment-id ID
aws auditmanager list-assessment-frameworks --framework-type Standard

# Config
aws configservice describe-conformance-packs
aws configservice describe-conformance-pack-compliance --conformance-pack-name NAME
aws configservice get-compliance-summary-by-config-rule

# Security Hub
aws securityhub describe-standards
aws securityhub get-findings --filters '{"ComplianceStatus": [{"Value": "FAILED", "Comparison": "EQUALS"}]}'
aws securityhub get-insights

# Well-Architected
aws wellarchitected list-workloads --output table
aws wellarchitected list-lenses --output table
```
