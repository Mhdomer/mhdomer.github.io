---
layout: post
title: AWS Systems Manager and Network Security Controls
date: 2026-06-25T10:00:00
categories:
  - SCS-C03 Cloud Security Speciality
  - Infrastructure Security
tags:
  - aws
  - systems-manager
  - session-manager
  - patch-manager
  - network-firewall
  - vpc-endpoints
  - cloud-security
  - scs-c03
author: muhammed
description: SCS-C03 Domain 3 — SSM Session Manager, Patch Manager, Network Firewall, VPC endpoints, Verified Access, security group and NACL controls, and Network Access Analyzer
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## AWS Systems Manager — Security Overview

**AWS Systems Manager** is the operations platform for managing EC2 instances and on-premises servers.
For the SCS-C03 exam, the most important SSM capabilities are:
- **Session Manager** — secure remote access without SSH keys or bastion hosts
- **Patch Manager** — automated OS patching with compliance reporting
- **Run Command** — execute scripts across fleets without SSH
- **OpsCenter** — centralized operational issues tracking (used in IR)

All SSM capabilities require the **SSM Agent** running on the instance and an **IAM instance profile** with SSM permissions.

---

## SSM Session Manager

**Session Manager** provides browser-based and CLI shell access to EC2 instances.
No inbound ports (22/SSH, 3389/RDP) need to be open — all traffic flows through the SSM service.
All session activity is logged to S3 and/or CloudWatch Logs.

### Why Session Manager Over Bastion Hosts

| | Bastion Host | Session Manager |
|---|---|---|
| **Inbound ports** | SSH 22 open | No inbound ports |
| **Key management** | SSH key rotation required | No SSH keys |
| **Audit** | Manual logging | CloudTrail + CW Logs + S3 |
| **Access control** | OS-level users | IAM policies |
| **Cost** | Bastion EC2 + EIP | Included in SSM |

```bash
# Start a session (requires AWS CLI + Session Manager plugin)
aws ssm start-session --target i-0abc123def456

# Start a session as a specific OS user
aws ssm start-session \
  --target i-0abc123def456 \
  --document-name AWS-StartInteractiveCommand \
  --parameters command="sudo -i"

# List active sessions
aws ssm describe-sessions --state Active

# Terminate a session
aws ssm terminate-session --session-id session-abc123
```

### Logging Session Activity

```bash
# Enable session logging in SSM preferences
aws ssm update-document \
  --name "SSM-SessionManagerRunShell" \
  --content '{
    "schemaVersion": "1.0",
    "description": "Session Manager preferences",
    "sessionType": "Standard_Stream",
    "inputs": {
      "s3BucketName": "my-ssm-session-logs",
      "s3EncryptionEnabled": true,
      "cloudWatchLogGroupName": "/aws/ssm/sessions",
      "cloudWatchEncryptionEnabled": true,
      "runAsEnabled": true,
      "runAsDefaultUser": "ssm-user"
    }
  }' \
  --document-version "\$LATEST"
```

**Exam tip:** A question asking how to eliminate SSH access to EC2 → the answer is Session Manager.
Combine with a security group that has no inbound rules to enforce this.

---

## IAM Policy for Session Manager

