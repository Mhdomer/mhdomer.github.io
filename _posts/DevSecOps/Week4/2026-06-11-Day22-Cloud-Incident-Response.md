---
layout: post
title: "Week 4 — Day 22: Cloud Incident Response"
date: 2026-06-11 10:00:00 +0800
categories:
  - DevSecOps
  - Week4
tags:
  - IncidentResponse
  - AWS
  - CloudSecurity
  - Forensics
  - DevSecOps
author: muhammed
description: A full walkthrough of cloud incident response on AWS — the IR lifecycle, detecting and investigating a compromised IAM key or EC2 instance, containment steps, and building a runbook.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fwww.cyberforensicacademy.com%2Fblog%2Fuploads%2Fimages%2F202512%2Fimage_870x_693baf0cd224c.webp&f=1&nofb=1&ipt=8a30ad2368263679db44fe28941c4a22dadfd01f6f70bc6c157075f8b8e60c2c
---

## Why Cloud IR Is Different

Traditional incident response assumed you could physically touch the affected machine — pull the hard drive, image it, run forensics. In the cloud:

- Instances can be terminated and their storage wiped in seconds
- Evidence is spread across CloudTrail, VPC Flow Logs, GuardDuty, application logs — in different services
- Actions happen via API calls, not keyboard input
- The "attacker" may be an IAM key operating autonomously long after the initial breach
- Blast radius can span multiple accounts and regions within minutes

Cloud IR requires different procedures, different tools, and speed above all else.

---

## The IR Lifecycle in AWS

```
Detection → Analysis → Containment → Eradication → Recovery → Lessons Learned
```

### Detection
- GuardDuty finding fires
- Security Hub alert
- CloudTrail anomaly (unusual API calls, new IAM user created)
- Application alert (spike in errors, auth failures)
- External notification (AWS Abuse report, bug bounty)

### Analysis
- Understand the scope: what was accessed, what was changed, what was exfiltrated?
- Who is the identity involved?
- What resources are affected?
- Is the attack ongoing?

### Containment
- Stop the bleeding — revoke credentials, isolate instances, block IPs
- Preserve evidence — snapshot EBS volumes, export logs before they expire

### Eradication
- Remove the attacker's access and any persistence they established
- Patch the vulnerability that allowed entry

### Recovery
- Restore from known-good snapshots or redeploy from IaC
- Monitor closely post-recovery for re-entry attempts

### Lessons Learned
- Root cause analysis
- Update runbooks, detection rules, and preventive controls

---

## Scenario 1 — Compromised IAM Access Key

This is the most common cloud incident — a key leaked in a GitHub repo, a `.env` file exposed, or malware exfiltrating credentials from an EC2 instance.

### Step 1 — Detect

GuardDuty finding: `UnauthorizedAccess:IAMUser/MaliciousIPCaller` or `CredentialAccess:IAMUser/AnomalousBehavior`

Or you notice it manually — someone emails you a "your AWS key is in this public repo" notification.

### Step 2 — Identify the Key

```bash
# Find the access key ID mentioned in the alert or repo
# Format: AKIA... (long-term) or ASIA... (temporary STS)

# Find which IAM user/role owns it
aws iam get-access-key-last-used --access-key-id AKIAIOSFODNN7EXAMPLE
```

> `[SCREENSHOT]` — *Terminal showing aws iam get-access-key-last-used output returning the UserName, last used date, service, and region — showing recent suspicious activity*

### Step 3 — Preserve Evidence First

Before touching anything, export relevant CloudTrail events:

```bash
# Export all API calls made with this key in the last 24 hours
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAIOSFODNN7EXAMPLE \
  --start-time $(date -d '24 hours ago' --iso-8601=seconds) \
  --output json > compromised-key-events.json
```

> `[SCREENSHOT]` — *Terminal showing cloudtrail lookup-events output listing API calls made with the compromised key — showing eventNames like CreateUser, AttachRolePolicy, RunInstances made from unusual IPs*

### Step 4 — Contain — Revoke the Key Immediately

```bash
# Deactivate the key (reversible)
aws iam update-access-key \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --status Inactive \
  --user-name affected-user

# Delete the key (irreversible — do this after you've saved evidence)
aws iam delete-access-key \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --user-name affected-user
```

> `[SCREENSHOT]` — *Terminal showing the aws iam update-access-key command completing with no error, confirming the key is deactivated*

### Step 5 — Check for Attacker Persistence

The attacker may have created resources to maintain access even after the key is revoked:

