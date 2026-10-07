---
layout: post
title: "Day 3: CloudTrail & AWS Config - API Auditing and Drift Detection"
date: 2026-03-03 10:00:00 +0800
categories:
  - DevSecOps
  - Week1
tags:
  - AWS
  - CloudTrail
  - AWSConfig
  - Auditing
  - CloudSecurity
author: muhammed
description: A practical walkthrough of AWS CloudTrail for API auditing and AWS Config for continuous resource compliance, configuration tracking, and drift detection.
toc: true
pin: false
math: false
mermaid: false
image: https://miro.medium.com/0*YVphulvs386TZYkE.png
---

## The Big Beginner Question: CloudTrail vs AWS Config

When learning AWS security, one of the most common points of confusion is:

> *"AWS has CloudTrail and AWS Config. Don't both of them just monitor and log stuff in AWS? Why do I need both, and what is the difference between them?"*

I struggled with this distinction until someone explained the difference using physical security:

- **CloudTrail is the Security Camera DVR:** It records the continuous video feed of actions. Who opened the front door? At what exact millisecond? Which key did they use? What room did they enter? It answers: *"Who did what, when, and from where?"*
- **AWS Config is the Building Inspector / Inventory Ledger:** It inspects the actual physical state of the building over time. Is the front door currently locked? Was a new window installed yesterday without a permit? Does the building match the city building code? It answers: *"What does our infrastructure look like right now, how has its state evolved, and does it comply with our security rules?"*

```
+-------------------------------------------------------------------+
|               CloudTrail: "Who did what and when?"                |
|  Alice called CreateSecurityGroup at 14:02:11 from 203.0.113.5    |
+-------------------------------------------------------------------+
                                  vs
+-------------------------------------------------------------------+
|            AWS Config: "What does the resource look like?"        |
|  sg-01234 has Port 22 open to 0.0.0.0/0 -> STATUS: NON_COMPLIANT  |
+-------------------------------------------------------------------+
```

Together, they form the bedrock of cloud detection and forensics. Today, we will break down both services, learn how to query audit logs with Amazon Athena, set up automated compliance rules, and avoid the billing traps that catch juniors off guard.

---

## AWS CloudTrail: The Audit Trail

In AWS, every single action is an API call. When you click a button in the AWS console, run an AWS CLI command, or deploy Terraform code, an HTTPS request hits an AWS API endpoint.

**AWS CloudTrail records every single one of those API calls.**

### Anatomy of a CloudTrail Event

Every CloudTrail log record contains detailed JSON metadata about the request:

| Field | Meaning | Why It Matters for Forensics |
| :--- | :--- | :--- |
| `eventTime` | UTC timestamp | Pinpoints the exact moment an action occurred |
| `userIdentity` | Identity that made the call | Reveals the IAM user, assumed role, or AWS service |
| `eventName` | The API method executed | e.g. `RunInstances`, `DeleteBucket`, `AuthorizeSecurityGroupIngress` |
| `sourceIPAddress`| Caller IP address | Identifies where the request originated (corporate VPN vs suspicious IP) |
| `requestParameters`| Inputs sent with the request | Shows which CIDR blocks were opened or which bucket was targeted |
| `responseElements` | AWS API response | Shows created resource IDs or returned metadata |
| `errorCode` | Error status if rejected | Reveals brute force attempts or `AccessDenied` anomalies |

![CloudTrail event record JSON overview](/assets/devsecops/week1/Pasted%20image%2020260521174358.png)

### CloudTrail Event Types

CloudTrail categorizes events into two main buckets:

1. **Management Events (Control Plane):** Actions that create, modify, or delete resources (e.g. creating a VPC, creating an IAM user, changing a security group).
   - *Cost note:* The first copy of management events delivered to an S3 bucket is **free**.
2. **Data Events (Data Plane):** High-volume operations performed *inside* a resource (e.g. `s3:GetObject`, `s3:PutObject`, DynamoDB reads/writes, Lambda invocations).
   - *Cost warning:* Data events are not logged by default. Enabling them costs $0.10 per 100,000 events. If you enable data events on a busy S3 bucket that serves static website assets receiving millions of requests, you can easily rack up hundreds of dollars in unexpected charges!
3. **CloudTrail Insights:** An optional machine learning add-on that analyzes management event baselines to detect anomalous spikes in API calls or error rates.

---

## Setting Up an Enterprise Trail

By default, the AWS console only keeps management event history for **90 days**. If an attacker penetrated your environment four months ago and you did not configure a Trail, your forensic evidence is permanently gone!

