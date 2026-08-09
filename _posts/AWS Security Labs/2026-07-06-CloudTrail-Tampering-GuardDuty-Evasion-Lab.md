---
layout: post
title: "Lab CloudTrail Tampering and GuardDuty Evasion: How Attackers Go Dark"
date: 2026-07-06T10:00:00
categories:
  - AWS Security Labs
  - Detection lab
tags:
  - aws
  - cloudtrail
  - guardduty
  - detection-evasion
  - anti-forensics
  - logging
  - incident-response
  - cloud-attack
  - lab
author: muhammed
description: A hands-on lab demonstrating how an attacker with an over-privileged "on-call" role disables CloudTrail, suspends GuardDuty, and deletes CloudWatch log groups to operate undetected  and how organization trails, delegated GuardDuty administration, S3 Object Lock, and SCPs make logging survive even a fully compromised member account.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fpersol-serverworks.co.jp%2Fblog%2FCloudTrail.png&f=1&nofb=1&ipt=d5ad7480aa9c8223793334ec2a929c22e65d1f849ad968d87b7da1050662e20f
---

## Objective

Start with credentials for an over-privileged "SRE on-call" IAM role — the kind of standing access ops teams grant themselves "for incident response."
Use it to disable the exact controls that would otherwise catch everything else the attacker does: CloudTrail, GuardDuty, and CloudWatch log groups holding VPC Flow Logs.
Then show what stops this from working: organization trails, delegated GuardDuty administration, S3 Object Lock, and service control policies that survive compromise of the member account itself.

**Run this only in an AWS account you own.**

---

## Why This Is the Move That Matters Most

Every previous lab in this series ended with a line like "GuardDuty finding: `X`." That framing quietly assumes GuardDuty is still running, and CloudTrail is still recording, by the time the attacker does something worth detecting.

Real intrusions don't work that way. Once an attacker lands credentials with enough reach  often exactly the "just in case" permissions given to on-call and platform teams the first move isn't the noisy one. It's turning off the lights: stop the trail, suspend the detector, delete the flow logs. Only after that does the attacker create backdoor users, exfiltrate data, or pivot across accounts, because now none of it gets written down.

This lab is deliberately not about a flashy technical exploit. It's about a handful of unglamorous API calls  `StopLogging`, `DeleteTrail`, `UpdateDetector`, `DeleteLogGroup`  that are far more dangerous than most of the "attacks" in this series, because they don't just cause damage, they erase the ability to know damage happened at all.

---

## Attack Chain

```
Compromised IAM identity: sre-oncall-role (over-privileged "incident response" access)
    │
    │  Step 1: Enumerate active security controls
    ▼
CloudTrail trails + GuardDuty detectors + CloudWatch log groups discovered
    │
    │  Step 2: Stop / delete the CloudTrail trail
    │  Step 3: Suspend or delete the GuardDuty detector
    │  Step 4: Delete the CloudWatch log group holding VPC Flow Logs
    ▼
Defender blind spot — everything from this point on is unlogged
    │
    │  Step 5 (defender): out-of-band controls catch the tampering itself
    ▼
Detection despite the blinding attempt — but only if those controls existed beforehand
```

---

## Lab Architecture

```
AWS Account: security-lab (single account, to isolate the technique)

IAM Role: sre-oncall-role
    Trust: assumable by the on-call engineering group
    Policy (real-world "just in case" over-grant):
      - cloudtrail:StopLogging, cloudtrail:DeleteTrail
      - guardduty:UpdateDetector, guardduty:DeleteDetector, guardduty:ArchiveFindings
      - logs:DeleteLogGroup
      - ec2:DeleteFlowLogs

CloudTrail: management-trail
    Single-account trail, standard S3 bucket, no Object Lock (initial state)

GuardDuty: detector in this account only — not a delegated-admin member,
    no organization-wide protection (initial state)

CloudWatch Log Group: /aws/vpc/flow-logs
    Receiving VPC Flow Logs for the account's default VPC
```

---

## Phase 0 — Setup

### Step 0.1 — Create the CloudTrail Trail