Control who can start sessions using IAM.
Use conditions to restrict sessions to specific instances or tags.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ssm:StartSession",
      "Resource": "arn:aws:ec2:eu-west-1:123456789012:instance/*",
      "Condition": {
        "StringEquals": {
          "ssm:resourceTag/Environment": "prod",
          "ssm:resourceTag/SessionManagerAccess": "allowed"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:DescribeSessions",
        "ssm:GetConnectionStatus",
        "ssm:DescribeInstanceProperties",
        "ec2:DescribeInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## SSM Patch Manager

**Patch Manager** automates OS patching for EC2 and on-premises servers.
It uses **patch baselines** to define which patches are approved, and **maintenance windows** to schedule when patches are applied.

### Patch Baseline

A patch baseline defines the rules for auto-approving patches — by severity, classification, and days-after-release.

```bash
# Create a custom patch baseline for Amazon Linux 2023
aws ssm create-patch-baseline \
  --name "AL2023-Security-7day" \
  --operating-system AMAZON_LINUX_2023 \
  --approval-rules PatchRules='[{
    "PatchFilterGroup": {
      "PatchFilters": [
        {"Key": "CLASSIFICATION", "Values": ["Security", "Bugfix"]},
        {"Key": "SEVERITY", "Values": ["Critical", "Important"]}
      ]
    },
    "ApproveAfterDays": 7
  }]' \
  --rejected-patches "kernel*" \
  --rejected-patches-action BLOCK

# Register the baseline as the default for the OS
aws ssm register-default-patch-baseline \
  --baseline-id pb-abc123
```

### Patch Compliance Reporting

```bash
# Run patching now (or schedule via maintenance window)
aws ssm send-command \
  --document-name AWS-RunPatchBaseline \
  --targets Key=tag:PatchGroup,Values=prod-linux \
  --parameters Operation=Install,RebootOption=RebootIfNeeded

# Check patch compliance for an instance
aws ssm describe-instance-patch-states \
  --instance-ids i-0abc123def456

# List non-compliant instances
aws ssm describe-instance-patch-states-for-patch-group \
  --patch-group prod-linux \
  --filters Key=State,Values=Failed,Missing
```

Amazon Inspector integrates with Patch Manager — Inspector findings for OS vulnerabilities can trigger patch remediation via SSM.

---

## SSM Run Command

**Run Command** executes scripts or documents on one or more instances without SSH.
Used for IR: isolate a compromised instance by running a firewall rule, collect memory, revoke credentials.

```bash
# Run a shell command on tagged instances
aws ssm send-command \
  --document-name AWS-RunShellScript \
  --targets Key=tag:Role,Values=web \
  --parameters commands=["df -h", "free -m", "ps aux"]

# Collect instance metadata for forensics
aws ssm send-command \
  --document-name AWS-RunShellScript \
  --instance-ids i-0abc123 \
  --parameters commands=[
    "netstat -an > /tmp/netstat.txt",
    "ps aux > /tmp/processes.txt",
    "aws s3 cp /tmp/ s3://forensics-bucket/$(hostname)/ --recursive"
  ]
```

---

## Network Security Controls

### Security Groups vs Network ACLs

| | Security Groups | Network ACLs |
|---|---|---|
| **Level** | Instance level | Subnet level |
| **State** | Stateful — return traffic allowed automatically | Stateless — inbound + outbound rules both required |
| **Rules** | Allow only (no explicit deny) | Allow AND deny |
| **Evaluation** | All rules evaluated | Rules evaluated in order (lowest number wins) |
| **Applies to** | ENI (elastic network interface) | All traffic entering/leaving subnet |

```bash
# Create a restrictive security group (web tier — inbound HTTPS only)
aws ec2 create-security-group \
  --group-name prod-web-sg \
  --description "Web tier — HTTPS only" \
  --vpc-id vpc-abc123

aws ec2 authorize-security-group-ingress \
  --group-id sg-web123 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Create a NACL rule to block a specific IP (not possible with SG)
aws ec2 create-network-acl-entry \
  --network-acl-id acl-abc123 \
  --rule-number 50 \
  --protocol -1 \
  --rule-action deny \
  --ingress \
  --cidr-block 1.2.3.4/32
```

---

## AWS Network Firewall

**AWS Network Firewall** is a managed stateful firewall for your VPC.
Unlike security groups and NACLs, Network Firewall understands Layer 7 traffic — it can inspect HTTP headers, TLS SNI, DNS queries, and apply Suricata-compatible rules.

Deploy Network Firewall in a dedicated **firewall subnet** in each AZ.
Route traffic through the firewall before it reaches your application subnets.

```
Internet Gateway
      │
 Firewall Subnet (Network Firewall endpoint)
      │
 Application Subnet (EC2, ECS, etc.)
```

```bash
# Create a Network Firewall
aws network-firewall create-firewall \
  --firewall-name prod-vpc-firewall \
  --firewall-policy-arn arn:aws:network-firewall:eu-west-1:123456789012:firewall-policy/prod-policy \
  --vpc-id vpc-abc123 \
  --subnet-mappings SubnetId=subnet-fw1 SubnetId=subnet-fw2

# Create a stateless rule group (Layer 4 — IP/port)
aws network-firewall create-rule-group \
  --rule-group-name block-known-bad-ips \
  --type STATELESS \
  --capacity 100 \
  --rule-group '{
    "RulesSource": {
      "StatelessRulesAndCustomActions": {
        "StatelessRules": [{
          "RuleDefinition": {
            "MatchAttributes": {
              "Sources": [{"AddressDefinition": "1.2.3.0/24"}]
            },
            "Actions": ["aws:drop"]
          },
          "Priority": 1
        }]
      }
    }
  }'
```

### Suricata-Compatible Rules (Stateful)

Network Firewall supports Suricata IDS/IPS rule syntax for stateful inspection:

```
# Block outbound connections to known C2 domains
alert dns $HOME_NET any -> any 53 (msg:"C2 domain"; dns.query; content:"malware.evil.com"; sid:1001; rev:1;)

# Block non-HTTPS traffic to the internet from app subnet
drop tcp $HOME_NET any -> $EXTERNAL_NET !443 (msg:"Block non-HTTPS"; sid:1002; rev:1;)
```

---

## VPC Endpoints — Private Access to AWS Services

**VPC endpoints** allow your private subnets to access AWS services without traversing the internet.

| Type | Services | Access |
|---|---|---|
| **Gateway endpoint** | S3, DynamoDB | Free — route table entry |
| **Interface endpoint** | Most AWS services (SSM, KMS, Secrets Manager, etc.) | $0.01/hr + data — ENI in subnet |

```bash
# Create gateway endpoint for S3
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.eu-west-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-private1 rtb-private2

# Create interface endpoint for SSM (required if instances have no internet access)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.eu-west-1.ssm \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-private1 subnet-private2 \
  --security-group-ids sg-endpoint \
  --private-dns-enabled

# SSM requires THREE interface endpoints for Session Manager:
# com.amazonaws.REGION.ssm
# com.amazonaws.REGION.ssmmessages
# com.amazonaws.REGION.ec2messages
```

**Exam tip:** If instances are in private subnets with no NAT Gateway, they need interface VPC endpoints for SSM, EC2 Messages, and SSM Messages to use Session Manager.

---

## AWS Verified Access

**Verified Access** provides zero-trust network access to internal applications without a VPN.
It integrates with IAM Identity Center, Okta, or other identity providers.
Access decisions are based on identity, device posture, and trust policies — evaluated on every request.

```bash
# Create a Verified Access instance
aws ec2 create-verified-access-instance \
  --description "Zero-trust access for internal apps"

# Create a trust provider (IAM Identity Center)
aws ec2 create-verified-access-trust-provider \
  --trust-provider-type user \
  --user-trust-provider-type iam-identity-center \
  --policy-reference-name idc

# Create an endpoint for an internal application
aws ec2 create-verified-access-endpoint \
  --verified-access-group-id vagr-abc123 \
  --endpoint-type load-balancer \
  --load-balancer-options LoadBalancerArn=arn:...,Port=443,Protocol=https,SubnetIds=subnet-abc \
  --domain-certificate-arn arn:aws:acm:...:certificate/abc
```

---

## Network Access Analyzer

**Network Access Analyzer** identifies unintended network access paths to your resources.
It finds paths where EC2 instances, RDS databases, or load balancers are reachable from the internet or across accounts when they should not be.
It is different from Inspector network reachability — Network Access Analyzer models the entire network topology.

```bash
# Create a network access scope (find internet-accessible resources)
aws ec2 create-network-insights-access-scope \
  --match-paths '[{
    "Source": {"ResourceStatement": {"ResourceTypes": ["AWS::EC2::InternetGateway"]}},
    "Destination": {"ResourceStatement": {"ResourceTypes": ["AWS::EC2::Instance"]}},
    "ThroughResources": [{"ResourceStatement": {"ResourceTypes": ["AWS::EC2::SecurityGroup"]}}]
  }]'

# Analyze the scope
aws ec2 start-network-insights-access-scope-analysis \
  --network-insights-access-scope-id nis-abc123
```

---

## Exam Key Points

- **Session Manager**: no SSH, no bastion, no open ports — all access via IAM + SSM Agent
- **Patch Manager**: patch baselines define approval rules; Inspector findings can trigger patching
- **Security Groups vs NACLs**: SGs are stateful (allow only), NACLs are stateless (allow + deny, numbered rules)
- **Network Firewall**: Layer 7 stateful inspection, Suricata rules, inspect HTTP/TLS/DNS — not possible with SGs or NACLs
- **Interface endpoints**: required for private subnets to reach SSM, KMS, Secrets Manager, ECR without NAT
- **Gateway endpoints**: free for S3 and DynamoDB — always prefer over NAT for those services
- **Verified Access**: zero-trust access to internal apps via identity provider — no VPN needed

---

## Quick Reference

```bash
# SSM Fleet
aws ssm describe-instance-information --output table
aws ssm start-session --target i-0abc123
aws ssm send-command --document-name AWS-RunPatchBaseline --targets Key=tag:Env,Values=prod --parameters Operation=Scan

# Security Groups
aws ec2 describe-security-groups --filters Name=vpc-id,Values=vpc-abc123
aws ec2 describe-network-acls --filters Name=vpc-id,Values=vpc-abc123

# VPC Endpoints
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=vpc-abc123

# Network Firewall
aws network-firewall list-firewalls
aws network-firewall describe-firewall --firewall-name prod-vpc-firewall
```
