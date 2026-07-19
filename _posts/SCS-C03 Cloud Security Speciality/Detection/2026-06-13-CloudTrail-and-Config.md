---
layout: post
title: AWS CloudTrail and AWS Config — Audit Logging and Compliance
date: 2026-06-13T10:00:00
categories:
  - SCS-C03 Cloud Security Speciality
  - Detection
tags:
  - aws
  - cloudtrail
  - config
  - audit
  - compliance
  - cloud-security
  - scs-c03
author: muhammed
description: SCS-C03 Domain 1 — CloudTrail organisation trails, log integrity validation, Athena analysis, Config rules, conformance packs, and automated remediation
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## AWS CloudTrail

**CloudTrail** records every API call made in your AWS account — who did what, when, from where.
It is the primary audit trail for AWS.
Every action in the console, CLI, SDK, or any AWS service that calls another AWS service creates a CloudTrail event.

---

## Trail Types

| Type | Scope | Use Case |
|---|---|---|
| **Single-region trail** | One region only | Legacy, avoid |
| **Multi-region trail** | All current and future regions | Standard — always use this |
| **Organisation trail** | All accounts in an AWS Organization | Security baseline — enables centralised logging |

### Organisation Trail

An organisation trail is created in the **management account** or **delegated admin account**.
It automatically applies to all existing and new member accounts.
Member accounts cannot modify or delete it.
This is the correct way to centralise audit logs at scale.

```bash
# Create an organisation multi-region trail
aws cloudtrail create-trail \
  --name org-audit-trail \
  --s3-bucket-name org-cloudtrail-logs \
  --is-multi-region-trail \
  --is-organization-trail \
  --enable-log-file-validation \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123

# Start logging
aws cloudtrail start-logging --name org-audit-trail
```

> CloudTrail is **global** for the management events of global services (IAM, STS, CloudFront) regardless of region.
> Always enable log file validation and KMS encryption.

> 📸 **SCREENSHOT:** CloudTrail → Trails → select trail → General details.
> Show the multi-region toggle enabled, organisation trail toggle enabled, log file validation enabled, and the S3 bucket destination.

---

## Event Types

| Event Type | What It Captures | Default? |
|---|---|---|
| **Management events** | Control plane — create, modify, delete resources | Enabled (free) |
| **Data events** | Data plane — S3 object reads/writes, Lambda invocations, DynamoDB GetItem | Disabled (costs extra) |
| **Insights events** | Anomalous API activity — unusual call volumes | Disabled (costs extra) |

**For the exam:** Data events are critical for S3 forensics and detecting data exfiltration.
Enable them on sensitive buckets (at minimum) or account-wide.

---

## Log File Integrity Validation

CloudTrail creates a **digest file** every hour that contains hashes of all log files delivered in that period.
Each digest file is signed with CloudTrail's private key.
If a log file is modified, deleted, or forged, the validation fails.

```bash
# Validate log integrity (checks all digest files in a time range)
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:us-east-1:123456789012:trail/org-audit-trail \
  --start-time 2026-06-01T00:00:00Z \
  --end-time 2026-06-13T23:59:59Z
```

**Exam gotcha:** Log integrity validation only works if the trail was configured with `--enable-log-file-validation` from the start.
Enabling it later only validates future logs.

---

## Protecting CloudTrail Logs

The S3 bucket storing CloudTrail logs must be protected:

```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::org-cloudtrail-logs/*"
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutBucketPolicy",
      "Resource": "arn:aws:s3:::org-cloudtrail-logs",
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::123456789012:role/CloudTrailRole"
        }
      }
    }
  ]
}
```

Also enable **S3 Object Lock** (WORM) with Compliance mode on the log bucket for tamper-proof retention.
Enable **MFA Delete** on the bucket versioning so deletions require MFA.

---

## Querying CloudTrail with Athena

CloudTrail logs are stored as JSON in S3.
Use **Athena** to query them with SQL — far faster than searching raw JSON.

CloudTrail can automatically create an Athena table for you.

