---
layout: post
title: IAM Access Analyzer, STS, and Authorization Controls
date: 2026-06-29T10:00:00
categories:
  - SCS-C03 Cloud Security Speciality
  - IAM
tags:
  - aws
  - iam
  - access-analyzer
  - sts
  - permission-boundaries
  - least-privilege
  - cloud-security
  - scs-c03
author: muhammed
description: SCS-C03 Domain 4 — IAM Access Analyzer findings, unused access, AssumeRole and session policies, permission boundaries, IAM Roles Anywhere, policy simulation, and ABAC/RBAC
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## IAM Access Analyzer

**IAM Access Analyzer** continuously monitors resource policies to identify access that is granted to external principals — outside your account or AWS organization.
It generates findings for any resource that is accessible to an entity you did not intend.

Access Analyzer also identifies **unused access** — roles, policies, and access keys that have not been used and represent unnecessary attack surface.

---

## Access Analyzer — External Access Findings

Access Analyzer examines resource-based policies on:
- S3 buckets
- KMS keys
- IAM roles (trust policies)
- SQS queues
- Lambda functions and layers
- Secrets Manager secrets
- SNS topics
- EBS volume snapshots

When any of these grants access to a principal **outside the analyzer's zone of trust** (account or organization), a finding is generated.

```bash
# Create an analyzer (zone of trust = account)
aws accessanalyzer create-analyzer \
  --analyzer-name account-analyzer \
  --type ACCOUNT

# Create an analyzer (zone of trust = entire org)
aws accessanalyzer create-analyzer \
  --analyzer-name org-analyzer \
  --type ORGANIZATION

# List findings
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:eu-west-1:123456789012:analyzer/account-analyzer

# Get a specific finding
aws accessanalyzer get-finding \
  --analyzer-arn arn:aws:access-analyzer:eu-west-1:123456789012:analyzer/account-analyzer \
  --id finding-id-here

# Archive a finding (intentional access — suppress the alert)
aws accessanalyzer update-findings \
  --analyzer-arn arn:aws:access-analyzer:eu-west-1:123456789012:analyzer/account-analyzer \
  --ids '["finding-id-here"]' \
  --status ARCHIVED
```

### Finding Status Lifecycle

| Status | Meaning |
|---|---|
| **Active** | Access exists and has not been reviewed |
| **Archived** | Reviewed — access is intentional |
| **Resolved** | The access no longer exists (policy was changed) |

---

## Access Analyzer — Unused Access

Access Analyzer also detects **unused permissions** by analysing CloudTrail logs against granted permissions.
Findings for:
- IAM roles not used in the last 90 days
- Access keys not used in the last 90 days
- IAM policies with actions never invoked
- S3 access granted but never used

```bash
# Create an unused access analyzer (requires org-level for full visibility)
aws accessanalyzer create-analyzer \
  --analyzer-name unused-access-analyzer \
  --type ORGANIZATION_UNUSED_ACCESS \
  --configuration '{
    "unusedAccess": {
      "unusedAccessAge": 90
    }
  }'

# List unused access findings
aws accessanalyzer list-findings-v2 \
  --analyzer-arn arn:aws:access-analyzer:eu-west-1:123456789012:analyzer/unused-access-analyzer \
  --filter '{"findingType": {"eq": ["UnusedIAMRole"]}}'
```

Use unused access findings to implement **least privilege** — remove permissions and roles that are never used.

---

## Access Analyzer — Policy Validation

Access Analyzer validates IAM policies against best practices and AWS policy grammar before you deploy them.

```bash
# Validate a policy document
aws accessanalyzer validate-policy \
  --policy-document file://policy.json \
  --policy-type IDENTITY_POLICY

# Generate a least-privilege policy from CloudTrail activity
aws accessanalyzer generate-policy \
  --configuration '{
    "cloudTrail": {
      "allRegions": true,
      "trailArns": ["arn:aws:cloudtrail:eu-west-1:123456789012:trail/org-trail"],
      "startTime": "2026-01-01T00:00:00Z",
      "endTime": "2026-06-01T00:00:00Z"
    }
  }' \
  --principal-arn arn:aws:iam::123456789012:role/my-role
```