```bash
aws s3 mb s3://security-lab-cloudtrail-logs

cat > trail-bucket-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::security-lab-cloudtrail-logs"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::security-lab-cloudtrail-logs/AWSLogs/*",
      "Condition": {"StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}}
    }
  ]
}
EOF

aws s3api put-bucket-policy --bucket security-lab-cloudtrail-logs --policy file://trail-bucket-policy.json

aws cloudtrail create-trail \
  --name management-trail \
  --s3-bucket-name security-lab-cloudtrail-logs \
  --is-multi-region-trail

aws cloudtrail start-logging --name management-trail
```

### Step 0.2 — Enable GuardDuty

```bash
aws guardduty create-detector --enable
DETECTOR_ID=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text)
```

### Step 0.3 — Enable VPC Flow Logs to CloudWatch

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)

aws logs create-log-group --log-group-name /aws/vpc/flow-logs

aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flow-logs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flow-logs-role
```

### Step 0.4 — Create the Over-Privileged "On-Call" Role

This is the real-world misconfiguration the lab is built around: an operational role given broad logging/security permissions so it can "respond to incidents," which doubles as the ability to destroy incident response evidence.

```bash
cat > sre-oncall-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:UpdateTrail",
        "guardduty:UpdateDetector",
        "guardduty:DeleteDetector",
        "guardduty:ArchiveFindings",
        "logs:DeleteLogGroup",
        "ec2:DeleteFlowLogs"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-role --role-name sre-oncall-role --assume-role-policy-document file://order-processor-trust.json
aws iam put-role-policy --role-name sre-oncall-role --policy-name sre-oncall-inline --policy-document file://sre-oncall-policy.json
```

IAM console showing `sre-oncall-role`'s inline policy with `cloudtrail:DeleteTrail`, `guardduty:DeleteDetector`, and `logs:DeleteLogGroup` all granted account-wide

Assume the role to simulate the attacker who has obtained these credentials  through phishing, a leaked access key, or any of the credential-theft paths covered in earlier labs in this series.

---

## Phase 1 — Enumerate Active Security Controls

Before touching anything, the attacker checks what's actually watching.

```bash
aws cloudtrail describe-trails
aws cloudtrail get-trail-status --name management-trail

aws guardduty list-detectors
aws guardduty get-detector --detector-id $DETECTOR_ID

aws logs describe-log-groups
aws ec2 describe-flow-logs
```

This reconnaissance is itself logged as `LookupEvents` / read-only management events  normal and easy to miss among the noise of legitimate `Describe*`/`Get*`/`List*` calls that happen constantly in any account. It's the destructive calls that follow that matter.

---

## Phase 2 — Stop and Delete CloudTrail

```bash
# Suspends logging but leaves the trail resource in place
aws cloudtrail stop-logging --name management-trail

# Confirm
aws cloudtrail get-trail-status --name management-trail
# "IsLogging": false
```

For a cleaner exit, delete the trail entirely:

```bash
aws cloudtrail delete-trail --name management-trail
```

> 📸 **SCREENSHOT:** `aws cloudtrail get-trail-status` showing `"IsLogging": false` immediately after `stop-logging`

Both `StopLogging` and `DeleteTrail` are themselves CloudTrail management events  but only up until the exact moment logging actually stops. A single-account trail can record its own shutdown call, and then nothing after. This is the core limitation of relying on one trail as your only source of truth: it is a security control the same identity that owns the account can always turn off.

---

## Phase 3 — Suspend or Delete GuardDuty

```bash
# Soft disable — keeps the detector but stops analysis
aws guardduty update-detector --detector-id $DETECTOR_ID --enable false

# Or remove it entirely, losing configuration and finding history
aws guardduty delete-detector --detector-id $DETECTOR_ID
```

If the attacker prefers to leave GuardDuty running but hide the evidence it's already collected:

```bash
aws guardduty list-findings --detector-id $DETECTOR_ID

aws guardduty archive-findings \
  --detector-id $DETECTOR_ID \
  --finding-ids finding-id-1 finding-id-2
