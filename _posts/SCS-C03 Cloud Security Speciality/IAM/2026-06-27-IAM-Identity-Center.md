---
layout: post
title: AWS IAM Identity Center — SSO, Permission Sets, and Federated Access
date: 2026-06-27T10:00:00
categories:
  - SCS-C03 Cloud Security Speciality
  - IAM
tags:
  - aws
  - iam
  - sso
  - identity-center
  - saml
  - abac
  - cloud-security
  - scs-c03
author: muhammed
description: SCS-C03 Domain 4 — IAM Identity Center permission sets, SCIM provisioning, ABAC with attributes, IdP federation, multi-account access, and troubleshooting authentication
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## What is IAM Identity Center?

**IAM Identity Center** (formerly AWS SSO) is the centralized identity and access management service for multi-account AWS environments.
It provides a single sign-on portal where users log in once and access any AWS account or application they are authorized for.
IAM Identity Center integrates with AWS Organizations — you manage access across all accounts from one place.

Use IAM Identity Center when:
- You manage multiple AWS accounts and want centralized access control
- You have an external identity provider (Okta, Entra ID, Google Workspace) and need federation
- You want to avoid long-lived IAM users with access keys across accounts
- You need centralized audit of who accessed which account at what time

---

## Core Concepts

| Term | Meaning |
|---|---|
| **Instance** | The IAM Identity Center deployment (one per org, in the management or delegated admin account) |
| **Identity source** | Where users and groups come from — IAM Identity Center directory, Active Directory, or external IdP |
| **Permission set** | A collection of IAM policies applied to a user/group in an account — analogous to an IAM role |
| **Account assignment** | Binding a user/group to a permission set in a specific account |
| **User portal** | The web URL where users sign in and see their assigned accounts |

---

## Identity Sources

| Source | Description |
|---|---|
| **IAM Identity Center directory** | Built-in directory — create users and groups directly in Identity Center |
| **Active Directory** | AWS Managed Microsoft AD or AD Connector for on-premises AD |
| **External IdP (SAML 2.0)** | Okta, Azure AD, Google Workspace, Ping, any SAML 2.0 provider |

When using an external IdP, you configure:
1. SAML 2.0 trust between the IdP and IAM Identity Center
2. SCIM provisioning to sync users and groups automatically

---

## SCIM Provisioning

**SCIM (System for Cross-domain Identity Management)** automatically syncs users and groups from your IdP to IAM Identity Center.
Without SCIM, you must manually create users — with SCIM, provisioning and deprovisioning happen automatically when the IdP changes.

```
IdP (Okta / Entra ID)
    │ SCIM 2.0 provisioning
    ▼
IAM Identity Center (synced users and groups)
    │ SAML 2.0 authentication (at login)
    ▼
AWS accounts (via permission sets)
```

Configure SCIM in the IdP by:
1. Getting the IAM Identity Center SCIM endpoint URL and bearer token
2. Entering these in the IdP's provisioning configuration
3. Mapping IdP groups to IAM Identity Center groups

---

## Permission Sets

A **permission set** defines what IAM permissions a user gets in a specific account.
Permission sets are stored in IAM Identity Center and provisioned as IAM roles in each account they are assigned to.

### Types of Permission Sets

| Type | How defined |
|---|---|
| **AWS managed policies** | Attach existing managed policies (e.g. `AdministratorAccess`, `ReadOnlyAccess`) |
| **Inline policy** | Custom JSON policy embedded in the permission set |
| **Customer managed policies** | Reference a customer-managed policy that must exist in the target account |
| **Permissions boundary** | Attach a boundary to limit maximum permissions even if the permission set grants more |

```bash
# Create a permission set
aws sso-admin create-permission-set \
  --instance-arn arn:aws:sso:::instance/ssoins-abc123 \
  --name SecurityAudit \
  --description "Read-only security audit access" \
  --session-duration PT4H

# Attach an AWS managed policy to the permission set
aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn arn:aws:sso:::instance/ssoins-abc123 \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-abc123/ps-abc \
  --managed-policy-arn arn:aws:iam::aws:policy/SecurityAudit

# Assign a group to the permission set in an account
aws sso-admin create-account-assignment \
  --instance-arn arn:aws:sso:::instance/ssoins-abc123 \
  --target-id 111122223333 \
  --target-type AWS_ACCOUNT \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-abc123/ps-abc \
  --principal-type GROUP \
  --principal-id group-id-here

# List all account assignments for a permission set
aws sso-admin list-account-assignments \
  --instance-arn arn:aws:sso:::instance/ssoins-abc123 \
  --account-id 111122223333 \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-abc123/ps-abc
```

---

## Attribute-Based Access Control (ABAC)

