---
layout: post
title: AWS Secrets Manager and ACM — Credential Rotation and TLS Certificates
date: 2026-06-23T10:00:00
categories:
  - SCS-C03 Cloud Security Speciality
  - Data Protection
tags:
  - aws
  - secrets-manager
  - acm
  - private-ca
  - tls
  - encryption
  - cloud-security
  - scs-c03
author: muhammed
description: SCS-C03 Domain 5 — Secrets Manager rotation, cross-account secrets, VPC endpoints, ACM certificate lifecycle, AWS Private CA hierarchy, and data-in-transit controls
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## AWS Secrets Manager

**AWS Secrets Manager** stores, rotates, and manages access to secrets — database credentials, API keys, OAuth tokens, and any sensitive string.
Unlike SSM Parameter Store (which is a general-purpose key-value store), Secrets Manager is purpose-built for secrets with native rotation support and cross-account access.

---

## Secrets Manager vs SSM Parameter Store

| | Secrets Manager | SSM Parameter Store |
|---|---|---|
| **Purpose** | Secrets with automatic rotation | General config + secrets |
| **Rotation** | Native, scheduled, Lambda-backed | Manual only |
| **Cost** | $0.40/secret/month | Free (Standard tier) |
| **Cross-account** | Yes — resource policy | Yes — with RAM or policy |
| **Encryption** | Always encrypted with KMS | SecureString = KMS, String = plaintext |
| **Versioning** | Yes (AWSCURRENT, AWSPREVIOUS, AWSPENDING) | Yes (versions, labels) |

**Exam rule:** If the question mentions automatic rotation, the answer is Secrets Manager.
SSM Parameter Store is for config values and does not rotate.

---

## Storing and Retrieving Secrets

```bash
# Store a secret
aws secretsmanager create-secret \
  --name prod/rds/db-password \
  --description "Production RDS master password" \
  --secret-string '{"username":"admin","password":"s3cur3P@ss!"}' \
  --kms-key-id arn:aws:kms:eu-west-1:123456789012:key/mrk-abc123

# Retrieve the secret value
aws secretsmanager get-secret-value \
  --secret-id prod/rds/db-password \
  --version-stage AWSCURRENT

# List secrets
aws secretsmanager list-secrets --output table

# Update (rotate manually — creates AWSPENDING version)
aws secretsmanager put-secret-value \
  --secret-id prod/rds/db-password \
  --secret-string '{"username":"admin","password":"newP@ss2026!"}'
```

---

## Automatic Rotation

Secrets Manager rotates secrets using a **Lambda function**.
For supported services (RDS, Redshift, DocumentDB, ElastiCache), AWS provides managed rotation functions.
For custom secrets, you write your own Lambda with the rotation lifecycle.

### Rotation Lifecycle (4 Steps)

```
1. createSecret  — generate new secret value, store as AWSPENDING
2. setSecret     — set the new value on the service (e.g. RDS password change)
3. testSecret    — verify the new credentials work
4. finishSecret  — promote AWSPENDING to AWSCURRENT, demote old to AWSPREVIOUS
```

```bash
# Enable automatic rotation (every 30 days)
aws secretsmanager rotate-secret \
  --secret-id prod/rds/db-password \
  --rotation-lambda-arn arn:aws:lambda:eu-west-1:123456789012:function:SecretsManagerRDSMySQLRotationSingleUser \
  --rotation-rules AutomaticallyAfterDays=30

# Force immediate rotation
aws secretsmanager rotate-secret --secret-id prod/rds/db-password

# Check rotation status
aws secretsmanager describe-secret \
  --secret-id prod/rds/db-password \
  --query '{RotationEnabled: RotationEnabled, LastRotatedDate: LastRotatedDate}'
```

**Exam gotcha:** The rotation Lambda must be in the same region as the secret OR have a VPC endpoint.
If the Lambda cannot reach Secrets Manager (no internet, no VPC endpoint), rotation fails.

---

## Cross-Account Access

Grant another account access to a secret via a **resource-based policy**.
Both the resource policy (on the secret) and the caller's IAM policy must allow the action.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccountRead",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::999988887777:role/app-role"
      },
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
# Attach the resource policy to the secret
aws secretsmanager put-resource-policy \
  --secret-id prod/rds/db-password \
  --resource-policy file://cross-account-policy.json