To preserve logs indefinitely, you must create a permanent **Trail** that exports to an encrypted S3 bucket.

![Trail creation S3 bucket settings](/assets/devsecops/week1/Pasted%20image%2020260521181033.png)

### Best Practice Configuration Checklist

When configuring an enterprise trail, ensure:

1. **Multi-Region Collection:** Always select "Apply trail to all regions". Attackers frequently launch unauthorized crypto-mining EC2 instances in unused regions (like `ap-southeast-2` or `eu-north-1`) hoping nobody will notice. Multi-region trails catch activity everywhere.
2. **Organization-Wide Trail:** If using AWS Organizations, create the trail in the Management Account with "Enable for all accounts in my organization". All member account logs stream centrally into a single dedicated Log Archive S3 bucket.
3. **Log File Validation:** Enable log file validation. Every hour, CloudTrail generates a cryptographic digest file (containing SHA-256 hashes of the logs) stored in S3. This allows you to mathematically prove that logs were not modified or deleted after the fact.
4. **SSE-KMS Encryption:** Encrypt the S3 bucket using a customer-managed AWS KMS key so only authorized security principals can decrypt and read the logs.

### Verifying Log Integrity via CLI

You can verify whether log files have been tampered with using the AWS CLI:

```bash
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:ap-southeast-1:123456789012:trail/org-audit-trail \
  --start-time 2026-05-20T00:00:00Z
```

![Validating logs CLI command](/assets/devsecops/week1/Pasted%20image%2020260521181850.png)

If any log file in S3 was altered or deleted, the validation command flags a hash mismatch error immediately.

---

## Querying CloudTrail Logs with Amazon Athena

CloudTrail writes logs as gzipped JSON files in S3. Trying to download and read these files manually during an active incident is painful.

Instead, we use **Amazon Athena**, a serverless SQL query engine, to query CloudTrail logs directly in S3.

### Setting Up Athena for CloudTrail

From the CloudTrail console, open **Event history** and click **Create Athena table**. AWS automatically generates the table schema mapped to your S3 log bucket.

![Athena table creation from CloudTrail](/assets/devsecops/week1/Pasted%20image%2020260521182333.png)

### Essential Forensic SQL Queries

Once the table is created, you can run SQL queries in Athena:

#### Query 1: Detect S3 Bucket Deletions

Find who deleted an S3 bucket in the last 7 days:

```sql
SELECT
  eventtime,
  useridentity.arn AS caller_identity,
  sourceipaddress,
  requestparameters
FROM cloudtrail_logs
WHERE eventname = 'DeleteBucket'
  AND eventtime > '2026-05-13'
ORDER BY eventtime DESC;
```

#### Query 2: Investigate Console Logins from Non-Corporate IPs

Identify root or IAM console logins originating outside your company IP range:

```sql
SELECT
  eventtime,
  useridentity.username,
  sourceipaddress,
  responseelements
FROM cloudtrail_logs
WHERE eventname = 'ConsoleLogin'
  AND sourceipaddress NOT LIKE '203.0.113.%'
ORDER BY eventtime DESC;
```

#### Query 3: Spot Unauthorized Security Group Modifications

Detect who opened sensitive ports to the public internet:

```sql
SELECT
  eventtime,
  useridentity.arn,
  sourceipaddress,
  requestparameters
FROM cloudtrail_logs
WHERE eventname IN ('AuthorizeSecurityGroupIngress', 'ModifySecurityGroupRules')
ORDER BY eventtime DESC
LIMIT 20;
```

---

## AWS Config: The Configuration & Compliance Engine

While CloudTrail records the stream of API calls, **AWS Config** tracks the actual state of your resources and detects **configuration drift**.

```
Day 1: Security Group sg-xyz has port 443 open. (Compliant)
Day 5: A developer modifies sg-xyz to open port 22 to 0.0.0.0/0.
       -> AWS Config flags: NON_COMPLIANT!
       -> Shows a visual diff of the changes.
```

![AWS Config resource timeline](/assets/devsecops/week1/Pasted%20image%2020260521183233.png)

### What AWS Config Answers

- *"What was the exact inbound rule configuration of this security group last Tuesday?"*
- *"When did this S3 bucket get its public access block disabled, and who was logged in at that moment?"*
- *"Which EBS volumes in our entire organization are not encrypted at rest?"*
- *"Are all our AWS accounts compliant with the CIS AWS Foundations Benchmark?"*

---

## High-Value AWS Config Managed Rules