```bash
# Check for new IAM users created recently
aws iam list-users --query 'Users[?CreateDate>`2026-05-01`]'

# Check for new access keys on all users
aws iam list-users --query 'Users[*].UserName' --output text | \
  xargs -I{} aws iam list-access-keys --user-name {}

# Check for new roles with external trust
aws iam list-roles --query \
  'Roles[?AssumeRolePolicyDocument.Statement[?Principal.AWS!=`null`]]'

# Check for new Lambda functions (possible backdoor)
aws lambda list-functions --query 'Functions[?LastModified>`2026-05-01`]'

# Check for EC2 instances launched recently
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[?LaunchTime>`2026-05-01`]'
```

> `[SCREENSHOT]` — *Terminal showing aws iam list-users query returning a new user "backup-admin" created at the time of the incident — confirming the attacker created a backdoor account*

### Step 6 — Revoke All Active Sessions

Even after deleting the key, existing STS sessions derived from it may still be valid (up to their expiry):

```bash
# Attach a policy to the user that explicitly denies everything
aws iam put-user-policy \
  --user-name affected-user \
  --policy-name DenyAll \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Deny","Action":"*","Resource":"*"}]}'
```

For roles — update the trust policy to remove external trust or add a condition that denies all.

---

## Scenario 2 — Compromised EC2 Instance

GuardDuty finding: `Backdoor:EC2/C&CActivity.B` or `CryptoCurrency:EC2/BitcoinTool.B` — an instance is communicating with known bad IPs or running a miner.

### Step 1 — Preserve Evidence

Before isolating the instance, capture its current state:

```bash
INSTANCE_ID="i-0abc123def456"

# Create a snapshot of each attached EBS volume
aws ec2 describe-volumes \
  --filters "Name=attachment.instance-id,Values=$INSTANCE_ID" \
  --query 'Volumes[*].VolumeId' \
  --output text | \
  xargs -I{} aws ec2 create-snapshot \
    --volume-id {} \
    --description "IR snapshot $INSTANCE_ID $(date +%Y%m%d)"
```

> `[SCREENSHOT]` — *Terminal showing the aws ec2 create-snapshot commands completing, returning snapshot IDs for each volume attached to the compromised instance*

### Step 2 — Isolate the Instance (Quarantine Security Group)

Create a quarantine security group that blocks all traffic:

```bash
# Create quarantine SG — no inbound or outbound rules
QUARANTINE_SG=$(aws ec2 create-security-group \
  --group-name "quarantine-ir-$(date +%Y%m%d)" \
  --description "Incident Response Quarantine - blocks all traffic" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

# Replace ALL security groups on the instance with the quarantine SG
aws ec2 modify-instance-attribute \
  --instance-id $INSTANCE_ID \
  --groups $QUARANTINE_SG
```

> `[SCREENSHOT]` — *Terminal showing the quarantine security group being created and applied to the instance, followed by aws ec2 describe-instances confirming the instance now only has the quarantine SG attached*

The instance is now isolated — no inbound, no outbound. It's running but can't communicate.

### Step 3 — Revoke the Instance's IAM Role Credentials

The IAM role on the instance may have been used maliciously. Revoke active sessions:

```bash
# Find the instance profile / role
ROLE_NAME=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].IamInstanceProfile.Arn' \
  --output text | cut -d'/' -f2)

# Revoke all active sessions for this role
aws iam put-role-policy \
  --role-name $ROLE_NAME \
  --policy-name DenyAll-IR \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Deny","Action":"*","Resource":"*","Condition":{"DateLessThan":{"aws:TokenIssueTime":"2026-06-11T12:00:00Z"}}}]}'
```

This denies any API call from sessions issued before the specified time — killing all existing sessions derived from the role without affecting new legitimate ones.

### Step 4 — Forensic Analysis

With the instance isolated, connect for analysis via SSM (if SSM agent is running):

```bash
aws ssm start-session --target $INSTANCE_ID
```

Or mount the EBS snapshot to a forensic EC2 instance:

```bash
# Launch a forensic instance in an isolated VPC
# Attach the snapshot as a volume
aws ec2 attach-volume \
  --volume-id vol-snapshot-id \
  --instance-id forensic-instance-id \
  --device /dev/sdf
```

> `[SCREENSHOT]` — *SSM Session Manager terminal connected to the quarantined instance showing investigation commands — checking running processes, cron jobs, bash history, and network connections*