```

If the secret is encrypted with a CMK, the KMS key policy must also allow the external account to use the key (`kms:Decrypt`).

---

## VPC Endpoint for Secrets Manager

Applications in private subnets should use the **VPC endpoint** (interface endpoint) so traffic stays within AWS.
Without it, rotation Lambda and application code must route through NAT Gateway to reach Secrets Manager.

```bash
# Create interface endpoint for Secrets Manager
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.eu-west-1.secretsmanager \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-private1 subnet-private2 \
  --security-group-ids sg-endpoint \
  --private-dns-enabled
```

---

## AWS Certificate Manager (ACM)

**ACM** provisions, manages, and auto-renews TLS/SSL certificates for AWS services.
Certificates are free when used with integrated services.
ACM handles the renewal process automatically — no manual certificate management.

### ACM Supported Services

- Application Load Balancer
- CloudFront distributions
- API Gateway
- Elastic Beanstalk
- AppSync
- CloudFormation (for the above)

**ACM certificates cannot be exported** — you cannot download the private key.
To use a certificate on an EC2 instance directly, use ACM Private CA to issue a cert and export it, or upload a third-party certificate.

```bash
# Request a public certificate
aws acm request-certificate \
  --domain-name example.com \
  --subject-alternative-names "*.example.com" "api.example.com" \
  --validation-method DNS

# List certificates
aws acm list-certificates --certificate-statuses ISSUED

# Describe a certificate (check expiry, SANs, status)
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:eu-west-1:123456789012:certificate/abc-123

# Import a third-party certificate
aws acm import-certificate \
  --certificate fileb://cert.pem \
  --private-key fileb://private-key.pem \
  --certificate-chain fileb://chain.pem
```

---

## Certificate Validation

| Method | How it works | Use when |
|---|---|---|
| **DNS validation** | Add a CNAME record to your domain's DNS | Preferred — ACM auto-renews as long as CNAME exists |
| **Email validation** | AWS sends an email to domain contacts | Cannot modify DNS (registrar-controlled domain) |

DNS validation is always preferred.
Once the CNAME record exists, ACM renews automatically — DNS validation certificates never expire as long as the record stays in place.

---

## AWS Private Certificate Authority (Private CA)

**ACM Private CA** issues private TLS certificates for internal resources — microservices, internal APIs, EC2 instances, IoT devices.
Private CA uses your own root or subordinate CA hierarchy — certificates are trusted only by your organization.

### CA Hierarchy

```
Root CA (offline, air-gapped)
    └── Subordinate CA (issuing CA, online, in AWS)
            ├── Server certificates (for services)
            ├── Client certificates (for mTLS)
            └── Code signing certificates
```

Keep the root CA in a separate, restricted account.
Issue certificates from subordinate CAs.

```bash
# Create a private CA
aws acm-pca create-certificate-authority \
  --certificate-authority-type SUBORDINATE \
  --certificate-authority-configuration '{
    "KeyAlgorithm": "RSA_2048",
    "SigningAlgorithm": "SHA256WITHRSA",
    "Subject": {
      "Country": "US",
      "Organization": "Example Corp",
      "CommonName": "Example Corp Internal CA"
    }
  }' \
  --revocation-configuration '{
    "CrlConfiguration": {
      "Enabled": true,
      "ExpirationInDays": 7,
      "S3BucketName": "my-crl-bucket"
    }
  }'

# Issue a certificate from the private CA
aws acm-pca issue-certificate \
  --certificate-authority-arn arn:aws:acm-pca:eu-west-1:123456789012:certificate-authority/abc123 \
  --csr fileb://server.csr \
  --signing-algorithm SHA256WITHRSA \
  --validity Value=365,Type=DAYS

# Export a private certificate (unlike ACM public certs, these CAN be exported)
aws acm export-certificate \
  --certificate-arn arn:aws:acm:eu-west-1:123456789012:certificate/abc-123 \
  --passphrase $(echo -n "mypassphrase" | base64)
