---
layout: post
title: Amazon GuardDuty — Threat Detection, Finding Types, and Automation
date: 2026-06-11T10:00:00
categories:
  - SCS-C03 Cloud Security Speciality
  - Detection
tags:
  - aws
  - guardduty
  - threat-detection
  - cloud-security
  - scs-c03
author: muhammed
description: SCS-C03 Domain 1 — GuardDuty finding types, data sources, multi-account setup, trusted IP lists, suppression rules, and EventBridge automation
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## What is GuardDuty?

**Amazon GuardDuty** is a managed threat detection service that continuously monitors your AWS accounts for malicious activity and unauthorised behaviour.
It analyses multiple data sources using machine learning, anomaly detection, and integrated threat intelligence.
There are no agents to install, no hardware to manage, and no log pipelines to build — you enable it and it starts detecting.

GuardDuty feeds its findings into **Security Hub** and can trigger **EventBridge** rules to automate responses.

---

## Data Sources

GuardDuty analyses the following sources automatically:

| Data Source | What It Detects |
|---|---|
| **CloudTrail Management Events** | API calls — unusual API activity, credential abuse, new IAM users |
| **CloudTrail S3 Data Events** | S3 object-level operations — exfiltration, ransomware staging |
| **VPC Flow Logs** | Network traffic — port scanning, C2 communication, crypto mining |
| **DNS Logs** | DNS queries from EC2 — domains associated with malware, data exfiltration via DNS |
| **EKS Audit Logs** | Kubernetes API calls — privilege escalation, lateral movement in clusters |
| **EBS Malware Protection** | Scans EBS volumes of suspicious EC2 instances for malware |
| **Lambda Network Activity** | Unusual Lambda network connections — C2 traffic, crypto mining |
| **RDS Login Activity** | Anomalous login patterns to Aurora databases |
| **Runtime Monitoring (EC2/ECS/EKS)** | OS-level activity — process execution, file access, network connections |

> 📸 **SCREENSHOT:** GuardDuty → Settings → Data sources.
> Show all data source toggles and their status (enabled/disabled) with the coverage summary.

---

## Finding Types

GuardDuty findings follow the naming convention: `ThreatPurpose:ResourceType/ThreatFamilyName.DetectionMechanism`

### High-Value Finding Types to Know for the Exam

**Credential Compromise:**
- `UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B` — successful console login from unusual location
- `UnauthorizedAccess:IAMUser/MaliciousIPCaller` — API calls from a known malicious IP
- `CredentialAccess:IAMUser/AnomalousBehavior` — unusual API calls for credentials
- `Policy:IAMUser/RootCredentialUsage` — root account used (should never happen in production)

**Instance Compromise:**
- `Backdoor:EC2/C&CActivity.B` — EC2 communicating with known command-and-control server
- `CryptoCurrency:EC2/BitcoinTool.B` — EC2 querying crypto mining pool
- `Trojan:EC2/DNSDataExfiltration` — EC2 exfiltrating data via DNS queries
- `Recon:EC2/PortProbeUnprotectedPort` — port scanning against unprotected ports

**S3 Threats:**
- `Policy:S3/BucketPublicAccessGranted` — bucket ACL changed to public
- `Stealth:S3/ServerAccessLoggingDisabled` — S3 access logging disabled
- `UnauthorizedAccess:S3/MaliciousIPCaller.Custom` — S3 access from IP on custom threat list

**Kubernetes:**
- `Execution:Kubernetes/ExecInKubePod` — exec command run inside a Kubernetes pod
- `Persistence:Kubernetes/ContainerWithSensitiveMount` — container with sensitive host path mounted
- `PrivilegeEscalation:Kubernetes/PrivilegedContainer` — privileged container launched

### Finding Severity

| Severity | Score | Meaning |
|---|---|---|
| **High** | 7.0–8.9 | Immediate action required — active threat |
| **Medium** | 4.0–6.9 | Investigate — suspicious activity |
| **Low** | 0.1–3.9 | Informational — monitor for patterns |

---

## Multi-Account Setup

In an organisation, enable GuardDuty from a **delegated administrator account** (typically a security tooling account).
The admin account can see and manage findings across all member accounts.
Member accounts cannot disable GuardDuty once enrolled from the admin.

```bash
# Enable GuardDuty
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES

# Get your detector ID
aws guardduty list-detectors

# Enable as organization admin (run from management account)
aws guardduty enable-organization-admin-account \
  --admin-account-id 111122223333

# Auto-enrol new accounts
aws guardduty update-organization-configuration \
  --detector-id abc123 \
  --auto-enable-organization-members ALL
```

> 📸 **SCREENSHOT:** GuardDuty → Settings → Accounts.
> Show the list of member accounts with their status (Enabled), and the "Auto-enable" toggle for new accounts.

