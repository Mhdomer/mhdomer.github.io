---
layout: post
title: AWS CLI & IAM — Credentials, Roles, Policies, and Attack Paths
date: 2026-05-09T10:00:00
categories:
  - AWS Learning Path
tags:
  - aws
  - iam
  - cloud
  - cloud-security
  - devsecops
author: muhammed
description: A practical guide to AWS CLI setup, IAM identities, credential types, policy structure, and how attackers abuse misconfigured IAM — written from a cloud security perspective
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## What is IAM?

IAM (Identity and Access Management) is the AWS service that controls **who** can do **what** on **which** AWS resources.
Every API call made to AWS — whether from the console, CLI, SDK, or a Lambda function — is authenticated and authorised through IAM.
Getting IAM wrong is the single most common cause of cloud security incidents.

---

## AWS CLI Setup

### Installation

```bash
# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# macOS
brew install awscli

# Verify
aws --version
```

### Configuration

```bash
aws configure                          # interactive setup — asks for 4 values
aws configure --profile dev            # create a named profile
aws configure list                     # show current config
aws configure list-profiles            # list all profiles
```

When you run `aws configure` it stores credentials in two files:

```
~/.aws/credentials   ← access keys (sensitive — never commit this)
~/.aws/config        ← region, output format, profiles
```

**`~/.aws/credentials`**

```ini
[default]
aws_access_key_id     = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[dev]
aws_access_key_id     = AKIAI44QH8DHBEXAMPLE
aws_secret_access_key = je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY
```

**`~/.aws/config`**

```ini
[default]
region = eu-west-1
output = json

[profile dev]
region = us-east-1
output = table
```

### Using profiles

```bash
aws s3 ls --profile dev                # use the dev profile for one command
export AWS_PROFILE=dev                 # set default profile for the session
```

### Output formats

```bash
aws ec2 describe-instances --output json    # default — machine-readable
aws ec2 describe-instances --output table  # human-readable table
aws ec2 describe-instances --output text   # plain text, good for scripting
aws ec2 describe-instances --output yaml   # YAML format
```

### Querying with --query (JMESPath)

```bash
# Get only instance IDs
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text

# Get instance ID and state
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' \
  --output table

# Filter running instances only
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text
```

---

## IAM Identities

There are four types of IAM identity.
Each one is used differently and has different security implications.

### 1. IAM Users

A **user** is a long-term identity for a person or application.
Users can have two types of credentials: a **password** (for console access) and **access keys** (for CLI/API access).
Access keys are the most common credential type found in breaches — they get hardcoded in code, committed to GitHub, or left in CI/CD logs.

```bash
aws iam list-users                              # list all IAM users
aws iam get-user --user-name alice              # get info about a specific user
aws iam list-access-keys --user-name alice      # list access keys for a user
aws iam create-user --user-name bob             # create a user
aws iam delete-user --user-name bob             # delete a user
```

### 2. IAM Groups

A **group** is a collection of users.
You attach policies to groups rather than individual users, which makes permission management much easier at scale.
A user can belong to multiple groups and inherits the combined permissions.

```bash
aws iam list-groups                             # list all groups
aws iam add-user-to-group --user-name alice --group-name Developers
aws iam remove-user-from-group --user-name alice --group-name Developers
aws iam list-groups-for-user --user-name alice  # groups a user belongs to
```

### 3. IAM Roles

A **role** is a temporary identity that can be **assumed** by a trusted entity.
Roles have no long-term credentials — when assumed, AWS issues short-lived temporary credentials (valid 15 minutes to 12 hours).
Roles are used by EC2 instances, Lambda functions, other AWS services, cross-account access, and federated users.

This is the **recommended** pattern for giving AWS resources access to other AWS services — never hardcode access keys in an EC2 instance when you can attach a role.

```bash
aws iam list-roles                              # list all roles
aws iam get-role --role-name MyRole             # get role details
aws sts assume-role \
  --role-arn arn:aws:iam::123456789:role/MyRole \
  --role-session-name mysession                 # assume a role manually
```

When you assume a role, you get back three temporary values:

```json
{
  "Credentials": {
    "AccessKeyId":     "ASIAIOSFODNN7EXAMPLE",
    "SecretAccessKey": "wJalrXUtnFEMI...",
    "SessionToken":    "AQoDYXdzEJr...",
    "Expiration":      "2026-05-09T14:00:00Z"
  }
}
```

You then export all three to use them:

```bash
export AWS_ACCESS_KEY_ID=ASIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI...
export AWS_SESSION_TOKEN=AQoDYXdzEJr...
```

### 4. Service-Linked Roles

These are pre-built roles created by AWS services automatically.
You cannot edit their trust policy, and they are deleted when the service is removed.
Examples: `AWSServiceRoleForEC2Spot`, `AWSServiceRoleForECS`.

