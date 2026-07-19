---
layout: post
title: "Week 4 — Day 24: CIS Benchmarks & Compliance Basics"
date: 2026-06-13 10:00:00 +0800
categories:
  - DevSecOps
  - Week4
tags:
  - CIS
  - Compliance
  - AWS
  - SecurityHub
  - CloudSecurity
author: muhammed
description: A full walkthrough of CIS AWS Foundations Benchmark — understanding the key controls, running automated checks via Security Hub, remediating failures, and mapping controls to broader compliance frameworks.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fcdn.prod.website-files.com%2F68c7bfed119e14b64d016be9%2F69021231c3d0988fac0e6b7f_CIS%2520AWS-p-800.png&f=1&nofb=1&ipt=d520d309b996b714bc3e8b26fb38f409480ca16755d8b38a1ea9c88af9caa871
---

## Why Compliance Matters in DevSecOps

Compliance frameworks define a baseline of security controls that have been validated by industry experts and regulators. They answer: *"What does a reasonably secure AWS environment look like?"*

For a DevSecOps engineer, compliance serves two purposes:
1. **Baseline** — a checklist of controls you should have regardless of regulatory requirements
2. **Audit readiness** — evidence that controls are in place, continuously monitored

The most relevant framework for AWS is the **CIS AWS Foundations Benchmark**.

---

## CIS AWS Foundations Benchmark

The Center for Internet Security (CIS) publishes hardening benchmarks for every major platform. The AWS Foundations Benchmark covers four domains:

| Section | Controls |
|---------|---------|
| IAM | Root account usage, MFA, access key rotation, password policy |
| Storage | S3 public access, logging, encryption |
| Logging | CloudTrail, Config, VPC flow logs |
| Monitoring | CloudWatch alarms for critical events |
| Networking | Default VPC, security groups, NACLs |

The benchmark has two levels:
- **Level 1** — baseline, low operational impact, should be implemented by everyone
- **Level 2** — more restrictive, may impact some legitimate use cases

---

## Running CIS Checks via Security Hub

Security Hub runs automated CIS checks continuously. Enable it once and it keeps checking.

**Enable the CIS standard:**
1. Security Hub → Security standards → CIS AWS Foundations Benchmark → Enable

> `[SCREENSHOT]` — *Security Hub → Security standards page showing the CIS AWS Foundations Benchmark standard enabled with an overall compliance score (e.g., 67%) and the number of passed/failed controls*

**View all control results:**
1. Security Hub → Security standards → CIS → view all controls

> `[SCREENSHOT]` — *Security Hub → CIS controls list showing controls with their status: green PASSED, red FAILED, grey UNKNOWN — with control IDs like 1.1, 1.2, 2.1 etc.*

---

## Key CIS Controls — Section by Section

### Section 1 — IAM

#### 1.1 — Root account MFA
The root account has unrestricted access to everything. MFA must be enabled.

**Check:**
```bash
aws iam get-account-summary --query 'SummaryMap.AccountMFAEnabled'
# Should return 1
```

**Fix:** AWS Console → Account (top right) → Security credentials → MFA → Activate MFA

> `[SCREENSHOT]` — *Security Hub showing CIS control 1.1 (root MFA) as PASSED with the green checkmark*

#### 1.4 — No root access keys
Root access keys should not exist — use IAM roles instead.

**Check:**
```bash
aws iam get-account-summary --query 'SummaryMap.AccountAccessKeysPresent'
# Should return 0
```

**Fix:** IAM → Security credentials for root → Delete all access keys

#### 1.9 — IAM password policy configured
Enforce a strong password policy for all IAM users.

**Fix:**
```bash
aws iam update-account-password-policy \
  --minimum-password-length 14 \
  --require-symbols \
  --require-numbers \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --allow-users-to-change-password \
  --max-password-age 90 \
  --password-reuse-prevention 24
```

> `[SCREENSHOT]` — *IAM → Account settings → Password policy showing the configured requirements (minimum length 14, require symbols, numbers, etc.)*

#### 1.14 — Hardware MFA for root
For maximum security, use a hardware MFA device (YubiKey, etc.) for root, not TOTP.