**ABAC** in IAM Identity Center uses user attributes from the identity source as tags on the assumed IAM role session.
These session tags can then be used in IAM policy conditions — enabling dynamic access control without assigning separate permission sets per team or project.

### How ABAC Works with IAM Identity Center

1. User in Okta has attribute `Department = security`
2. SCIM syncs the user with that attribute to Identity Center
3. Enable **attribute mappings** in Identity Center to pass `Department` as a session tag `PrincipalTag/Department`
4. IAM policy uses condition `aws:PrincipalTag/Department` to control access

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::team-data/${aws:PrincipalTag/Department}/*"
}
```

This policy lets users access only the S3 prefix matching their department tag — without any policy change when a user changes teams in the IdP.

```bash
# Enable attribute mappings (console required for full setup, CLI for viewing)
aws sso-admin list-instances
# In console: Identity Center → Attributes for access control → Enable, map IdP attributes to session tags
```

---

## How Users Access Accounts

When a user selects an account from the portal, IAM Identity Center issues **temporary credentials** by assuming the provisioned IAM role in that account.
The role has a trust policy allowing `sso.amazonaws.com` to assume it.

```bash
# Users can also use the CLI with automatic credential retrieval
aws configure sso
# Prompts: SSO start URL, SSO region, account, permission set
# Creates a named profile in ~/.aws/config

# Use the profile
aws s3 ls --profile my-sso-profile
```

CloudTrail logs show the assumed role ARN with the user's identity as the session context:
```
userIdentity.type: AssumedRole
userIdentity.sessionContext.sessionIssuer.type: Role
userIdentity.sessionContext.webIdFederationData.federatedIdentity: user@example.com
```

---

## Multi-Account Strategy with Permission Sets

A typical multi-account setup has permission sets covering:

| Permission Set | Users | Accounts |
|---|---|---|
| `AdministratorAccess` | Cloud ops team | Management, log archive |
| `SecurityAudit` | Security team | All accounts (aggregated) |
| `DeveloperAccess` | Dev team | Dev, staging accounts |
| `ReadOnly` | All engineers | Prod (view only) |
| `BreakGlass` | On-call leads | All accounts (MFA enforced) |

Keep break-glass accounts documented but access rarely assigned — grant via emergency process.

---

## Troubleshooting Authentication

| Problem | Where to look |
|---|---|
| User cannot see account in portal | Check account assignment — user or their group must be assigned to the permission set in that account |
| User gets access denied after login | Check the permission set's policies — use IAM Policy Simulator with the assumed role ARN |
| SCIM not syncing | Check IdP provisioning logs, verify SCIM endpoint URL and token, check IAM Identity Center provisioning errors |
| Session expires too quickly | Default session is 1 hour — increase session duration on the permission set (max 12 hours) |
| MFA not prompted | Check authentication flow settings in Identity Center — enable MFA required for each user or all users |

```bash
# Check provisioning errors
aws identitystore list-users \
  --identity-store-id d-abc123 \
  --filters AttributePath=UserName,AttributeValue=user@example.com

# List groups a user belongs to
aws identitystore list-group-memberships-for-member \
  --identity-store-id d-abc123 \
  --member-id UserId=user-id-here

# Check what accounts a user is assigned to
aws sso-admin list-account-assignments-for-principal \
  --instance-arn arn:aws:sso:::instance/ssoins-abc123 \
  --principal-type USER \
  --principal-id user-id-here
```

---

## Exam Key Points

- **Identity Center vs IAM users**: Identity Center for centralized multi-account access; IAM users for single-account programmatic access only
- **Permission sets become IAM roles**: each permission set assignment creates an IAM role in the target account with a trust policy for `sso.amazonaws.com`
- **SCIM**: automatic user/group sync from IdP — deprovisioning in IdP automatically removes access in AWS
- **ABAC with session tags**: attributes from IdP → session tags → IAM conditions — scales access control without per-team permission sets
- **SAML 2.0**: used for authentication (user login); SCIM is used for provisioning (user/group sync) — they are separate
- **Session duration**: configurable per permission set, default 1 hour, max 12 hours
- **CloudTrail**: all Identity Center logins and role assumptions are logged — user identity visible in `webIdFederationData`

---

## Quick Reference

```bash
# List instances
aws sso-admin list-instances

# List permission sets
aws sso-admin list-permission-sets \
  --instance-arn arn:aws:sso:::instance/ssoins-abc123

# List account assignments across org
aws sso-admin list-accounts-for-provisioned-permission-set \
  --instance-arn arn:aws:sso:::instance/ssoins-abc123 \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-abc123/ps-abc

# Get user in identity store
aws identitystore describe-user \
  --identity-store-id d-abc123 \
  --user-id user-id-here
```