```

---

## Data in Transit Controls

### TLS Enforcement on ALB

Create a listener policy that requires TLS 1.2 or later and rejects older protocols.

```bash
# Create HTTPS listener with security policy
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/prod-alb/abc \
  --protocol HTTPS \
  --port 443 \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --certificates CertificateArn=arn:aws:acm:...:certificate/abc123 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/prod-tg/abc

# Redirect HTTP to HTTPS
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/prod-alb/abc \
  --protocol HTTP \
  --port 80 \
  --default-actions '[{
    "Type": "redirect",
    "RedirectConfig": {
      "Protocol": "HTTPS",
      "Port": "443",
      "StatusCode": "HTTP_301"
    }
  }]'
```

**TLS security policies — exam reference:**

| Policy | TLS versions | Use when |
|---|---|---|
| `ELBSecurityPolicy-TLS13-1-2-2021-06` | TLS 1.2 + 1.3 | Standard modern apps |
| `ELBSecurityPolicy-TLS13-1-3-2021-06` | TLS 1.3 only | Highest security requirement |
| `ELBSecurityPolicy-FS` | TLS 1.2, forward secrecy only | Compliance requiring PFS |

### S3 Bucket Policy — Require HTTPS

```json
{
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "Bool": {"aws:SecureTransport": "false"}
      }
    }
  ]
}
```

---

## CloudWatch Logs — Data Protection Policies

**CloudWatch Logs data protection policies** detect and mask sensitive data (PII, credentials, health data) in log groups.
Matches are replaced with `***PROTECTED***` in the log stream.

```bash
# Create a data protection policy on a log group
aws logs put-data-protection-policy \
  --log-group-identifier arn:aws:logs:eu-west-1:123456789012:log-group:/app/prod-api \
  --policy-document '{
    "Name": "data-protection-policy",
    "DataIdentifiers": [
      "arn:aws:dataprotection::aws:data-identifier/EmailAddress",
      "arn:aws:dataprotection::aws:data-identifier/CreditCardNumber",
      "arn:aws:dataprotection::aws:data-identifier/AwsSecretKey"
    ],
    "Configuration": {
      "CustomDataIdentifiers": [],
      "ManagedDataIdentifiers": [
        {"Identifier": "EmailAddress"},
        {"Identifier": "CreditCardNumber"},
        {"Identifier": "AwsSecretKey"}
      ]
    },
    "Statement": [{
      "Sid": "audit-and-mask",
      "DataIdentifier": ["EmailAddress", "CreditCardNumber", "AwsSecretKey"],
      "Operation": {
        "Audit": {
          "FindingsDestination": {
            "S3": {"Bucket": "my-findings-bucket"}
          }
        },
        "Deidentify": {
          "MaskConfig": {}
        }
      }
    }]
  }'
```

---

## Exam Key Points

- **Secrets Manager vs Parameter Store**: Secrets Manager for automatic rotation; Parameter Store for config + secrets without rotation
- **Rotation failure**: most common cause is the Lambda function cannot reach Secrets Manager — add a VPC endpoint or NAT Gateway
- **Cross-account secrets**: resource policy on the secret + IAM policy in the external account + KMS key policy (if CMK is used)
- **ACM certificates**: free, auto-renew, cannot be exported — only work with AWS-integrated services
- **Private CA**: issues private certs that CAN be exported; use for internal services, mTLS, IoT
- **TLS on ALB**: use `ELBSecurityPolicy-TLS13-1-2-2021-06` for TLS 1.2+1.3; redirect HTTP 80 to HTTPS 443
- **S3 HTTPS enforcement**: bucket policy with `"aws:SecureTransport": "false"` → Deny

---

## Quick Reference

```bash
# Secrets Manager
aws secretsmanager list-secrets
aws secretsmanager get-secret-value --secret-id prod/rds/db-password
aws secretsmanager describe-secret --secret-id prod/rds/db-password
aws secretsmanager rotate-secret --secret-id prod/rds/db-password

# ACM
aws acm list-certificates --certificate-statuses ISSUED
aws acm describe-certificate --certificate-arn ARN
aws acm renew-certificate --certificate-arn ARN  # trigger manual renewal check

# Private CA
aws acm-pca list-certificate-authorities
aws acm-pca describe-certificate-authority --certificate-authority-arn ARN
aws acm-pca get-certificate-authority-csr --certificate-authority-arn ARN
```