AWS provides over 200 pre-built managed rules. Here are the core baseline rules you should enable in every account:

| Managed Rule Name | Security Check |
| :--- | :--- |
| `s3-bucket-public-read-prohibited` | Verifies that no S3 buckets allow public read access |
| `s3-bucket-public-write-prohibited` | Verifies that no S3 buckets allow public write access |
| `restricted-ssh` | Flags security groups permitting inbound SSH (port 22) from `0.0.0.0/0` |
| `restricted-common-ports` | Flags open administrative ports (22, 3389, 1433, 3306) |
| `root-account-mfa-enabled` | Flags accounts where root user does not have MFA enabled |
| `iam-user-mfa-enabled` | Flags any IAM user with console access lacking MFA |
| `encrypted-volumes` | Verifies EBS volumes are encrypted with KMS |
| `cloudtrail-enabled` | Verifies CloudTrail logging is active across all regions |

---

## Automated Remediation with Systems Manager (SSM)

Detection is only half the battle. If a developer accidentally opens an S3 bucket to the public, you do not want to wait 24 hours for a human to notice the email alert.

AWS Config supports **Auto-Remediation** using AWS Systems Manager (SSM) Automation documents:

1. In AWS Config, open the rule `s3-bucket-public-read-prohibited`.
2. Click **Actions -> Manage remediation**.
3. Choose **Automatic remediation**.
4. Select the pre-built SSM document: `AWS-DisableS3BucketPublicReadWrite`.
5. Enter the parameter mapping: `BucketName = ResourceId`.
6. Save the remediation configuration.

Whenever an S3 bucket becomes publicly accessible, AWS Config automatically triggers the SSM automation document to re-enable public access blocks, closing the security gap in seconds!

---

## Quick Comparison: CloudTrail vs AWS Config

| Feature | AWS CloudTrail | AWS Config |
| :--- | :--- | :--- |
| **Primary Question** | "Who called what API?" | "What is the state of this resource?" |
| **Data Format** | Stream of API event records | Resource configuration snapshots & diffs |
| **Historical Focus** | Temporal log of past requests | Configuration evolution over time |
| **Compliance Checking**| No (Raw event logs) | Yes (Evaluates rules: Compliant vs Non-compliant) |
| **Auto-Remediation** | Requires EventBridge + Lambda | Built-in SSM Automation triggers |
| **Console Retention** | 90 days default (indefinite with S3 Trail)| Maintained in S3 snapshot history |

---

## Junior Pitfalls to Avoid

1. **The S3 Data Events Surprise Bill:** Do not enable CloudTrail Data Events on high-throughput S3 buckets without calculating costs. Stick to Management Events first!
2. **Assuming CloudTrail Logs Cannot Be Deleted:** If an attacker compromises your AWS account, their first action is often deleting the CloudTrail trail. Protect your trail using an SCP (`cloudtrail:DeleteTrail` denied) and use S3 Object Lock / MFA Delete on the log bucket.
3. **AWS Config Item Churn:** Recording all resources in high-churn environments (like Kubernetes nodes spinning up and down every 10 minutes) can drive up Config recorder costs. Scope recording to critical security resources (IAM, VPC, S3, Security Groups, EC2) if budget is tight.
4. **Relying Only on the 90-Day Console History:** Always create an S3-backed Trail on Day 1. The default 90-day console event history is not sufficient for incident response investigations.

---

## Key Takeaways

- CloudTrail tracks who did what; AWS Config tracks what resources look like and whether they comply with security baselines.
- Always configure an organization-level, multi-region CloudTrail trail with log file validation and SSE-KMS encryption.
- Use Amazon Athena to query CloudTrail logs with SQL during forensic investigations.
- Enable AWS Config managed rules (like `restricted-ssh` and `s3-bucket-public-read-prohibited`) with auto-remediation to fix drift automatically.
- Keep management events on, but be mindful of data events to prevent surprise AWS bills.

---

## References

<div class="references">
<ul>
  <li><a href="https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html" target="_blank">AWS CloudTrail User Guide</a></li>
  <li><a href="https://docs.aws.amazon.com/config/latest/developerguide/WhatIsAWSConfig.html" target="_blank">AWS Config Developer Guide</a></li>
  <li><a href="https://docs.aws.amazon.com/athena/latest/ug/cloudtrail-logs.html" target="_blank">Querying CloudTrail Logs with Amazon Athena</a></li>
  <li><a href="https://docs.aws.amazon.com/config/latest/developerguide/conformancepack-sample-templates.html" target="_blank">AWS Config Conformance Pack Templates</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