#### 1.16 — IAM policies attached only to groups or roles
No policies attached directly to users — use groups and roles.

**Check:**
```bash
aws iam generate-credential-report && \
aws iam get-credential-report --query 'Content' --output text | \
  base64 -d | grep -v "^user," | cut -d, -f1,8,9,10
# Look for users with directly attached policies
```

---

### Section 2 — Storage

#### 2.1.1 — S3 block public access (account level)
Block public access at the account level — applies to all buckets.

**Fix:**
```bash
aws s3control put-public-access-block \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

> `[SCREENSHOT]` — *S3 console → Block Public Access settings for this account page showing all four options enabled (green checkmarks)*

#### 2.1.2 — S3 versioning enabled
Versioning protects against accidental deletion and ransomware.

**Check with Config:**
- Config → Rules → `s3-bucket-versioning-enabled` → view non-compliant buckets

#### 2.2.1 — EBS default encryption enabled
Enable EBS encryption by default at the account level:

```bash
aws ec2 enable-ebs-encryption-by-default --region ap-southeast-1
```

> `[SCREENSHOT]` — *EC2 → Account attributes → EBS encryption showing "Default encryption: Enabled" with the KMS key ARN*

---

### Section 3 — Logging

#### 3.1 — CloudTrail enabled in all regions
```bash
aws cloudtrail describe-trails --query 'trailList[?IsMultiRegionTrail]'
# Should have at least one multi-region trail
```

#### 3.2 — CloudTrail log file validation enabled
```bash
aws cloudtrail describe-trails \
  --query 'trailList[*].{Name:Name,Validation:LogFileValidationEnabled}'
# LogFileValidationEnabled should be true
```

#### 3.4 — CloudTrail bucket not publicly accessible

```bash
# Check the bucket policy for the CloudTrail bucket
BUCKET=$(aws cloudtrail describe-trails --query 'trailList[0].S3BucketName' --output text)
aws s3api get-bucket-policy --bucket $BUCKET
```

#### 3.7 — Ensure CloudTrail logs are encrypted at rest

```bash
aws cloudtrail describe-trails \
  --query 'trailList[*].{Name:Name,KMSKey:KMSKeyId}'
# KMSKeyId should not be null
```

#### 3.10 — VPC flow logs enabled on all VPCs

```bash
aws ec2 describe-vpcs --query 'Vpcs[*].VpcId' --output text | \
  xargs -I{} aws ec2 describe-flow-logs \
    --filter "Name=resource-id,Values={}" \
    --query 'FlowLogs[0].FlowLogStatus'
```

> `[SCREENSHOT]` — *VPC console showing a VPC's Flow Logs tab with an active flow log entry showing the destination (CloudWatch Logs group or S3 bucket) and traffic type*

---

### Section 4 — Monitoring (CloudWatch Alarms)

The CIS benchmark requires CloudWatch metric filters and alarms for critical events. These alert you when dangerous things happen.

**Required alarms:**

| Control | Event to alarm on |
|---------|-----------------|
| 4.1 | Unauthorized API calls |
| 4.2 | Console login without MFA |
| 4.3 | Root account usage |
| 4.4 | IAM policy changes |
| 4.5 | CloudTrail configuration changes |
| 4.6 | Console authentication failures |
| 4.9 | AWS Config changes |
| 4.11 | Route table changes |
| 4.12 | VPC changes |
| 4.14 | Security group changes |

**Example — alarm on root account usage:**

```bash
# Create metric filter
aws logs put-metric-filter \
  --log-group-name "CloudTrail/logs" \
  --filter-name "RootAccountUsage" \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }' \
  --metric-transformations \
    metricName=RootAccountUsageCount,metricNamespace=CISBenchmark,metricValue=1

# Create alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "CIS-RootAccountUsage" \
  --alarm-description "Root account used" \
  --metric-name RootAccountUsageCount \
  --namespace CISBenchmark \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:security-alerts \
  --treat-missing-data notBreaching
```

> `[SCREENSHOT]` — *CloudWatch → Alarms page showing the CIS benchmark alarms listed with their states — OK (green) in normal operation, ALARM (red) if triggered*

---

### Section 5 — Networking

#### 5.1 — No unrestricted security group on port 22

```bash
aws ec2 describe-security-groups \
  --query "SecurityGroups[?IpPermissions[?FromPort==\`22\` && IpRanges[?CidrIp=='0.0.0.0/0']]].{ID:GroupId,Name:GroupName}" \
  --output table