---

## IAM Policies

A **policy** is a JSON document that defines permissions.
It lists which **actions** are allowed or denied on which **resources** under which **conditions**.
IAM uses a **default deny** model — everything is denied unless explicitly allowed.

### Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

| Field | Purpose |
|---|---|
| `Version` | Always `"2012-10-17"` — the policy language version |
| `Statement` | Array of permission blocks |
| `Sid` | Optional statement ID for human readability |
| `Effect` | `Allow` or `Deny` |
| `Action` | AWS API actions (e.g. `s3:GetObject`, `ec2:*`) |
| `Resource` | ARN of the resource the action applies to |
| `Condition` | Optional conditions (IP address, MFA, time, tags) |

### Wildcards in Actions and Resources

```json
"Action": "s3:*"                         // all S3 actions
"Action": "s3:Get*"                      // all S3 Get actions
"Resource": "*"                          // all resources (dangerous)
"Resource": "arn:aws:s3:::my-bucket/*"  // all objects in a specific bucket
```

### Conditions

Conditions restrict when a policy applies.

```json
"Condition": {
  "Bool": {
    "aws:MultiFactorAuthPresent": "true"      // only if MFA was used
  }
}

"Condition": {
  "IpAddress": {
    "aws:SourceIp": ["192.168.1.0/24"]        // only from this IP range
  }
}

"Condition": {
  "StringEquals": {
    "aws:RequestedRegion": "eu-west-1"        // only in this region
  }
}
```

### Types of Policies

| Type | Attached to | Notes |
|---|---|---|
| **AWS Managed** | Users, groups, roles | Pre-built by AWS, cannot edit |
| **Customer Managed** | Users, groups, roles | You create and own these |
| **Inline** | Single user/group/role | Embedded directly, not reusable |
| **Resource-based** | S3 buckets, SQS, Lambda, etc. | Attached to the resource, not the identity |
| **SCPs** | AWS Organisations OUs | Set maximum permissions for an entire account |
| **Permission boundaries** | Users or roles | Cap what a user/role can do |

### Trust Policies (for Roles)

A **trust policy** defines who is allowed to assume a role.
It is a resource-based policy attached to the role itself.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"         // EC2 can assume this role
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```json
"Principal": {
  "AWS": "arn:aws:iam::123456789012:user/alice"  // specific IAM user
}

"Principal": {
  "AWS": "arn:aws:iam::999999999999:root"        // entire external account
}

"Principal": {
  "Federated": "cognito-identity.amazonaws.com"  // federated identity
}
```

---

## Credential Types and Priority

When the AWS CLI or SDK looks for credentials, it checks in this order:

| Priority | Source | Notes |
|---|---|---|
| 1 | CLI flags `--profile` / env vars | Highest priority |
| 2 | `AWS_ACCESS_KEY_ID` env var | Overrides everything below |
| 3 | `~/.aws/credentials` file | Default and named profiles |
| 4 | `~/.aws/config` file | |
| 5 | EC2 instance profile / ECS task role | Metadata service |
| 6 | IAM role for web identity (OIDC) | GitHub Actions, K8s |

**Environment variables:**

```bash
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...          # required for temporary credentials
export AWS_DEFAULT_REGION=eu-west-1
export AWS_PROFILE=dev
```

### Checking who you are

```bash
aws sts get-caller-identity           # always run this first — shows account, user/role ARN
```

Output:

```json
{
  "UserId": "AIDAIOSFODNN7EXAMPLE",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/alice"
}
```

---

## Useful IAM Commands

```bash
# Who am I?
aws sts get-caller-identity

# List everything about a user
aws iam list-attached-user-policies --user-name alice
aws iam list-user-policies --user-name alice         # inline policies
aws iam list-groups-for-user --user-name alice

# Get a policy document
aws iam get-policy --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
aws iam get-policy-version \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess \
  --version-id v1

# List roles and their trust policies
aws iam list-roles --query 'Roles[*].[RoleName,Arn]' --output table
aws iam get-role --role-name MyRole

# Simulate whether an action is allowed
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/alice \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/*

# Generate an IAM credential report (all users + key age + MFA status)
aws iam generate-credential-report
aws iam get-credential-report --query 'Content' --output text | base64 -d
```

---

## IAM Attack Paths

This section covers how attackers abuse misconfigured IAM.
Understanding these patterns is essential for both offensive security (OSCP, cloud pentesting) and defensive posture (threat modelling, IAM hardening).

### 1. Leaked Access Keys

The most common entry point.
Access keys get committed to GitHub, left in Docker images, stored in `.env` files, or printed in CI/CD logs.
Tools like `trufflehog`, `gitleaks`, and `git-secrets` scan for them.

