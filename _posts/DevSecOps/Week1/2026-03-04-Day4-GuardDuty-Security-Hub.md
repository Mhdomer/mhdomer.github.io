---
layout: post
title: "Day 4: GuardDuty & Security Hub - Intelligent Threat Detection"
date: 2026-03-04 10:00:00 +0800
categories:
  - DevSecOps
  - Week1
tags:
  - AWS
  - GuardDuty
  - SecurityHub
  - ThreatDetection
  - CloudSecurity
author: muhammed
description: A practical walkthrough of AWS GuardDuty for intelligent threat detection and AWS Security Hub for centralizing security posture and compliance across your cloud environment.
toc: true
pin: false
math: false
mermaid: false
image: https://assets.community.aws/a/2rsPJnEigEzYHeQiyzyYFqujlmo/SecH.webp?imgSize=1000x525
---

## The Detective Layer: Taming the Security Tool Sprawl

On Day 3, we learned how CloudTrail acts as our audit camera and AWS Config records resource drift. But logs by themselves do not protect you. If an attacker compromises an EC2 instance at 2:00 AM and starts mining Monero or exfiltrating your customer database, CloudTrail will faithfully record every single API call, but nobody is sitting at their desk manually reading millions of JSON log lines in real time.

This brings us to the detection and aggregation layer:

- **AWS GuardDuty:** The automated **guard dog** sniffing network and identity traffic for active intrusions and abnormal behavior.
- **AWS Security Hub:** The **central command center** (single pane of glass) that aggregates alarms from GuardDuty, AWS Config, Amazon Inspector, IAM Access Analyzer, and third-party tools into one unified triage desk.

```
+-------------------------------------------------------------+
|                     AWS Security Hierarchy                  |
+-------------------------------------------------------------+
   [ GuardDuty ]     [ AWS Config ]     [ Amazon Inspector ]
   (Threats & C2)   (Resource Drift)    (Software CVEs)
          \                |                 /
           \               |                /
            v              v               v
       +---------------------------------------+
       |           AWS Security Hub            |
       |  Central Dashboard + Compliance Scores|
       +---------------------------------------+
                           |
                           v
              [ EventBridge -> PagerDuty/Slack ]
```

Let's break down how GuardDuty detects intrusions without installing agents, how Security Hub normalizes findings, and how to automate alerting as a junior engineer.

---

## AWS GuardDuty: The Intelligent Intrusion Detection System

GuardDuty is a fully managed threat detection service. What makes GuardDuty special is that it is completely **agentless**. You do not need to install software agents on your servers or configure complex log forwarding pipelines.

When enabled, GuardDuty taps directly into internal AWS telemetry feeds:

| Telemetry Stream | Intrusions Detected |
| :--- | :--- |
| **VPC Flow Logs** | Communicating with known Command & Control (C2) servers, port scanning, unusual outbound traffic spikes |
| **DNS Query Logs** | Domain Generation Algorithms (DGA), DNS tunneling for data exfiltration, lookups for phishing domains |
| **CloudTrail Management Events** | Unusual API calls, impossible travel logins, reconnaissance from malicious IP addresses |
| **CloudTrail S3 Data Events** | Anomalous bulk downloads, data exfiltration from private buckets |
| **EKS Audit Logs** | Compromised Kubernetes pods, privilege escalation in containers |
| **RDS & Lambda Activity** | Database brute force attempts, compromised Lambda functions making external calls |

![GuardDuty architectural overview](/assets/devsecops/week1/Pasted%20image%2020260523103341.png)

GuardDuty correlates these streams against AWS threat intelligence feeds, CrowdStrike intelligence, Proofpoint feeds, and machine learning anomaly detection baselines.

---

## Enabling GuardDuty Across the Organization

Enabling GuardDuty takes literally two clicks in the console.

1. Navigate to **AWS GuardDuty** in the management console.
2. Click **Get Started -> Enable GuardDuty**.

![GuardDuty protection plans console view](/assets/devsecops/week1/Pasted%20image%2020260523103441.png)