```

**Fix:** Remove the `0.0.0.0/0` rule and replace with your specific IP or bastion host SG.

#### 5.2 — No unrestricted security group on port 3389 (RDP)

Same pattern as above — no `0.0.0.0/0` on 3389.

#### 5.3 — Default security group blocks all traffic

```bash
# Find default security groups with rules
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=default" \
  --query "SecurityGroups[?IpPermissions!=\`[]\` || IpPermissionsEgress!=\`[]\`]"
```

**Fix:** Remove all rules from the default SG and never use it.

---

## Compliance Score Dashboard

After enabling CIS in Security Hub, track your score over time:

> `[SCREENSHOT]` — *Security Hub → Summary page showing the CIS benchmark compliance score as a percentage, with a trend graph showing improvement over the past 30 days as controls were remediated*

**Target score:** 90%+ for Level 1 controls. 100% is the goal but some Level 2 controls may not apply to your environment.

---

## Mapping CIS to Other Frameworks

CIS controls map to broader frameworks — fixing CIS issues often satisfies multiple compliance requirements:

| CIS Control | SOC 2 | ISO 27001 | NIST CSF |
|-------------|-------|-----------|---------|
| Root MFA (1.1) | CC6.1 | A.9.4.2 | PR.AC-7 |
| CloudTrail enabled (3.1) | CC7.2 | A.12.4.1 | DE.CM-3 |
| Unrestricted SSH (5.1) | CC6.6 | A.13.1.3 | PR.AC-5 |
| S3 public access (2.1.1) | CC6.1 | A.13.2.1 | PR.DS-5 |
| Password policy (1.9) | CC6.1 | A.9.4.3 | PR.AC-1 |

This means improving your CIS score also improves your SOC 2 and ISO 27001 readiness.

---

## Lab — Run CIS Check and Remediate Failures

**Objective:** Enable Security Hub CIS standard, identify the top 5 failures, and fix them.

1. Enable Security Hub CIS standard if not already enabled
2. Security Hub → Security standards → CIS → view all controls
3. Sort by status → filter to FAILED

> `[SCREENSHOT]` — *Security Hub → CIS controls page filtered to FAILED showing the failing controls with their IDs, descriptions, and severity levels*

4. Pick the top 5 highest severity failures
5. For each failure — click it → read the remediation guidance → apply the fix

**Common quick wins:**
- Enable S3 block public access at account level (2.1.1) → 2 CLI commands
- Enable EBS encryption by default (2.2.1) → 1 CLI command per region
- Delete root access keys if they exist (1.4) → Console
- Ensure CloudTrail log validation is on (3.2) → 1 CLI command

6. After fixing — wait 24 hours for Security Hub to re-evaluate
7. Return to the CIS dashboard → verify score improved

> `[SCREENSHOT]` — *Security Hub CIS score before remediation (e.g., 58%) vs after (e.g., 78%) — showing the improvement after the quick wins were applied*

---

## Key Takeaways

- CIS AWS Foundations Benchmark Level 1 is your minimum baseline — implement all of it
- Security Hub automates CIS checking continuously — not just point-in-time
- Root account controls (MFA, no access keys) are the most critical — a root compromise is a full account compromise
- CloudWatch alarms for CIS monitoring events give you near-real-time detection of administrative changes
- Fixing CIS controls satisfies requirements across multiple compliance frameworks simultaneously
- Track your compliance score over time — every sprint should move it toward 90%+

---

## References

<div class="references">
<ul>
  <li><a href="https://www.cisecurity.org/benchmark/amazon_web_services" target="_blank">CIS AWS Foundations Benchmark</a></li>
  <li><a href="https://docs.aws.amazon.com/securityhub/latest/userguide/cis-aws-foundations-benchmark.html" target="_blank">Security Hub CIS Standard</a></li>
  <li><a href="https://github.com/prowler-cloud/prowler" target="_blank">Prowler — Open Source AWS Security Tool</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
