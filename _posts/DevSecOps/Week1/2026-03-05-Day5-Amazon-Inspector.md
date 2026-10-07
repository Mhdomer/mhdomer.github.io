---
layout: post
title: "Day 5: Amazon Inspector - Automated Cloud Vulnerability Scanning"
date: 2026-03-05 10:00:00 +0800
categories:
  - DevSecOps
  - Week1
tags:
  - AWS
  - Inspector
  - VulnerabilityManagement
  - CloudSecurity
author: muhammed
description: A practical walkthrough of Amazon Inspector v2 - automated vulnerability scanning for EC2 instances, ECR container images, Lambda functions, and smart prioritization.
toc: true
pin: false
math: false
mermaid: false
image: /assets/devsecops/inspector.png
---

## The Question: How Do You Know Your Workloads Are Vulnerable?

When you deploy infrastructure to the cloud, you are rarely writing 100% of the code or compiling the operating system yourself:

- You launch an Ubuntu or Amazon Linux EC2 instance running Nginx and OpenSSL.
- You build a Docker container based on `python:3.9` containing dozens of Debian packages.
- You deploy a serverless Lambda function importing libraries like `requests`, `urllib3`, or `boto3`.

Every week, security researchers discover new vulnerabilities (CVEs) in those exact libraries. How do you know when an EC2 instance in your production fleet or a container image in your registry has an unpatched remote code execution vulnerability?

Do you SSH into every machine manually and run package audits every morning?

That is the exact problem **Amazon Inspector v2** solves. It is AWS's fully managed, continuous vulnerability scanner that inspects your EC2 instances, ECR container registries, and Lambda functions automatically.

![Amazon Inspector dashboard summary overview](/assets/devsecops/week1/Pasted%20image%2020260523201014.png)

---

## What Amazon Inspector v2 Covers

Unlike legacy scanners that require scheduled maintenance windows or heavy third-party agents, Inspector v2 is continuous and near-real-time:

1. **Amazon EC2 Instances:** Scans installed operating system packages (via apt, yum, or dnf) and application runtime libraries without installing a dedicated security agent. It leverages the AWS Systems Manager (SSM) Agent already baked into standard cloud AMIs.
2. **Amazon ECR (Elastic Container Registry):** Scans container images both on-push and continuously as new CVEs are added to the national vulnerability database.
3. **AWS Lambda Functions:** Scans application dependencies packaged inside your function code (e.g. `package.json`, `requirements.txt`, Gemfiles).

All discovered vulnerabilities automatically feed directly into AWS Security Hub.

---

## How Inspector Works Under the Hood

### Agentless Scanning via SSM

For EC2 instances, Inspector relies on the **AWS Systems Manager (SSM) Agent**.

![Amazon Inspector coverage SSM agent status](/assets/devsecops/week1/Pasted%20image%2020260523202651.png)

If an instance is managed by SSM, Inspector collects the inventory of installed packages and evaluates them against vulnerability feeds. It also checks network reachability: looking at VPC route tables, internet gateways, and security groups to determine if the port associated with that vulnerable service is reachable from the open internet!

---

## Decoding the Inspector Score: Why Context Matters

One of the biggest headaches for junior security engineers is **alert fatigue**. You run a scanner, and it spits out 400 "Critical" and "High" CVEs. Where do you start?

A generic CVSS v3 score (e.g. 9.8 Critical) only measures the theoretical severity of a vulnerability in a lab setting. It does not know anything about your cloud architecture.

Amazon Inspector calculates a adjusted **Amazon Inspector Score** that combines three factors:

```
[ CVSS Base Score ] + [ Network Reachability ] + [ Exploit Availability ] 
                           = Amazon Inspector Adjusted Score
```

| Factor | What Inspector Evaluates |
| :--- | :--- |
| **CVSS Base Score** | The severity of the software flaw itself |
| **Network Reachability** | Is the affected port actually open to `0.0.0.0/0` via security groups and internet gateways, or is the server isolated in a private subnet? |
| **Exploit Availability** | Is there a weaponized public exploit available in the wild (e.g. on Metasploit or GitHub)? |

![Amazon Inspector findings list with scores](/assets/devsecops/week1/Pasted%20image%2020260523203025.png)

### The Real-World Impact

- A Critical CVE on an isolated backend database inside a private subnet without internet access will have its Inspector score adjusted downward because attackers cannot reach it directly.
- The exact same CVE on an internet-facing public web server with an exploit published online will receive a maximum 10.0 Inspector score and trigger urgent alarms.

---

## Container Vulnerability Scanning in Amazon ECR

When you push a container image to Amazon ECR, Inspector unpacks the image layers and analyzes both the base operating system packages (like glibc or OpenSSL) and language dependencies.

### Continuous Re-Scanning: The Silent Risk

A common misconception among beginners:

> *"I scanned my Docker image before deploying it last month and it had zero vulnerabilities, so my container is secure."*

New CVEs are published every single day. An image that was 100% clean in January might have three Critical CVEs discovered in March.