### Multi-Account Enrolment

In an AWS Organization, you designate a **Delegated Administrator** account (usually your dedicated Security Tooling account). From that administrator account, you enable GuardDuty and toggle **Auto-enable for all new accounts**. Any new developer sandbox or production account joined to your organization is automatically protected from the second it is created.

---

## Decoding GuardDuty Findings

GuardDuty findings are named using a structured taxonomy:

```
ThreatPurpose:ResourceType/ThreatFamilyName.DetectionMechanism!Artifact
```

For example:

- `CryptoCurrency:EC2/BitcoinTool.B`
  - *ThreatPurpose:* CryptoCurrency (Resource hijacking for crypto mining)
  - *ResourceType:* EC2 instance
  - *ThreatFamilyName:* BitcoinTool (Running mining software)
- `UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B`
  - *Meaning:* A console login succeeded from an unusual location or IP address never seen before for that user.
- `Recon:IAMUser/MaliciousIPCaller`
  - *Meaning:* API calls are being made using your IAM credentials from an IP address flagged on global threat intelligence lists.

![GuardDuty findings list overview](/assets/devsecops/week1/Pasted%20image%2020260523103524.png)

### Severity Scale

GuardDuty scores findings from `0.1` to `8.9`:

- **Low (0.1 - 3.9):** Suspicious or unusual behavior, but low confidence of malicious intent (e.g. port scan against an EC2 instance that was blocked by security groups).
- **Medium (4.0 - 6.9):** Activity that deviates from baseline behavior (e.g. an IAM user calling APIs from an unusual location).
- **High (7.0 - 8.9):** High-confidence active compromise (e.g. EC2 instance communicating with a known C2 server, cryptocurrency mining detected, or AWS credentials exfiltrated).

![GuardDuty finding detail view](/assets/devsecops/week1/Pasted%20image%2020260523104004.png)

---

## Testing with Sample Findings

You should never wait for a real attacker to verify that your alerts and dashboards work. GuardDuty provides a built-in generator for realistic sample findings:

1. In the GuardDuty console, open **Settings**.
2. Scroll down to **Sample findings** and click **Generate sample findings**.
3. Return to the **Findings** tab. You will see simulated findings prefixed with `[SAMPLE]`.

![GuardDuty generated sample findings](/assets/devsecops/week1/Pasted%20image%2020260523110623.png)

---

## Automated Alerting with Amazon EventBridge

A finding sitting silently in the console is useless if nobody sees it. We use **Amazon EventBridge** to catch High and Critical findings and push them immediately to our alerting channels.