**What to look for:**
```bash
# Suspicious processes
ps auxf | grep -v "^\[" | sort -k3 -rn | head -20

# Cron persistence
crontab -l
cat /etc/cron* 2>/dev/null

# Recently modified files
find / -newer /tmp -type f 2>/dev/null | grep -v proc | head -50

# Bash history
cat ~/.bash_history
cat /home/*/.bash_history

# Network connections at time of incident (from VPC flow logs)
# Query from Athena or CloudWatch Logs
```

---

## CloudTrail Forensic Queries

During any incident, Athena is your best tool for querying CloudTrail at scale:

```sql
-- What did the compromised identity do?
SELECT eventtime, eventname, sourceipaddress, requestparameters, errorcode
FROM cloudtrail_logs
WHERE useridentity.accesskeyid = 'AKIAIOSFODNN7EXAMPLE'
  AND eventtime > '2026-06-10'
ORDER BY eventtime;

-- Were any IAM changes made?
SELECT eventtime, eventname, requestparameters, useridentity.arn
FROM cloudtrail_logs
WHERE eventsource = 'iam.amazonaws.com'
  AND eventtime > '2026-06-10'
  AND errorcode IS NULL
ORDER BY eventtime;

-- Any new resources created (possible persistence)?
SELECT eventtime, eventname, awsregion, requestparameters
FROM cloudtrail_logs
WHERE eventname IN ('CreateUser','CreateRole','RunInstances','CreateFunction20150331','CreateBucket')
  AND eventtime > '2026-06-10'
ORDER BY eventtime;

-- Console logins from unusual IPs
SELECT eventtime, useridentity.username, sourceipaddress, additionaleventdata
FROM cloudtrail_logs
WHERE eventname = 'ConsoleLogin'
  AND eventtime > '2026-06-01'
  AND sourceipaddress NOT LIKE '10.%'
ORDER BY eventtime DESC;
```

> `[SCREENSHOT]` — *Athena query editor showing one of the IR queries running with results below — listing the attacker's API calls with timestamps, event names, and source IPs*

---

## IR Runbook Template

Document your IR procedures as runbooks — executable checklists that any team member can follow during an incident.

```markdown
## Runbook: Compromised IAM Access Key

**Severity:** P1 — drop everything
**Detection source:** GuardDuty / CloudTrail alert / external report

### Immediate Actions (first 15 minutes)
- [ ] Identify the key ID from the alert
- [ ] Run: `aws iam get-access-key-last-used --access-key-id <KEY_ID>`
- [ ] Export CloudTrail events for the key (last 24h)
- [ ] Deactivate the key: `aws iam update-access-key --status Inactive`
- [ ] Notify security team and management

### Investigation (first hour)
- [ ] Review all API calls made with the key
- [ ] Check for new IAM users, roles, access keys created
- [ ] Check for new EC2, Lambda, RDS instances launched
- [ ] Check for data exfiltration (S3 GetObject on sensitive buckets)
- [ ] Identify how the key was exposed (git, logs, env var)

### Containment
- [ ] Delete the key (after evidence preserved)
- [ ] Add DenyAll inline policy to revoke active sessions
- [ ] Delete any attacker-created IAM users/roles/keys
- [ ] Terminate any attacker-launched resources

### Recovery
- [ ] Issue new credentials to the legitimate user
- [ ] Rotate any secrets that may have been accessed
- [ ] Verify system integrity of any instances the key touched

### Post-Incident
- [ ] Root cause written up
- [ ] Detection gap identified and new GuardDuty/Config rule added
- [ ] Runbook updated with lessons learned
```

> `[SCREENSHOT]` — *The runbook opened in a text editor or wiki, showing the checklist format with checkboxes — a clean, executable document*

---

## Key Takeaways

- Speed is everything — the longer a compromised key is active, the more damage is done
- Preserve evidence before containment — snapshots and CloudTrail exports first
- Quarantine security group is the cleanest EC2 isolation technique — fast and reversible
- The `DenyAll` inline policy with a `DateLessThan` condition revokes existing STS sessions without breaking new legitimate access
- Always check for attacker persistence — new IAM users, Lambda backdoors, cron jobs
- Athena + CloudTrail is your forensic platform — practice the queries before you need them
- Runbooks must be written before an incident — not during one

---

## References

<div class="references">
<ul>
  <li><a href="https://docs.aws.amazon.com/whitepapers/latest/aws-security-incident-response-guide/welcome.html" target="_blank">AWS Security Incident Response Guide</a></li>
  <li><a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html" target="_blank">Managing IAM Access Keys</a></li>
  <li><a href="https://github.com/aws-samples/aws-incident-response-playbooks" target="_blank">AWS IR Playbooks (GitHub)</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
