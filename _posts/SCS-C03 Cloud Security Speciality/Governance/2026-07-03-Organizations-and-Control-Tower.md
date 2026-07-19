---
layout: post
title: AWS Organizations and Control Tower — Multi-Account Governance
date: 2026-07-03T10:00:00
categories:
  - SCS-C03 Cloud Security Speciality
  - Governance
tags:
  - aws
  - organizations
  - control-tower
  - scp
  - rcp
  - governance
  - cloud-security
  - scs-c03
author: muhammed
description: SCS-C03 Domain 6 — AWS Organizations OU structure, SCPs, RCPs, delegated administrators, Control Tower landing zone, Account Factory, guardrails, and root user management
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## AWS Organizations

**AWS Organizations** is the service for centrally managing multiple AWS accounts.
Instead of managing each account independently, you create a hierarchy of **Organizational Units (OUs)** and apply policies from the top down.

### Key Concepts

| Term | Meaning |
|---|---|
| **Management account** | The root account that owns the organization — cannot be restricted by SCPs |
| **Member account** | Any other account in the org |
| **OU (Organizational Unit)** | A container for accounts — policies applied to an OU apply to all accounts in it |
| **Root** | The top of the organization hierarchy — policies here apply to all accounts |
| **SCP** | Service Control Policy — restricts what member accounts can do |
| **RCP** | Resource Control Policy — restricts what resources in member accounts can accept |
| **Delegated administrator** | A member account granted admin rights for a specific AWS service |

---

## Recommended OU Structure

```
Root
├── Management
│   └── management account (billing, org admin)
├── Security
│   ├── log-archive account (CloudTrail, Config, VPC Flow Logs)
│   └── security-tooling account (GuardDuty admin, Security Hub admin, Inspector admin)
├── Infrastructure
│   └── shared-services account (Transit Gateway, DNS, AD)
├── Workloads
│   ├── Prod OU
│   │   └── prod-account-1, prod-account-2
│   ├── NonProd OU
│   │   └── dev-account-1, staging-account-1
│   └── Sandbox OU
│       └── developer sandbox accounts
└── Exceptions
    └── accounts that need non-standard policies
```

---

## Service Control Policies (SCPs)