Once an attacker has keys:

```bash
aws sts get-caller-identity           # find out what account and identity
aws iam list-attached-user-policies --user-name <user>   # what can I do?
aws iam list-user-policies --user-name <user>
aws iam simulate-principal-policy ... # test specific actions
```

**Defence:** rotate keys immediately, never store keys in code, use roles instead of users for applications.

### 2. Overpermissioned Roles

EC2 instances, Lambda functions, and ECS tasks can have IAM roles attached.
If that role has `AdministratorAccess` or broad `*` permissions, anyone who can exec into the container or run code on the instance inherits those permissions.

An attacker with a foothold on an EC2 instance queries the metadata service:

```bash
# IMDSv1 (vulnerable — no auth required)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/MyRole
```

This returns temporary credentials that can be used directly.

**Defence:** enforce IMDSv2 (requires a session token), apply least-privilege to instance roles.

### 3. IMDSv1 vs IMDSv2

| | IMDSv1 | IMDSv2 |
|---|---|---|
| Authentication | None — any HTTP GET works | Requires a session token (PUT first) |
| SSRF risk | High — any SSRF reaches it | Mitigated — requires a PUT request first |
| Enforcement | Default on older instances | Must be explicitly required |

**Require IMDSv2 on an existing instance:**

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-1234567890abcdef0 \
  --http-tokens required \
  --http-endpoint enabled
```

**Fetch credentials using IMDSv2 (the correct way):**

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

### 4. Privilege Escalation via IAM

If a user has certain IAM permissions, they can escalate to admin even without having `AdministratorAccess` directly.

Common escalation paths:

| Permission | Escalation method |
|---|---|
| `iam:CreatePolicyVersion` | Create a new version of an existing policy with `*` permissions |
| `iam:SetDefaultPolicyVersion` | Switch to a permissive policy version |
| `iam:AttachUserPolicy` | Attach `AdministratorAccess` to yourself |
| `iam:CreateAccessKey` | Create new access keys for another user |
| `iam:PassRole` + `ec2:RunInstances` | Launch EC2 with an admin role, exec into it |
| `iam:PassRole` + `lambda:CreateFunction` | Create a Lambda with an admin role and invoke it |
| `sts:AssumeRole` | Assume a more permissive role |

**Tool:** [Pacu](https://github.com/RhinoSecurityLabs/pacu) — AWS exploitation framework that automates privilege escalation enumeration.

### 5. Confused Deputy via Resource-Based Policies

A confused deputy attack happens when a service with broad permissions is tricked into acting on behalf of an attacker.
In AWS, this can occur with Lambda, S3 bucket policies, or cross-account role assumptions without a proper `ExternalId` condition.

**Defence:** always include an `ExternalId` condition in cross-account trust policies.

```json
"Condition": {
  "StringEquals": {
    "sts:ExternalId": "unique-customer-id-abc123"
  }
}
```

### 6. S3 Public Bucket + Role Escalation

A misconfigured public S3 bucket might expose application code, Terraform state files, or CloudFormation templates.
These files often contain hardcoded ARNs, role names, or even credentials.
An attacker reads these files to map the account structure before escalating.

**Defence:** block public access at the account level using S3 Block Public Access settings, and scan Terraform state for secrets before storing it.

---

## IAM Security Best Practices

```
✅  Enable MFA for all IAM users, especially root
✅  Never use the root account for day-to-day work
✅  Use IAM roles for applications — never hardcode access keys
✅  Apply least privilege — start with no permissions, add what's needed
✅  Set a permissions boundary on developer-created roles
✅  Enable CloudTrail to log all API calls
✅  Rotate access keys every 90 days
✅  Use AWS Config rules to detect policy violations continuously
✅  Require IMDSv2 on all EC2 instances
✅  Use AWS Organizations SCPs to enforce guardrails across accounts
✅  Review the IAM credential report regularly for inactive keys
```

---

## Quick Reference

```bash
# Identity
aws sts get-caller-identity

# Users
aws iam list-users
aws iam list-attached-user-policies --user-name alice
aws iam list-access-keys --user-name alice

# Roles
aws iam list-roles
aws iam get-role --role-name MyRole
aws sts assume-role --role-arn arn:... --role-session-name s

# Policies
aws iam list-policies --scope Local          # customer-managed only
aws iam get-policy-version --policy-arn arn:... --version-id v1

# Simulate permissions
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/alice \
  --action-names s3:PutObject \
  --resource-arns arn:aws:s3:::my-bucket/*

# Credential report
aws iam generate-credential-report
aws iam get-credential-report --query 'Content' --output text | base64 -d

# Require IMDSv2
aws ec2 modify-instance-metadata-options \
  --instance-id i-xxx \
  --http-tokens required
```