---

## AWS STS — Security Token Service

**STS** issues temporary security credentials — access key ID, secret access key, and session token — with a limited lifetime.
Temporary credentials are always preferred over long-lived IAM user access keys.

### Key STS Operations

| Operation | Use Case |
|---|---|
| `AssumeRole` | Switch to another IAM role (cross-account or same account) |
| `AssumeRoleWithSAML` | Federation via SAML 2.0 (corporate IdP → AWS) |
| `AssumeRoleWithWebIdentity` | Federation via OIDC (Cognito, Google, GitHub Actions) |
| `GetSessionToken` | Get temporary credentials for an IAM user (with MFA) |
| `GetFederationToken` | Legacy — issue creds for a federated user without assuming a role |

```bash
# Assume a role (cross-account access)
aws sts assume-role \
  --role-arn arn:aws:iam::999988887777:role/cross-account-admin \
  --role-session-name audit-session \
  --duration-seconds 3600 \
  --external-id "shared-secret-external-id"

# Assume role with MFA enforcement
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/sensitive-ops \
  --role-session-name mfa-session \
  --serial-number arn:aws:iam::123456789012:mfa/user@example.com \
  --token-code 123456

# Get caller identity (who am I?)
aws sts get-caller-identity
```

### Role Trust Policy

The trust policy on the role defines who can assume it.
For cross-account access, the external account's IAM policy must also allow `sts:AssumeRole`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {"sts:ExternalId": "unique-external-id"},
        "Bool": {"aws:MultiFactorAuthPresent": "true"}
      }
    }
  ]
}
```

**ExternalId** prevents the confused deputy problem — a third-party service cannot assume your role on behalf of another customer unless it provides the unique ExternalId you shared with it.

---

## Session Policies

A **session policy** is an inline policy passed at the time of `AssumeRole`.
It can only restrict — not expand — the permissions granted by the role's identity policy.
The effective permissions are the intersection of the role's permissions AND the session policy.

```bash
# Assume a role with a restrictive session policy (limit to S3 read only)
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/data-analyst \
  --role-session-name restricted-session \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }]
  }'
```

Session policies are useful when you want to issue least-privilege credentials from a broad role — e.g., a CI/CD system that assumes a role but restricts each job to only what it needs.

---

## Permission Boundaries

A **permission boundary** is a managed policy attached to an IAM role or user that sets the maximum permissions they can ever have.
Even if the role's policies grant more, the effective permissions cannot exceed the boundary.

```
Effective permissions = Role policies ∩ Permission boundary ∩ SCPs (if in org)
```

Use permission boundaries to let developers create IAM roles without escalating their own privileges.

```bash
# Create a permission boundary (developers can only create roles with this boundary)
aws iam create-policy \
  --policy-name developer-boundary \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "s3:*", "dynamodb:*", "lambda:*", "logs:*", "cloudwatch:*"
      ],
      "Resource": "*"
    }]
  }'

# Attach boundary when creating a role
aws iam create-role \
  --role-name developer-role \
  --assume-role-policy-document file://trust.json \
  --permissions-boundary arn:aws:iam::123456789012:policy/developer-boundary
```

---

## IAM Roles Anywhere

**IAM Roles Anywhere** lets workloads running **outside AWS** (on-premises servers, CI/CD pipelines, other clouds) assume IAM roles using X.509 certificates — no long-lived access keys needed.

```
On-premises server (has X.509 cert from your CA)
    │ authenticates with cert
    ▼
IAM Roles Anywhere trust anchor (trusts your CA)
    │ issues temporary credentials
    ▼
IAM role in your account (with normal role permissions)
```

```bash
# Create a trust anchor (trust your private CA)
aws rolesanywhere create-trust-anchor \
  --name on-prem-ca-anchor \
  --source SourceType=AWS_ACM_PCA,SourceData='{
    "acmPcaArn": "arn:aws:acm-pca:eu-west-1:123456789012:certificate-authority/abc123"
  }' \
  --enabled