```sql
-- Find all actions by a specific user
SELECT eventtime, eventname, sourceipaddress, requestparameters
FROM cloudtrail_logs
WHERE useridentity.arn = 'arn:aws:iam::123456789012:user/alice'
  AND eventtime BETWEEN '2026-06-01' AND '2026-06-13'
ORDER BY eventtime DESC;

-- Find all failed API calls (access denied)
SELECT eventtime, eventname, errorcode, errormessage, useridentity.arn
FROM cloudtrail_logs
WHERE errorcode = 'AccessDenied'
ORDER BY eventtime DESC
LIMIT 100;

-- Find root account usage
SELECT eventtime, eventname, sourceipaddress
FROM cloudtrail_logs
WHERE useridentity.type = 'Root'
ORDER BY eventtime DESC;

-- Find IAM changes
SELECT eventtime, eventname, useridentity.arn, requestparameters
FROM cloudtrail_logs
WHERE eventsource = 'iam.amazonaws.com'
  AND eventname IN ('CreateUser','AttachUserPolicy','CreateAccessKey','PutUserPolicy')
ORDER BY eventtime DESC;
```

---

## CloudWatch Logs Integration

Stream CloudTrail logs to CloudWatch Logs to create **metric filters and alarms** for real-time alerting.

**Security-critical alarms to configure:**

```bash
# Alarm: root account login
aws logs put-metric-filter \
  --log-group-name CloudTrail/DefaultLogGroup \
  --filter-name RootAccountUsage \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }' \
  --metric-transformations MetricName=RootAccountUsage,MetricNamespace=CloudTrailMetrics,MetricValue=1

# Alarm: CloudTrail disabled
--filter-pattern '{ ($.eventName = StopLogging) }'

# Alarm: S3 bucket policy change
--filter-pattern '{ ($.eventSource = s3.amazonaws.com) && (($.eventName = PutBucketAcl) || ($.eventName = PutBucketPolicy) || ($.eventName = DeleteBucketPolicy)) }'

# Alarm: IAM policy change
--filter-pattern '{ ($.eventName = DeleteGroupPolicy) || ($.eventName = DeleteRolePolicy) || ($.eventName = DeleteUserPolicy) || ($.eventName = PutGroupPolicy) || ($.eventName = PutRolePolicy) || ($.eventName = PutUserPolicy) }'

# Alarm: security group changes
--filter-pattern '{ ($.eventName = AuthorizeSecurityGroupIngress) || ($.eventName = AuthorizeSecurityGroupEgress) || ($.eventName = RevokeSecurityGroupIngress) || ($.eventName = RevokeSecurityGroupEgress) }'
```

> 📸 **SCREENSHOT:** CloudWatch → Log groups → CloudTrail log group → Metric filters.
> Show the list of metric filters with their filter patterns and the corresponding CloudWatch alarms linked to SNS.

---

## AWS Config

**AWS Config** records the configuration state of your AWS resources over time.
It answers: "What did this resource look like at a specific point in time?" and "Has this resource drifted from its required configuration?"

Config is complementary to CloudTrail — CloudTrail records API calls (events), Config records resource state (snapshots and history).

### Config Components

| Component | Purpose |
|---|---|
| **Configuration recorder** | Records resource configuration changes |
| **Delivery channel** | Where Config sends snapshots and history (S3 + SNS) |
| **Config rules** | Evaluate resources against desired configuration |
| **Remediation actions** | Automatically fix non-compliant resources via SSM Automation |
| **Aggregator** | View compliance across multiple accounts and regions |
| **Conformance pack** | A bundle of Config rules deployed together |

```bash
# Enable Config (create recorder + delivery channel)
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=arn:aws:iam::123456789012:role/ConfigRole \
  --recording-group allSupported=true,includeGlobalResourceTypes=true

aws configservice put-delivery-channel \
  --delivery-channel name=default,s3BucketName=my-config-bucket,snsTopicARN=arn:aws:sns:...:config-notifications

aws configservice start-configuration-recorder --configuration-recorder-name default
```

---

## Config Rules

Config rules evaluate your resources against a desired state.
AWS provides **managed rules** (pre-built) and you can write **custom rules** using Lambda.