Amazon Inspector performs **continuous scanning** on ECR repositories. Whenever a new CVE is disclosed, Inspector retroactively re-evaluates all stored images and alerts you immediately if an existing image in production has become vulnerable.

### Fixing Container Findings: The DevSecOps Pattern

Never attempt to patch a container by running `apt update` inside a live running container. Containers should be immutable.

Instead, fix the root cause in your `Dockerfile`:

```dockerfile
# BEFORE: Using an outdated, bloated base image with 80+ known CVEs
FROM python:3.9

# AFTER: Using an updated, slim base image with regular security updates
FROM python:3.12-slim
```

Update your base image, rebuild the container in your CI/CD pipeline, run your tests, and deploy the new image tag.

---

## AWS Lambda Function Scanning

For serverless architectures, Inspector scans both the function code dependencies and Lambda layers:

```txt
# requirements.txt - BEFORE (Vulnerable to CVE-2023-32681)
requests==2.25.0

# requirements.txt - AFTER (Patched version)
requests==2.31.0
```

When you update your dependency version in your requirements file and redeploy via SAM, Terraform, or the Serverless Framework, Inspector re-evaluates the function and automatically closes the finding.

---

## Prioritization Strategy: How to Triage Without Losing Your Mind

When facing a long list of findings across your accounts, use this practical triage matrix:

1. **Priority 1 (Fix within 24-48 hours):** Inspector Score 9.0 - 10.0, public exploit available, and network reachable from the internet.
2. **Priority 2 (Fix within 14 days):** High severity (7.0 - 8.9) with public exploits available.
3. **Priority 3 (Fix within 30 days):** Critical severity without known public exploits.
4. **Priority 4 (Regular maintenance cycle):** Medium and Low severity findings addressed during scheduled library upgrades.

---

## Hands-on Walkthrough: Finding and Patching an EC2 Vulnerability

Let's walk through identifying an unpatched package on an EC2 instance and watching Inspector close the finding:

### Step 1: Locate the Vulnerability in Inspector

1. Open **Amazon Inspector -> Findings**.
2. Filter by your EC2 instance ID.
3. Review the top finding: note the CVE ID, the affected package name (e.g. `curl`), and the **Fixed version**.

### Step 2: Connect via AWS Systems Manager Session Manager

Avoid using raw SSH keys. Connect securely using SSM Session Manager:

```bash
aws ssm start-session --target i-0123456789abcdef0
```

### Step 3: Upgrade the Affected Package

Run the package manager update command:

```bash
# On Amazon Linux 2023 / RHEL
sudo dnf update curl -y

# On Ubuntu / Debian
sudo apt-get update && sudo apt-get --only-upgrade install curl -y
```

### Step 4: Verify Automatic Closure

Within a few minutes of package installation, the SSM Agent sends an updated software inventory to Inspector. Inspector validates that the installed package version is now equal to or greater than the fixed version, and moves the finding state to **CLOSED** automatically.

---

## Junior Pitfalls to Avoid

1. **Missing SSM IAM Permissions:** If your EC2 instances do not show up under Amazon Inspector coverage, 99% of the time it is because the instance lacks an IAM role with the `AmazonSSMManagedInstanceCore` policy attached. Without this policy, the instance cannot communicate with AWS Systems Manager.
2. **Ignoring Docker Base Image Hygiene:** Using `latest` tags or full development base images (e.g. `ubuntu:latest` or `python:3.9`) pulls in hundreds of unnecessary utilities (compilers, debuggers, curl) that inflate your vulnerability count. Always use minimal base images like `-slim`, `-alpine`, or distroless images.
3. **Treating Vulnerability Management as a One-Time Task:** Security is not a checklist item you complete before launch. Set up automated Slack or email alerts from Security Hub so engineers are notified whenever a new Critical finding appears in ECR or EC2.

---

## Key Takeaways

- Amazon Inspector provides agentless, continuous vulnerability scanning across EC2 instances, ECR container repositories, and Lambda functions.
- The Inspector Score intelligently factors in exploit availability and actual network reachability, saving you from alert fatigue.
- ECR continuous scanning catches newly disclosed CVEs in images that were deployed weeks or months ago.
- Remediate container vulnerabilities at the source (updating Dockerfile base images) rather than patching live containers.
- Ensure all EC2 instances have the SSM Agent active and the `AmazonSSMManagedInstanceCore` IAM role attached.

---

## References

<div class="references">
<ul>
  <li><a href="https://docs.aws.amazon.com/inspector/latest/user/what-is-inspector.html" target="_blank">Amazon Inspector v2 User Guide</a></li>
  <li><a href="https://docs.aws.amazon.com/inspector/latest/user/findings-understanding-severity.html" target="_blank">Understanding Amazon Inspector Finding Severity</a></li>
  <li><a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html" target="_blank">AWS Systems Manager Session Manager Overview</a></li>
  <li><a href="https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning-enhanced.html" target="_blank">Enhanced Container Scanning with Amazon Inspector and ECR</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