---

## Trusted IP Lists and Threat Lists

**Trusted IP list** — IP addresses that GuardDuty should NOT generate findings for (e.g. your office IPs, VPN exit points).
**Threat IP list** — additional malicious IP addresses you supply beyond GuardDuty's built-in intelligence.

```bash
# Upload a trusted IP list (file must be in S3)
aws guardduty create-ip-set \
  --detector-id abc123 \
  --name "Office-IPs" \
  --format TXT \
  --location s3://my-guardduty-lists/trusted-ips.txt \
  --activate

# Upload a threat list
aws guardduty create-threat-intel-set \
  --detector-id abc123 \
  --name "Custom-Threat-IPs" \
  --format TXT \
  --location s3://my-guardduty-lists/threat-ips.txt \
  --activate
```

---

## Suppression Rules

**Suppression rules** automatically archive findings that match your criteria — useful for known-good activity that GuardDuty flags as suspicious.

Example: suppress port probe findings from your vulnerability scanner's IP range.

> 📸 **SCREENSHOT:** GuardDuty → Findings → Suppression rules → Create suppression rule.
> Show a rule filtering on finding type (Recon:EC2/PortProbeUnprotectedPort) and source IP CIDR matching your scanner.

```bash
aws guardduty create-filter \
  --detector-id abc123 \
  --name "suppress-vuln-scanner" \
  --action ARCHIVE \
  --finding-criteria '{
    "Criterion": {
      "type": {"Eq": ["Recon:EC2/PortProbeUnprotectedPort"]},
      "service.action.networkConnectionAction.remoteIpDetails.ipAddressV4": {
        "Eq": ["10.0.100.50"]
      }
    }
  }'
```

---

## EventBridge Automation

GuardDuty publishes every new finding as an **EventBridge event**.
Use EventBridge rules to trigger automated responses.

**Common automated response patterns:**

| Trigger | Automated Action |
|---|---|
| `Policy:IAMUser/RootCredentialUsage` | SNS alert → PagerDuty page |
| `UnauthorizedAccess:IAMUser/MaliciousIPCaller` | Lambda → revoke IAM access keys, notify team |
| `CryptoCurrency:EC2/BitcoinTool.B` | Lambda → isolate instance (move to quarantine SG) |
| `Backdoor:EC2/C&CActivity.B` | Lambda → snapshot instance + isolate |

```bash
# EventBridge rule for high-severity findings
aws events put-rule \
  --name "guardduty-high-severity" \
  --event-pattern '{
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": {
      "severity": [{"numeric": [">=", 7]}]
    }
  }'

# Target — Lambda function to isolate the instance
aws events put-targets \
  --rule guardduty-high-severity \
  --targets '[{
    "Id": "isolate-instance",
    "Arn": "arn:aws:lambda:eu-west-1:123456789012:function:isolate-ec2"
  }]'
```

---

## GuardDuty + Security Hub Integration

When you enable the Security Hub integration, all GuardDuty findings are automatically sent to Security Hub as findings in **ASFF (Amazon Security Finding Format)**.
Security Hub normalises findings from GuardDuty, Inspector, Macie, and third-party tools into one view.

```bash
# Enable Security Hub integration (done from Security Hub side)
aws securityhub enable-import-findings-for-product \
  --product-arn arn:aws:securityhub:eu-west-1::product/aws/guardduty
```

---

## Exam Key Points

- GuardDuty does **not** prevent threats — it **detects** them. Prevention is WAF, SG, NACLs.
- Disabling GuardDuty loses all existing findings — **suppression rules** are the right way to reduce noise.
- **Root credential usage** always generates a finding regardless of legitimacy — rotate away from root for everything.
- **Multi-account**: delegated admin in security account, member accounts auto-enrolled via Organizations.
- GuardDuty analyses VPC Flow Logs and DNS logs **without enabling them in your account** — it reads them directly from the underlying infrastructure.
- EBS Malware Protection creates a **replica snapshot** to scan — it does not affect the running instance.

---

## Quick Reference

```bash
# List findings (filtered by severity)
aws guardduty list-findings \
  --detector-id abc123 \
  --finding-criteria '{"Criterion": {"severity": {"Gte": 7}}}'

# Get finding details
aws guardduty get-findings \
  --detector-id abc123 \
  --finding-ids finding-id-here

# Archive findings (mark as reviewed)
aws guardduty archive-findings \
  --detector-id abc123 \
  --finding-ids finding-id-here

# Generate sample findings (all finding types, for testing)
aws guardduty create-sample-findings \
  --detector-id abc123 \
  --finding-types "Backdoor:EC2/C&CActivity.B" "CryptoCurrency:EC2/BitcoinTool.B"

# Detector status
aws guardduty get-detector --detector-id abc123
```