### Key Managed Rules for the Exam

| Rule | What It Checks |
|---|---|
| `s3-bucket-public-read-prohibited` | No S3 bucket allows public read |
| `s3-bucket-server-side-encryption-enabled` | S3 buckets have encryption enabled |
| `cloudtrail-enabled` | CloudTrail is enabled and logging |
| `root-mfa-enabled` | Root account has MFA enabled |
| `iam-user-mfa-enabled` | All IAM users have MFA |
| `iam-password-policy` | Account password policy meets requirements |
| `restricted-ssh` | No security group allows SSH from 0.0.0.0/0 |
| `restricted-common-ports` | No SG allows unrestricted access to common ports |
| `rds-instance-public-access-check` | RDS instances are not publicly accessible |
| `encrypted-volumes` | All EBS volumes are encrypted |
| `guardduty-enabled-centralized` | GuardDuty is enabled in all accounts |
| `securityhub-enabled` | Security Hub is enabled |

```bash
# Create a Config rule
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::S3::Bucket"]
    }
  }'

# Check compliance
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name s3-bucket-public-read-prohibited \
  --compliance-types NON_COMPLIANT
```

---

## Automated Remediation

Config rules can trigger **SSM Automation documents** to automatically fix non-compliant resources.

```bash
aws configservice put-remediation-configurations \
  --remediation-configurations '[{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "TargetType": "SSM_DOCUMENT",
    "TargetId": "AWS-DisableS3BucketPublicReadWrite",
    "Parameters": {
      "AutomationAssumeRole": {
        "StaticValue": {"Values": ["arn:aws:iam::123456789012:role/ConfigRemediationRole"]}
      },
      "S3BucketName": {
        "ResourceValue": {"Value": "RESOURCE_ID"}
      }
    },
    "Automatic": true,
    "MaximumAutomaticAttempts": 3,
    "RetryAttemptSeconds": 60
  }]'
```

---

## Conformance Packs

A **conformance pack** is a collection of Config rules and remediation actions deployed as a single unit.
AWS provides pre-built packs for common compliance frameworks.

```bash
# Deploy CIS AWS Foundations Benchmark conformance pack
aws configservice put-conformance-pack \
  --conformance-pack-name CIS-Benchmark \
  --template-s3-uri s3://aws-config-managed-rules/CISAwsFoundationsBenchmark.yaml

# Deploy across organisation (from delegated admin)
aws configservice put-organization-conformance-pack \
  --organization-conformance-pack-name CIS-Org-Wide \
  --template-s3-uri s3://aws-config-managed-rules/CISAwsFoundationsBenchmark.yaml
```

---

## Config Aggregator

An **aggregator** collects compliance data from multiple accounts and regions into a single view.

```bash
# Create an organisation aggregator
aws configservice put-configuration-aggregator \
  --configuration-aggregator-name org-aggregator \
  --organization-aggregation-source '{
    "RoleArn": "arn:aws:iam::123456789012:role/ConfigAggregatorRole",
    "AllAwsRegions": true
  }'
```

---

## CloudTrail vs Config — Exam Distinction

| | CloudTrail | Config |
|---|---|---|
| **Records** | API calls (who did what) | Resource configuration state (what it looks like now) |
| **Answers** | "Who deleted this security group rule?" | "Was this S3 bucket ever publicly accessible?" |
| **Timeline** | Event-based log | Continuous configuration snapshot |
| **Alerting** | CloudWatch metric filters | Config rules → SNS / EventBridge |
| **Forensics** | Investigate what happened | Investigate what state resources were in |

---

## Quick Reference

```bash
# CloudTrail
aws cloudtrail describe-trails
aws cloudtrail get-trail-status --name org-audit-trail
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteBucket

# Config
aws configservice describe-configuration-recorders
aws configservice get-compliance-summary-by-resource-type
aws configservice list-discovered-resources --resource-type AWS::S3::Bucket
aws configservice get-resource-config-history \
  --resource-type AWS::EC2::SecurityGroup \
  --resource-id sg-abc123
```