**SCPs** define the maximum permissions available to accounts and users within an OU.
SCPs do not grant permissions — they restrict what the account's IAM policies can allow.
They apply to all principals in the member account (IAM users, roles, root user) **except the management account**.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLeaveOrganization",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    },
    {
      "Sid": "DenyRegionsOutsideEU",
      "Effect": "Deny",
      "NotAction": [
        "iam:*", "sts:*", "support:*", "budgets:*",
        "cloudfront:*", "route53:*", "s3:ListAllMyBuckets",
        "waf:*", "wafv2:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["eu-west-1", "eu-west-2", "eu-central-1"]
        }
      }
    },
    {
      "Sid": "RequireS3Encryption",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyRootUser",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
```

```bash
# Create an SCP
aws organizations create-policy \
  --name deny-regions-outside-eu \
  --type SERVICE_CONTROL_POLICY \
  --description "Block all regions except EU" \
  --content file://scp-deny-non-eu.json

# Attach an SCP to an OU
aws organizations attach-policy \
  --policy-id p-abc123 \
  --target-id ou-abc123

# List SCPs attached to an OU
aws organizations list-policies-for-target \
  --target-id ou-abc123 \
  --filter SERVICE_CONTROL_POLICY
```

---

## Resource Control Policies (RCPs)

**RCPs** are a newer policy type that restrict what **resources** in member accounts can accept, regardless of the identity making the request.
Where SCPs restrict what identities can do, RCPs restrict what resources can accept.

Common RCP use case — prevent S3 buckets from accepting requests from outside the organization:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireOrgAccessForS3",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalOrgID": "o-abc123xyz"
        },
        "BoolIfExists": {
          "aws:PrincipalIsAWSService": "false"
        }
      }
    }
  ]
}
```

RCPs are attached to OUs and Root just like SCPs.

---

## Delegated Administrator Accounts

**Delegated administrators** allow you to manage org-wide security services from a member account instead of the management account.
This follows the principle that the management account should be used as little as possible.

```bash
# Enable security services at the org level
# GuardDuty
aws guardduty enable-organization-admin-account --admin-account-id 111122223333

# Security Hub
aws securityhub enable-organization-admin-account --admin-account-id 111122223333

# Inspector
aws inspector2 enable-delegated-admin-account --delegated-admin-account-id 111122223333

# Config aggregator
aws organizations register-delegated-administrator \
  --account-id 111122223333 \
  --service-principal config.amazonaws.com

# List all delegated admins
aws organizations list-delegated-administrators
```

Best practice: one dedicated **security-tooling account** is the delegated admin for all security services.

---

## Centralized Root User Management

Since 2024, Organizations supports **centralized root access management** — you can disable root user credentials in all member accounts from the management account.

```bash
# Enable centralized root access management
aws iam enable-organizations-root-credentials-management

# Enable root sessions (required before centralized management)
aws iam enable-organizations-root-sessions

# List member accounts where root is active
aws iam list-organizations-features
```

Best practice: disable root user credentials in all member accounts, and manage the management account root with hardware MFA stored in a physical safe.

---

## AWS Control Tower

**AWS Control Tower** builds a multi-account landing zone with a pre-configured OU structure, baseline guardrails, centralized logging, and account vending.
It orchestrates Organizations, IAM Identity Center, Config, CloudTrail, and Security Hub into a ready-to-use foundation.

### Landing Zone Components

| Component | What it creates |
|---|---|
| **Log Archive account** | CloudTrail org trail, Config recordings stored in a locked S3 bucket |
| **Audit account** | Security read-only access for the security team |
| **IAM Identity Center** | Pre-configured SSO for AWS accounts |
| **Mandatory guardrails** | SCPs that enforce baseline security controls |

### Guardrails

Guardrails are pre-packaged governance rules — either **preventive** (SCPs) or **detective** (Config rules).

| Type | Mechanism | Examples |
|---|---|---|
| **Mandatory** | Always enforced — cannot disable | Disallow changes to CloudTrail, require log encryption |
| **Strongly recommended** | Optional but recommended | Enable MFA for root, enable EBS encryption |
| **Elective** | Optional — opt-in for specific OUs | Disallow public S3 buckets, restrict EC2 instance types |

```bash
# List guardrails
aws controltower list-controls

# Enable an elective guardrail on an OU
aws controltower enable-control \
  --control-identifier arn:aws:controltower:eu-west-1::control/AWS-GR_S3_BUCKET_PUBLIC_READ_PROHIBITED \
  --target-identifier arn:aws:organizations::123456789012:ou/o-abc123/ou-abc-123

# Check guardrail compliance
aws controltower get-control-operation \
  --operation-identifier op-abc123
```

---

## Account Factory

**Account Factory** is Control Tower's account vending machine — provision new AWS accounts with pre-configured baselines from a self-service portal or API.
Accounts are automatically enrolled in the landing zone, have SSO configured, and inherit guardrails from their OU.

```bash
# Create an account via Account Factory (Account Factory for Terraform uses Terraform)
# Or via Service Catalog (Account Factory creates a Service Catalog product):
aws servicecatalog provision-product \
  --product-id prod-abc123 \
  --provisioning-artifact-id pa-abc123 \
  --provisioned-product-name new-team-dev-account \
  --provisioning-parameters \
    Key=AccountName,Value=team-x-dev \
    Key=AccountEmail,Value=aws+team-x-dev@company.com \
    Key=ManagedOrganizationalUnit,Value=NonProd \
    Key=SSOUserEmail,Value=admin@team-x.com \
    Key=SSOUserFirstName,Value=Admin \
    Key=SSOUserLastName,Value=User
```

---

## AI Service Opt-Out Policies

**AI opt-out policies** prevent AWS AI services (Rekognition, Comprehend, Transcribe, etc.) from using your data to improve their models.
Apply at the root to cover all member accounts.

```json
{
  "Version": "2012-10-17",
  "services": {
    "default": {
      "opt_out_policy": {
        "@@assign": "optOut"
      }
    }
  }
}
```

```bash
aws organizations create-policy \
  --name ai-opt-out \
  --type AISERVICES_OPT_OUT_POLICY \
  --content file://ai-opt-out-policy.json

aws organizations attach-policy \
  --policy-id p-abc123 \
  --target-id r-abc123  # root
```

---

## Exam Key Points

- **SCPs restrict, do not grant**: even `Allow *` in an SCP does not grant permissions — it just does not restrict; IAM policies still determine what is allowed
- **Management account**: SCPs never apply to the management account — it is unrestricted
- **RCPs restrict resources**: any external principal calling an S3 API on a bucket in your org can be blocked via RCP regardless of the bucket's own bucket policy
- **Delegated administrator**: security services (GuardDuty, Security Hub, Inspector) should be managed from a dedicated security account, not the management account
- **Control Tower guardrails**: mandatory (always on), strongly recommended (opt-out), elective (opt-in) — preventive via SCP, detective via Config
- **Account Factory**: automated account provisioning with OU assignment, SSO, and baseline compliance
- **Centralized root management**: disable root credentials in all member accounts from the management account

---

## Quick Reference

```bash
# Organizations
aws organizations describe-organization
aws organizations list-accounts --output table
aws organizations list-organizational-units-for-parent --parent-id r-abc123
aws organizations list-policies --filter SERVICE_CONTROL_POLICY

# Attach/detach policies
aws organizations attach-policy --policy-id p-abc123 --target-id ou-abc-123
aws organizations detach-policy --policy-id p-abc123 --target-id ou-abc-123

# Control Tower
aws controltower list-landing-zones
aws controltower list-controls --output table
aws controltower get-landing-zone --landing-zone-identifier lz-abc123
```
