---
layout: post
title: Cloud Incident Response and Amazon Detective — IR Playbooks and Investigation
date: 2026-06-17T10:00:00
categories:
  - SCS-C03 Cloud Security Speciality
  - Incident Response
tags:
  - aws
  - incident-response
  - detective
  - forensics
  - cloud-security
  - scs-c03
author: muhammed
description: SCS-C03 Domain 2 — Cloud IR phases, EC2 forensics and containment, automated response with SSM and Step Functions, and Amazon Detective graph-based investigation
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## Cloud Incident Response Overview

Cloud IR follows the same phases as traditional IR but with cloud-specific tooling and considerations.

```
Prepare → Detect → Contain → Investigate → Eradicate → Recover → Post-Incident
```

The key difference from on-premises IR: in AWS you can act programmatically at scale — isolate an instance, snapshot a disk, revoke credentials, and spin up a forensic environment all via API calls in seconds.

---

## Phase 1 — Prepare

Preparation happens before any incident occurs.

**IAM break-glass accounts:**
Keep a dedicated emergency IAM role with broad permissions, locked behind MFA, only used during incidents.
Access to this role is itself monitored via CloudTrail.

**Pre-built IR runbooks in SSM OpsCenter:**
Document and store your response procedures as SSM Automation documents.
When an incident is triggered by GuardDuty → EventBridge, OpsCenter creates an OpsItem and can auto-trigger the runbook.

**Security tooling baseline (all accounts):**
- GuardDuty enabled, organisation-wide
- CloudTrail organisation trail active with log integrity validation
- Config recording all resource types
- VPC Flow Logs enabled on all VPCs
- S3 access logging on sensitive buckets
- Security Hub enabled with FSBP and CIS standards

---

## Phase 2 — Detect

Detection sources in AWS:

| Source | What It Catches |
|---|---|
| **GuardDuty** | Malicious activity, credential misuse, C2 communication |
| **Security Hub** | Policy violations, CSPM failures, aggregated findings |
| **CloudTrail + CloudWatch alarms** | Specific API events — root login, SG changes, IAM changes |
| **Config rules** | Resource configuration drift |
| **Macie** | Sensitive data exposure in S3 |
| **VPC Flow Logs** | Unusual traffic patterns |

---

## Phase 3 — Contain (EC2 Compromise)

When an EC2 instance is suspected compromised, the containment goal is to stop the spread without destroying evidence.

### Step 1 — Isolate the Instance

Move the instance to a **quarantine security group** — no inbound, no outbound except to your forensic tools.

```bash
# Create quarantine security group (no inbound or outbound)
aws ec2 create-security-group \
  --group-name quarantine-sg \
  --description "Quarantine — no inbound or outbound traffic" \
  --vpc-id vpc-abc123

# Remove all outbound rules (default allows all outbound)
aws ec2 revoke-security-group-egress \
  --group-id sg-quarantine \
  --ip-permissions '[{"IpProtocol": "-1", "IpRanges": [{"CidrIp": "0.0.0.0/0"}]}]'

# Move the instance to the quarantine SG
aws ec2 modify-instance-attribute \
  --instance-id i-0abc123 \
  --groups sg-quarantine
```

### Step 2 — Preserve Evidence

Take a snapshot of every EBS volume before doing anything else.
The snapshot is the forensic artifact.

```bash
# Get all volumes attached to the instance
VOLUMES=$(aws ec2 describe-instances \
  --instance-ids i-0abc123 \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings[*].Ebs.VolumeId' \
  --output text)

# Snapshot each volume
for VOL in $VOLUMES; do
  aws ec2 create-snapshot \
    --volume-id $VOL \
    --description "FORENSIC-$(date +%Y%m%d)-$VOL" \
    --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Purpose,Value=forensic},{Key=IncidentId,Value=INC-2026-001}]'
done
```

### Step 3 — Revoke IAM Credentials

If the instance had an IAM role, invalidate any active sessions immediately.

```bash
# Revoke all active sessions for a role (by setting a policy that denies all actions before now)
aws iam put-role-policy \
  --role-name compromised-instance-role \
  --policy-name RevokeOldSessions \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "DateLessThan": {"aws:TokenIssueTime": "2026-06-17T10:00:00Z"}
      }
    }]
  }'
```

### Step 4 — Disable IAM User Access Keys (if keys were compromised)

```bash
# Disable the key immediately
aws iam update-access-key \
  --user-name compromised-user \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --status Inactive

# Detach all policies (if account is fully compromised)
aws iam list-attached-user-policies --user-name compromised-user
# ... then detach each one
```

---

## Phase 4 — Investigate

### Forensic EC2 Instance

Create a separate **forensic workstation** in an isolated forensic VPC.
Attach a copy of the suspect volume (created from the snapshot) to the forensic instance.
Analyse it without ever powering on the compromised instance again.