```

 GuardDuty console showing the detector toggled to disabled, or a batch of findings moved to "Archived"

**GuardDuty finding (if it still catches this):**

```
Stealth:IAMUser/CloudTrailLoggingDisabled
```

This finding type exists specifically for this scenario  GuardDuty watching for its own logging source being turned off. The catch: it depends on the detector itself still being alive and able to process the `StopLogging` event before (or if) it's also disabled. In a single-account setup where one identity controls both CloudTrail and GuardDuty, there's a real race  whichever gets disabled first may prevent the other's tampering from ever being flagged.

---

## Phase 4 — Delete CloudWatch Log Groups and VPC Flow Logs

With the account-level trail and detector down, the attacker cleans up the remaining record of network activity.

```bash
aws logs delete-log-group --log-group-name /aws/vpc/flow-logs

FLOW_LOG_ID=$(aws ec2 describe-flow-logs --query 'FlowLogs[0].FlowLogId' --output text)
aws ec2 delete-flow-logs --flow-log-ids $FLOW_LOG_ID
```

From here, the attacker can create a backdoor IAM user with `AdministratorAccess`, pivot across the account, or exfiltrate whatever they want  and in this account, none of it has a CloudTrail record, a GuardDuty finding, or a flow log entry. This is the point of the lab: everything after Phase 4 is invisible to a defender relying only on this account's local controls.

---

## Phase 5 — Detection Despite the Evasion Attempt

This is where the lab flips from attacker to defender. None of Phases 2–4 should have been possible without tripping an alarm  if the account had been set up correctly beforehand.

### Fix 1 — Organization CloudTrail Trail

An organization trail is created once, from the AWS Organizations management account (or a delegated administrator), and applies to every member account. Member-account principals including admins  cannot stop, modify, or delete it.

```bash
# Run from the Organizations management account
aws cloudtrail create-trail \
  --name org-management-trail \
  --s3-bucket-name org-cloudtrail-logs-central \
  --is-organization-trail \
  --is-multi-region-trail

aws cloudtrail start-logging --name org-management-trail
```

`sre-oncall-role` in the member account has no API surface that can touch `org-management-trail` at all  it isn't visible via `describe-trails` run from inside the member account in the way that would let it be altered. This alone converts the attack in Phases 2–3 from "logging destroyed" to "local trail destroyed, org trail intact."

### Fix 2 — Delegated GuardDuty Administrator

```bash
# From the Organizations management account
aws guardduty enable-organization-admin-account --admin-account-id 999999999999

# From the delegated admin account (999999999999)
aws guardduty update-organization-configuration \
  --detector-id CENTRAL_DETECTOR_ID \
  --auto-enable-organization-members ALL
```

With a delegated administrator, member-account users can disable *their local view* of GuardDuty, but findings are still generated and retained centrally in the delegated admin account, which the compromised member account has no access to.

### Fix 3 — S3 Object Lock and Log File Validation on the Archive Bucket

Store the org trail's logs in a bucket in a separate log-archive account, with Object Lock in compliance mode — not even that account's root user can delete objects before the retention period expires.

```bash
aws s3api create-bucket --bucket org-cloudtrail-logs-central --object-lock-enabled-for-bucket

aws s3api put-object-lock-configuration \
  --bucket org-cloudtrail-logs-central \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {"DefaultRetention": {"Mode": "COMPLIANCE", "Days": 365}}
  }'

aws cloudtrail update-trail --name org-management-trail --enable-log-file-validation
```

Log file validation lets you cryptographically prove after the fact whether any delivered log file was altered or deleted  turning "did the attacker touch the logs" from a guess into a verifiable check.

### Fix 4 — Service Control Policy Denying Destructive Logging Actions

The single most important fix: an SCP applied at the OU level, so it binds every account in scope regardless of what IAM policies exist inside them  including accounts an attacker fully compromises.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLoggingTampering",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:UpdateTrail",
        "guardduty:DeleteDetector",
        "guardduty:UpdateDetector",
        "guardduty:DisassociateFromMasterAccount",
        "logs:DeleteLogGroup",
        "ec2:DeleteFlowLogs"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": "arn:aws:iam::123456789012:role/breakglass-logging-admin"
        }
      }
    }
  ]
}
```