# Create a profile (links trust anchor to IAM roles)
aws rolesanywhere create-profile \
  --name on-prem-servers-profile \
  --role-arns '["arn:aws:iam::123456789012:role/on-prem-role"]' \
  --enabled
```

---

## IAM Policy Evaluation Logic

When a request is evaluated, AWS checks in this order:

```
1. Explicit Deny in any policy → DENIED
2. SCP (if in organization) does not allow → DENIED
3. Resource-based policy allows → ALLOWED (if no explicit deny above)
4. Identity policy allows → ALLOWED
5. Permission boundary allows → (must also be allowed by identity policy)
6. Session policy allows → (intersection with identity policy)
7. Default → DENIED
```

**Exam gotcha:** An explicit Allow in a resource-based policy can grant cross-account access even if the identity policy in the external account is restrictive — but only if the resource policy specifies the external account's principal explicitly.

---

## IAM Policy Simulator

Use the **IAM Policy Simulator** to test what a user, role, or group can do without making actual API calls.
Identifies which policy allows or denies a specific action.

```bash
# Simulate what an IAM role can do
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/my-role \
  --action-names s3:DeleteObject \
  --resource-arns arn:aws:s3:::prod-bucket/sensitive-file.txt

# Simulate a custom policy document
aws iam simulate-custom-policy \
  --policy-input-list file://policy.json \
  --action-names kms:Decrypt \
  --resource-arns arn:aws:kms:eu-west-1:123456789012:key/mrk-abc123
```

---

## S3 Presigned URLs

**Presigned URLs** grant temporary access to a specific S3 object without requiring AWS credentials.
Generated by the object owner — the URL embeds the owner's credentials and expires after a set time.

```bash
# Generate a presigned URL valid for 1 hour
aws s3 presign s3://my-bucket/report.pdf --expires-in 3600

# Presigned URL with SDK (Python example)
# boto3.client('s3').generate_presigned_url('get_object',
#   Params={'Bucket': 'my-bucket', 'Key': 'report.pdf'},
#   ExpiresIn=3600)
```

**Exam tip:** Presigned URLs inherit the permissions of the **IAM identity that generated them**.
If that IAM user or role loses access to the object (policy change), existing presigned URLs also stop working immediately.

---

## Exam Key Points

- **Access Analyzer external access**: finds resources accessible outside your account/org — archive intentional findings
- **Access Analyzer unused access**: identifies roles and policies not used in 90 days — foundation for least privilege
- **ExternalId**: prevents confused deputy — third-party must provide the shared ExternalId to assume your role
- **Session policy**: restricts (never expands) the permissions of an assumed role — effective perms = role ∩ session policy
- **Permission boundary**: caps maximum permissions — effective perms = identity policy ∩ boundary
- **SCP evaluation**: SCPs apply to all accounts in the OU (including management account if set) and cannot grant permissions, only restrict
- **IAM Roles Anywhere**: on-premises workloads use X.509 certs to get temporary AWS credentials — no static access keys
- **Presigned URLs**: issued by STS from S3 client — valid until expiry OR until the issuing identity loses access

---

## Quick Reference

```bash
# Access Analyzer
aws accessanalyzer list-analyzers
aws accessanalyzer list-findings --analyzer-arn ARN
aws accessanalyzer validate-policy --policy-document file://policy.json --policy-type IDENTITY_POLICY

# STS
aws sts get-caller-identity
aws sts assume-role --role-arn ARN --role-session-name session

# IAM Policy Simulator
aws iam simulate-principal-policy --policy-source-arn ARN --action-names s3:PutObject --resource-arns arn:aws:s3:::bucket/*

# IAM
aws iam get-account-authorization-details  # full dump of all users, roles, policies
aws iam generate-credential-report         # access key age, last used, MFA status
aws iam get-credential-report              # download the report (base64 CSV)
```
