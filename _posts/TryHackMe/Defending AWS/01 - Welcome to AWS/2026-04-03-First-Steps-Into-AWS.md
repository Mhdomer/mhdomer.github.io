---
layout: post
title: "Defending AWS: First Steps Into AWS - Console Navigation, CloudShell, and CLI Fundamentals"
date: 2026-04-03T10:00:00+03:00
categories:
  - TryHackMe
  - Defending AWS
tags:
  - tryhackme
  - cloud-security
  - aws
  - defending-aws
  - aws-cli
  - cloudshell
  - iam
  - ec2
  - s3
author: muhammed
description: Technical walkthrough of TryHackMe Defending AWS First Steps Into AWS. Explores AWS Management Console architecture, AWS CloudShell, STS identity verification with get-caller-identity, AWS CLI configuration, command syntax, and S3 resource enumeration.
toc: true
pin: false
math: false
mermaid: true
image: https://cdn-images.tryhackme.com/room-icons/68baea2454c82afe90fd7020-1781258694750
---

## Overview

[First Steps Into AWS](https://tryhackme.com/room/firststepsintoaws) is the foundational room of Module 1 (**Welcome to AWS**) in TryHackMe's **Defending AWS** learning path. The path consists of 5 modules and 23 hands-on labs designed to teach cloud defenders how to identify, triage, and mitigate real-world misconfigurations across Amazon Web Services.

Before analyzing complex identity boundary escalations, wide-open security groups, or unencrypted data buckets, security engineers must understand how administrative requests interact with the AWS control plane. This lab establishes operational proficiency across the three primary interaction methods:
1. **The AWS Management Console**: The graphical interface for configuration inspection and administrative navigation.
2. **AWS CloudShell**: A pre-authenticated, browser-based Linux shell environment for cloud automation without client-side credential storage.
3. **The AWS Command Line Interface (CLI)**: The programmatic interface used to query resources, parse JSON responses, and automate defensive auditing.

---

## 1. The AWS Management Plane Architecture

Every interaction with AWS (whether initiated through the web console, a command-line script, or Terraform) is an authenticated, encrypted HTTPS request against the AWS REST API:

```mermaid
flowchart TD
    subgraph Clients["Operator Interfaces"]
        Console["AWS Management Console<br/>(Web Browser via Session Cookie)"]
        CloudShell["AWS CloudShell<br/>(Ephemeral Container + Role Credentials)"]
        CLI["AWS CLI / SDKs<br/>(Access Keys in ~/.aws/credentials)"]
    end

    subgraph AuthPlane["Authentication & Control Plane"]
        IAM["AWS IAM / STS<br/>• Identity Verification<br/>• SigV4 Signature Validation<br/>• Policy Evaluation Logic"]
    end

    subgraph Services["Target AWS Services (us-east-1)"]
        S3["Amazon S3<br/>Object Storage & Buckets"]
        EC2["Amazon EC2<br/>Compute Instances & VPC"]
        Logs["CloudTrail / CloudWatch<br/>Control Plane Telemetry"]
    end

    Console --> AuthPlane
    CloudShell --> AuthPlane
    CLI --> AuthPlane
    AuthPlane --> S3
    AuthPlane --> EC2
    AuthPlane --> Logs
```

### Key Architectural Concepts

1. **Global Infrastructure (Regions vs Availability Zones):**
   - An **AWS Region** is a distinct geographic area (such as `us-east-1` in N. Virginia). Each region operates independently to maintain fault tolerance.
   - An **Availability Zone (AZ)** consists of one or more discrete data centers with redundant power, networking, and connectivity within a Region (such as `us-east-1a`, `us-east-1b`).
   - Security controls and resource listings are inherently region-scoped for regional services (EC2, VPC, Security Groups), while identity services (IAM) are global.

2. **The Signature Version 4 (SigV4) Signing Protocol:**
   - Every CLI request calculates a cryptographic HMAC-SHA256 signature using the user's secret access key, timestamp, and request body. The secret key is never sent across the wire.

3. **Caller Identity Verification:**
   - The Security Token Service (STS) endpoint provides instant validation of the active caller identity via `aws sts get-caller-identity`, confirming the Account ID, User ARN, and Unique User ID.

---

## 2. Navigating the AWS Web Interface

The AWS Management Console acts as the centralized graphical interface for managing cloud infrastructure.

```mermaid
flowchart LR
    subgraph ConsoleBar["AWS Console Top Navigation Bar"]
        Search["Service Search Bar<br/>(Alt + S)"]
        CS_Icon["CloudShell Icon<br/>(Terminal Access)"]
        Bell["Notifications<br/>& AWS Health"]
        RegionSelect["Region Selector<br/>(Default: us-east-1)"]
        UserMenu["Account ID / Role Menu<br/>(User Identity)"]
    end
```

### Essential Navigation Elements for Defenders

- **Unified Search Bar (`Alt + S`):** Quick-launch interface to locate services (IAM, GuardDuty, CloudTrail), documentation, and specific resource IDs directly.
- **Region Selector:** Displays the current operating region. In defensive operations, failing to verify the active region is a common source of enumeration misses (e.g., checking for unattached EBS volumes in `us-east-1` while an attacker launched crypto-miners in `ap-southeast-1`).
- **Account Identification Menu:** Displays the 12-digit AWS Account ID and the active IAM user or assumed role session name.

---

## 3. AWS CloudShell vs Local CLI

AWS CloudShell provides a browser-based, pre-authenticated compute environment built on Amazon Linux 2023.

### CloudShell Operational Characteristics

| Feature | AWS CloudShell | Local AWS CLI (`aws-cli`) |
|---|---|---|
| **Credential Storage** | Automatic (Temporary STS session credentials injected into environment) | Static or cached tokens stored in `~/.aws/credentials` |
| **Local Disk Space** | 1 GB persistent home directory (`/home/cloudshell-user`) | Local host disk space |
| **Pre-Installed Tools** | AWS CLI v2, Python 3, Node.js, Git, Docker, jq | Requires manual package installation |
| **Network Egress** | Direct AWS backbone connectivity | Subject to local network firewalls and proxies |
| **Credential Leak Risk** | Very low (No permanent access keys on physical disk) | Moderate to high if workstation is compromised |

### Verifying Identity via CloudShell

Upon launching CloudShell, running `aws sts get-caller-identity` verifies the current context:

```bash
[cloudshell-user@ip-10-0-1-5 ~]$ aws sts get-caller-identity
{
    "UserId": "AIDAKNZS5EXAMPLEKEY",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/thm-student"
}
```

The output yields three critical data points:
- `UserId`: Unique alphanumeric identifier assigned to the IAM entity.
- `Account`: The 12-digit AWS Account ID.
- `Arn`: The Amazon Resource Name uniquely identifying the security principal.

---

## 4. AWS CLI Setup and Command Architecture

The AWS Command Line Interface allows security engineers to query, script, and automate administrative and auditing tasks.

### CLI Configuration Files

When configuring the AWS CLI manually using `aws configure`, two configuration files are generated under the user's home directory:

1. **`~/.aws/credentials`:** Contains authentication secrets:
   ```ini
   [default]
   aws_access_key_id = AKIAIOSFODNN7EXAMPLE
   aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
   ```
2. **`~/.aws/config`:** Contains operational settings:
   ```ini
   [default]
   region = us-east-1
   output = json
   ```

### Anatomy of an AWS CLI Command

All AWS CLI commands adhere to a standardized hierarchical syntax:

```text
aws  <service>  <subcommand>  [--parameters]  [--options]
 |       |           |               |              |
root  target     API action      arguments      output / profile
```

Example command breakdown:
```bash
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" --output json
```
- `aws`: Root binary.
- `ec2`: Target AWS service.
- `describe-instances`: The API operation mapped to `ec2:DescribeInstances`.
- `--filters`: Parameter filtering the query results on the server side.
- `--output json`: Global option specifying the serialization format (json, text, table, yaml).

### Exploring Built-in Documentation

Every service module includes built-in interactive manual pages accessible through the `help` flag:

```bash
aws ec2 help
```

In the EC2 manual, the operations list is sorted alphabetically. The very first available subcommand in the list is:

```text
accept-reserved-instances-exchange-quote
```

### Enumerating Storage Resources via CLI

Amazon S3 commands exist at two abstraction layers:
1. **High-Level S3 Commands (`aws s3`):** Simplified file management commands (`ls`, `cp`, `sync`, `rm`, `mb`, `rb`).
2. **Low-Level S3 API Commands (`aws s3api`):** Direct mappings to REST endpoints (`list-buckets`, `get-bucket-policy`, `put-bucket-acl`).

Listing buckets in the lab environment:

```bash
[cloudshell-user@ip-10-0-1-5 ~]$ aws s3 ls
2026-04-03 09:15:22 tryhackme-bucket-123456789012
```

The bucket follows TryHackMe's lab naming convention with the prefix:

```text
tryhackme-bucket-
```

---

## 5. Defensive Engineering Best Practices

As security engineers managing AWS environments, adhere to the following baseline controls:

1. **Avoid Long-Lived Static Access Keys:**
   - Permanent IAM access keys stored in `~/.aws/credentials` represent one of the leading causes of cloud credential leaks on GitHub.
   - Enforce AWS IAM Identity Center (Single Sign-On) with short-lived session tokens (1 to 12 hours) or AWS CloudShell for administrative tasks.

2. **Lock Down Regions with Service Control Policies (SCPs):**
   - Prevent attackers from deploying shadow infrastructure in unmonitored regions by applying an SCP denying all actions outside designated business regions (e.g. `us-east-1` and `eu-west-1`):
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "DenyAllOutsideApprovedRegions",
         "Effect": "Deny",
         "NotAction": [
           "iam:*",
           "organizations:*",
           "route53:*",
           "cloudfront:*",
           "sts:*"
         ],
         "Resource": "*",
         "Condition": {
           "StringNotEquals": {
             "aws:RequestedRegion": [
               "us-east-1"
             ]
           }
         }
       }
     ]
   }
   ```

3. **Automate Output Parsing with JMESPath (`--query`):**
   - Filter responses directly on the client side using built-in JMESPath queries rather than relying on brittle shell pipelines:
   ```bash
   aws s3api list-buckets --query "Buckets[?starts_with(Name, 'tryhackme')].Name" --output text
   ```

---

## 6. Step-by-Step Task Walkthrough

### Task 1: Introduction

This task outlines the learning goals of the room, introducing the AWS Management Console, CloudShell, and the CLI.

- **Question:** I understand the goals of this room.
- **Answer:** `No answer needed`

---

### Task 2: TryHackMe AWS Environment Overview

Task 2 provides access credentials and explains how the sandboxed lab environment operates.
- Deploy the virtual lab.
- Log in to the AWS Management Console using the supplied temporary IAM credentials.

- **Question:** I have logged into the AWS console using the provided credentials.
- **Answer:** `No answer needed`

---

### Task 3: AWS Web Interface Overview

Task 3 guides users through navigating the web interface.
- Observe the top navigation bar.
- Locate the region selector in the top-right corner next to the account name.
- The default region assigned to the lab session is US East (N. Virginia), identified by its region code `us-east-1`.

#### Questions and Answers for Task 3

1. **Question:** What is the default AWS region configured for the lab session?
   - **Answer:** `us-east-1`

---

### Task 4: AWS CloudShell Overview

Task 4 introduces AWS CloudShell.
- Click the CloudShell icon in the top navigation bar or search for "CloudShell".
- Once the terminal session initializes, run the STS caller identity command to inspect the active credentials:
  ```bash
  aws sts get-caller-identity
  ```
- The output displays the current user's Account number, User ID, and full IAM ARN.

#### Questions and Answers for Task 4

1. **Question:** Run the command to check your current AWS identity in CloudShell. What is the command used?
   - **Answer:** `aws sts get-caller-identity`

---

### Task 5: AWS CLI Setup

Task 5 covers configuring the AWS CLI in environments where credentials are not pre-injected.
- Run `aws configure`.
- Input the Access Key ID, Secret Access Key, default region (`us-east-1`), and output format (`json`).
- Verify the generated files in `~/.aws/credentials` and `~/.aws/config`.

- **Question:** I have configured the AWS CLI with the provided credentials.
- **Answer:** `No answer needed`

---

### Task 6: AWS CLI Commands Structure

Task 6 covers command syntax and resource queries.
1. Run `aws ec2 help` to inspect available EC2 API operations.
   - Scroll to the "AVAILABLE COMMANDS" section.
   - The first command listed alphabetically is accept-c.
1. Run `aws s3 ls` to list existing storage buckets.
   - Examine the bucket matching the account ID.
   - The bucket prefix before the numeric account ID is `tryhackme-bucket-`.

#### Questions and Answers for Task 6

1. **Question:** In the list of commands for EC2 services, what is the first command that appears?
   - **Answer:** `address-transfer`

2. **Question:** Run `aws s3 ls`. What is the prefix of the S3 bucket that contains your account ID?
   - **Answer:** `tryhackme-bucket-`

---

### Task 7: Conclusion

Task 7 summarizes the foundational tools needed to begin the Defending AWS learning path.

- **Question:** I am ready to continue with the Defending AWS learning path!
- **Answer:** `No answer needed`

---

## Complete Question and Answer Summary

| Task            | Question / Prompt                                                                           | Answer                        |
| --------------- | ------------------------------------------------------------------------------------------- | ----------------------------- |
| **Task 1**      | I understand the goals of this room.                                                        | `No answer needed`            |
| **Task 2**      | I have logged into the AWS console using the provided credentials.                          | `No answer needed`            |
| **Task 3**      | What is the default AWS region configured for the lab session?                              | `us-east-1`                   |
| **Task 4**      | Run the command to check your current AWS identity in CloudShell. What is the command used? | `aws sts get-caller-identity` |
| **Task 5**      | I have configured the AWS CLI with the provided credentials.                                | `No answer needed`            |
| **Task 6 (Q1)** | In the list of commands for EC2 services, what is the first command that appears?           | `address-transfer`            |
| **Task 6 (Q2)** | Run `aws s3 ls`. What is the prefix of the S3 bucket that contains your account ID?         | `tryhackme-bucket-`           |
| **Task 7**      | I am ready to continue with the Defending AWS learning path!                                | `No answer needed`            |

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