```bash
# Create volume from forensic snapshot in forensic AZ
aws ec2 create-volume \
  --snapshot-id snap-forensic-abc123 \
  --availability-zone eu-west-1a \
  --volume-type gp3 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Purpose,Value=forensic}]'

# Attach to forensic workstation as read-only
aws ec2 attach-volume \
  --volume-id vol-forensic-abc123 \
  --instance-id i-forensic-workstation \
  --device /dev/sdf
```

### CloudTrail Forensics

Query CloudTrail to build a timeline of what the compromised principal did.

```bash
# Look up all actions by a compromised access key
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAIOSFODNN7EXAMPLE \
  --start-time 2026-06-10T00:00:00Z \
  --end-time 2026-06-17T23:59:59Z
```

For large volumes of logs, use Athena for SQL-based queries across weeks of CloudTrail data stored in S3.

### VPC Flow Log Analysis

Check what external IPs the instance communicated with.

```bash
# Query flow logs in Athena
SELECT srcaddr, dstaddr, dstport, action, bytes
FROM vpc_flow_logs
WHERE srcaddr = '10.0.1.50'
  AND start > 1718000000
ORDER BY bytes DESC
LIMIT 100;
```

---

## Automated IR with Step Functions

For repeatable incident scenarios, build automated response workflows using **Step Functions** + **Lambda** triggered by EventBridge.

```
GuardDuty Finding (high severity)
    │
    ▼ EventBridge Rule
Step Functions Workflow:
    1. Snapshot all EBS volumes (Lambda)
    2. Move instance to quarantine SG (Lambda)
    3. Revoke IAM role sessions (Lambda)
    4. Create OpsCenter OpsItem (Lambda)
    5. Notify security team via SNS (Lambda)
    6. Wait for human approval
    7. Terminate instance (Lambda)
```

---

## Amazon Detective

**Amazon Detective** automatically collects log data from your AWS accounts — CloudTrail, VPC Flow Logs, GuardDuty findings, EKS audit logs, Security Hub — and builds an interactive graph model of relationships and behaviours.

Where GuardDuty **detects** threats, Detective helps you **investigate** them.
You start from a finding and Detective shows you the full context: which principal made the call, what else they did, what's normal for that entity, and how the activity relates to other entities.

### Enabling Detective

```bash
# Enable Detective (it automatically starts collecting data from your account)
aws detective create-graph

# Enable across an organisation
aws detective enable-organization-admin-account \
  --account-id 111122223333

# Accept invitation as member account
aws detective accept-invitation \
  --graph-arn arn:aws:detective:eu-west-1:123456789012:graph:abc123
```

> 📸 **SCREENSHOT:** Amazon Detective → Summary page.
> Show the behaviour graph with entity types (AWS accounts, IAM users, IAM roles, IP addresses, EC2 instances) and the findings panel on the right showing linked GuardDuty findings.

### Detective Investigation Workflow

1. Start from a **GuardDuty finding** or **Security Hub finding**
2. Click "Investigate with Detective" — Detective opens with that entity in context
3. View the **entity profile** — what is normal for this IP / user / role?
4. Examine the **finding group** — other entities and findings connected to this incident
5. Review the **activity timeline** — API calls, network connections, process activity over time

### Key Detective Concepts

| Concept | Meaning |
|---|---|
| **Behaviour graph** | The continuously updated model of relationships and activity in your account |
| **Entity profile** | A view of all activity for a specific entity (IP, user, role, instance) over time |
| **Finding group** | A cluster of related findings that Detective thinks are connected to the same incident |
| **Scope time** | The time window Detective uses to show activity in an investigation |

---

## Exam Key Points

- **Containment order for EC2 compromise:** isolate (quarantine SG) → snapshot (preserve evidence) → revoke credentials
- **Never terminate** a compromised instance before snapshotting — you lose all forensic evidence
- **Forensic analysis** happens on a copy of the disk (snapshot → new volume) in an isolated forensic environment, never on the live instance
- **STS credential revocation** — there is no direct "revoke" API. The pattern is: add a deny policy with `DateLessThan aws:TokenIssueTime` to invalidate all tokens issued before the policy was applied
- **Detective vs GuardDuty**: GuardDuty detects, Detective investigates. Detective needs GuardDuty to be enabled.
- **Detective** retains data for **12 months** and the behaviour graph starts producing useful patterns after about 2 weeks of data ingestion
- **SSM OpsCenter** is the right place to manage incident tracking, runbook execution, and remediation automation from within AWS

---

## Quick Reference

```bash
# Detective
aws detective list-graphs
aws detective list-members --graph-arn arn:aws:detective:...:graph:abc123
aws detective search-graph --graph-arn ARN --query-body '{"EntityArn": "arn:aws:iam::..."}'

# SSM OpsCenter (IR tracking)
aws ssm create-ops-item \
  --title "Security Incident - Compromised Instance i-0abc123" \
  --source "GuardDuty" \
  --severity "1" \
  --priority 1 \
  --notifications '[{"Arn": "arn:aws:sns:...:security-alerts"}]'

aws ssm describe-ops-items --ops-item-filters 'Key=Status,Values=Open,Operator=Equal'
```