```bash
aws organizations create-policy \
  --name deny-logging-tampering \
  --type SERVICE_CONTROL_POLICY \
  --content file://scp-deny-logging-tampering.json

aws organizations attach-policy \
  --policy-id p-EXAMPLE \
  --target-id ou-example-workloads
```

This is the control that actually matters here: an SCP is enforced at the Organizations level and evaluated before any IAM policy in the account. Even a fully compromised account admin or the root user  cannot perform a denied action unless they're the single named break-glass role, which should itself require a separate approval workflow to assume.

### Fix 5 — Real-Time Alerting on the Tampering Itself

```bash
cat > eventbridge-pattern.json << 'EOF'
{
  "source": ["aws.cloudtrail", "aws.guardduty", "aws.logs"],
  "detail": {
    "eventName": [
      "StopLogging", "DeleteTrail", "UpdateTrail",
      "DeleteDetector", "UpdateDetector",
      "DeleteLogGroup", "DeleteFlowLogs"
    ]
  }
}
EOF

aws events put-rule \
  --name detect-logging-tampering \
  --event-pattern file://eventbridge-pattern.json

aws events put-targets \
  --rule detect-logging-tampering \
  --targets "Id"="1","Arn"="arn:aws:sns:us-east-1:123456789012:security-alerts"
```

 EventBridge rule `detect-logging-tampering` firing and an SNS notification landing within seconds of the `StopLogging` call

Wiring this to page the security team directly  not just log a finding closes the race condition from Phase 3: even if GuardDuty itself gets disabled a moment later, the EventBridge rule has already fired from the initial CloudTrail management event, because EventBridge processes the org trail independent of any single account's detector state.

### Fix 6 — Continuous Compliance Checks

```bash
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "cloudtrail-enabled",
  "Source": {"Owner": "AWS", "SourceIdentifier": "CLOUD_TRAIL_ENABLED"}
}'

aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "guardduty-enabled-centralized",
  "Source": {"Owner": "AWS", "SourceIdentifier": "GUARDDUTY_ENABLED_CENTRALIZED"}
}'
```

AWS Config continuously re-checks these conditions independent of any single event stream, catching drift even if an EventBridge rule was somehow missed or misconfigured.

### Fix 7 — Stop Granting Standing "Just in Case" Logging Permissions

The root cause in this lab wasn't a technical exploit  it was `sre-oncall-role` having `cloudtrail:DeleteTrail` and `guardduty:DeleteDetector` as standing permissions for a role used daily. Move break-glass logging/security administration to IAM Identity Center permission sets with short session durations and a separate approval step, rather than baking it into an always-on operational role.

---

## Key Takeaways

- Disabling logging and detection is usually an attacker's *first* move after gaining sufficient access, not a late-stage cleanup step  alerting only on "suspicious activity" and not on security-control changes misses the highest-signal event in the whole intrusion
- A single-account CloudTrail trail can record its own shutdown call, but only up to that exact moment  after that, silence. Resilience requires a trail the compromised account cannot control at all
- GuardDuty's `Stealth:IAMUser/CloudTrailLoggingDisabled` finding exists for exactly this scenario, but it's only reliable when GuardDuty itself is organizationally delegated and not dependent on the same account being attacked
- SCPs are the one control that survives compromise of a member account's own admin or root user  IAM policies inside that account are not a security boundary against that account's own identities
- "Just in case" standing permissions for on-call/ops roles are themselves a liability; the ability to destroy evidence should require the same rigor as the incident it's meant to respond to

---

## Cleanup

```bash
aws cloudtrail delete-trail --name management-trail
aws s3 rb s3://security-lab-cloudtrail-logs --force

aws guardduty delete-detector --detector-id $DETECTOR_ID

aws logs delete-log-group --log-group-name /aws/vpc/flow-logs
aws ec2 delete-flow-logs --flow-log-ids $FLOW_LOG_ID

aws iam delete-role-policy --role-name sre-oncall-role --policy-name sre-oncall-inline
aws iam delete-role --role-name sre-oncall-role

# If you created the organizational fixtures for Phase 5:
aws organizations detach-policy --policy-id p-EXAMPLE --target-id ou-example-workloads
aws organizations delete-policy --policy-id p-EXAMPLE
```

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