Here is an EventBridge rule pattern matching any GuardDuty finding with a severity of 7.0 or greater:

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [
      {
        "numeric": [">=", 7]
      }
    ]
  }
}
```

Target this rule to an SNS topic that sends an email, triggers a Slack webhook via AWS Lambda, or opens an incident in PagerDuty.

---

## AWS Security Hub: The Central Command Center

If GuardDuty is our guard dog, **AWS Security Hub** is the central security operations desk.

### What Security Hub Does

1. **Aggregates Findings:** Collects alerts from GuardDuty, AWS Config, Amazon Inspector, Macie, IAM Access Analyzer, and third-party scanners.
2. **Normalizes into ASFF:** Every tool speaks a different language. Security Hub translates all findings into the standard **Amazon Security Finding Format (ASFF)**, so every finding has a unified schema.
3. **Continuous Compliance Benchmarking:** Compares your accounts against industry benchmarks (CIS AWS Foundations Benchmark, AWS Foundational Security Best Practices, PCI DSS) and generates a quantitative compliance score.

![Security Hub main dashboard overview](/assets/devsecops/week1/Pasted%20image%2020260523110448.png)

---

## Enabling Security Standards

When you enable Security Hub, activate these two essential security standards:

1. **AWS Foundational Security Best Practices (FSBP):** Curated AWS security controls that test real-world misconfigurations.
2. **CIS AWS Foundations Benchmark v1.4 / v3.0:** The gold standard compliance baseline accepted by enterprise security auditors worldwide.

![Security Hub standards compliance view](/assets/devsecops/week1/Pasted%20image%2020260523111612.png)

### Triaging Findings in One Place

In Security Hub, you can filter across all finding sources simultaneously:

![Security Hub aggregated findings list](/assets/devsecops/week1/Pasted%20image%2020260523133756.png)

Each finding follows a lifecycle workflow:

- `NEW`: Finding generated, waiting for triage.
- `NOTIFIED`: Engineering or security team has been assigned.
- `SUPPRESSED`: Verified false positive or accepted business risk.
- `RESOLVED`: Underlying issue remediated.

### Drilling Into Failed CIS Controls

When a compliance control fails (e.g. CIS 1.16 "Ensure IAM policies are attached only to groups or roles"), Security Hub highlights exactly which resources are failing, provides remediation steps, and includes direct links to fix the issue.

![CIS benchmark control evaluation view](/assets/devsecops/week1/Pasted%20image%2020260523134657.png)

![Failed CIS control remediation guide](/assets/devsecops/week1/Pasted%20image%2020260523134346.png)

---

## Hands-on Walkthrough: End-to-End Detection Flow

Let's test the complete pipeline from threat generation to centralized triage:

1. Enable GuardDuty and Security Hub in your test account.
2. In Security Hub, navigate to **Integrations** and verify that AWS GuardDuty is enabled.
3. In GuardDuty, go to **Settings** and generate sample findings.
4. Switch to **Security Hub -> Findings** and filter by `Product name: GuardDuty`.
5. Observe the sample findings flowing into Security Hub in ASFF format.
6. Select a finding, review the affected resource ARN and evidence, and update its workflow status to `SUPPRESSED` since it is a test run.

---

## Junior Pitfalls to Avoid

1. **Ignoring the 30-Day Free Trial Notice:** GuardDuty offers a generous 30-day free trial. During this trial, open the GuardDuty usage tab to inspect your estimated monthly cost before the trial expires. In accounts with massive VPC network traffic or huge S3 data lakes, data plan add-ons can be costly.
2. **Alert Fatigue / Alert Blindness:** If your team receives 500 email alerts every day for low-severity issues, everyone will set up an email filter and ignore everything. Route only High and Critical severity findings (severity `>= 7.0`) to urgent channels, and review Medium/Low findings during weekly triage.
3. **Not Using Suppression Rules:** If your developers run penetration testing tools or legitimate vulnerability scanners from a known IP, GuardDuty will continuously trigger alarms. Create suppression rules to filter out known testing activity rather than turning off detection entirely.
4. **Single-Region Setup:** Threats can occur in any region. Always enable GuardDuty and Security Hub across all supported AWS regions.

---

## Key Takeaways

- GuardDuty is an agentless, intelligent threat detection service analyzing VPC flow logs, DNS queries, and CloudTrail events.
- Security Hub acts as the central command center, normalizing findings from multiple services into ASFF and scoring compliance against CIS benchmarks.
- Use EventBridge to automate alerting for high-severity findings to Slack or PagerDuty.
- GuardDuty finding taxonomy (`ThreatPurpose:ResourceType/ThreatFamilyName`) allows fast triage during security incidents.
- Treat compliance scores as continuous health metrics: aim for at least 85% to 90% compliance on the CIS benchmark.

---

## References

<div class="references">
<ul>
  <li><a href="https://docs.aws.amazon.com/guardduty/latest/ug/what-is-guardduty.html" target="_blank">AWS GuardDuty User Guide</a></li>
  <li><a href="https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html" target="_blank">AWS Security Hub User Guide</a></li>
  <li><a href="https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-active.html" target="_blank">GuardDuty Active Finding Types Documentation</a></li>
  <li><a href="https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-cis-gems.html" target="_blank">CIS AWS Foundations Benchmark Controls in Security Hub</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
