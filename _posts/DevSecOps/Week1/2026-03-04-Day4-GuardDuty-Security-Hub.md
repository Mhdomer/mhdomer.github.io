---
layout: post
title: "Week 1 — Day 4: GuardDuty & Security Hub"
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
description: A full walkthrough of AWS GuardDuty for intelligent threat detection and Security Hub for centralizing and prioritizing security findings across your AWS environment.
toc: true
pin: false
math: false
mermaid: false
image: https://assets.community.aws/a/2rsPJnEigEzYHeQiyzyYFqujlmo/SecH.webp?imgSize=1000x525
---

## The Detection Layer

CloudTrail and Config tell you what happened and what things look like. GuardDuty and Security Hub tell you when something looks **malicious** or **wrong**.

- **GuardDuty** — continuously analyzes your environment for threats using ML and threat intelligence
- **Security Hub** — aggregates findings from GuardDuty, Config, Inspector, Macie, and third-party tools into one dashboard with compliance scoring

---

## AWS GuardDuty

### How GuardDuty Works

GuardDuty is a managed threat detection service. You enable it, and it silently analyzes:

| Data Source                  | What it detects                                             |
| ---------------------------- | ----------------------------------------------------------- |
| CloudTrail management events | Unusual API calls, credential abuse, account reconnaissance |
| CloudTrail S3 data events    | Suspicious S3 access patterns                               |
| VPC Flow Logs                | Network anomalies, communication with known bad IPs         |
| DNS logs                     | Domains used for C2, data exfiltration via DNS              |
| EKS audit logs               | Suspicious activity in Kubernetes clusters                  |
| RDS login events             | Brute force, unusual login patterns                         |
| Lambda network activity      | Lambda calling unexpected external endpoints                |

GuardDuty uses AWS threat intelligence feeds, ML models trained on AWS-wide data, and anomaly detection — all without you having to configure anything.

![h](/assets/devsecops/week1/Pasted%20image%2020260523103341.png)

---

### Enabling GuardDuty ( no free service )

1. AWS Console → GuardDuty → Get Started → Enable GuardDuty
2. That's it — no agents, no log routing needed
3. Optionally enable additional protection plans: S3 Protection, EKS Protection, RDS Protection, Lambda Protection

*GuardDuty → Settings → Protection plans showing S3, EKS, RDS, Lambda toggles — all enabled*

![h](/assets/devsecops/week1/Pasted%20image%2020260523103441.png)

For multi-account orgs: enable GuardDuty from the Organizations management account as a delegated administrator. All member accounts are automatically enrolled.

---

### Understanding Findings

GuardDuty findings follow a naming pattern:

```
ThreatPurpose:ResourceType/ThreatFamilyName.DetectionMechanism!Artifact
```

**Examples:**

| Finding | Meaning |
|---------|---------|
| `UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B` | Console login from unusual location/IP |
| `Recon:IAMUser/MaliciousIPCaller` | API calls from a known malicious IP doing reconnaissance |
| `CredentialAccess:IAMUser/AnomalousBehavior` | Credential use anomaly detected by ML |
| `Backdoor:EC2/C&CActivity.B` | EC2 communicating with known C2 server |
| `CryptoCurrency:EC2/BitcoinTool.B` | EC2 running crypto mining software |
| `Exfiltration:S3/MaliciousIPCaller` | S3 data accessed from known malicious IP |
| `Impact:EC2/PortScanFromEC2` | An EC2 instance is scanning other hosts |

Findings have severity levels: **Critical**, **High**, **Medium**, **Low**, **Informational**.

*GuardDuty → Findings list showing several findings with their severity badges, resource type, and last seen timestamp*

![g](/assets/devsecops/week1/Pasted%20image%2020260523103524.png)



---

### Investigating a Finding

Click any finding to see the full detail:

*GuardDuty → a specific finding expanded, showing the affected resource ARN, the actor IP/ASN, the action type, and the evidence section with supporting CloudTrail events*

![g](/assets/devsecops/week1/Pasted%20image%2020260523104004.png)
Key fields to look at:
- **Resource** — which instance, role, or bucket is affected
- **Action** — what the threat actor did
- **Actor** — source IP, ASN, and whether it's on a threat intelligence list
- **Evidence** — the underlying CloudTrail events that triggered the finding

---

### Triggering Sample Findings

In a test account, generate sample findings without real threats:

1.  Settings → Sample findings → Generate sample findings
2. All finding types appear with `[SAMPLE]` prefix

![g](/assets/devsecops/week1/Pasted%20image%2020260523110623.png)

---

### Suppression Rules

Not all findings are actionable. You can suppress known-good patterns:

1. GuardDuty → Findings → select a finding → Actions → Add suppression rule
2. Define filter criteria (e.g. suppress `CryptoCurrency` findings from a specific instance used for legitimate mining in your environment)

---

### Automating Response with EventBridge

GuardDuty → EventBridge → Lambda/SNS/Security Hub for automated response:

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [{ "numeric": [">=", 7] }]
  }
}
```

This EventBridge rule triggers on any High or Critical finding and can notify a Slack channel, open a PagerDuty incident, or invoke a Lambda to isolate the affected resource.

*EventBridge → Rules → a GuardDuty rule showing the event pattern and the target (SNS topic or Lambda)*

---

## AWS Security Hub

### What Security Hub Does

Security Hub is your **centralized security dashboard**. It:

1. Aggregates findings from GuardDuty, Inspector, Config, Macie, IAM Access Analyzer, and 60+ third-party integrations
2. Normalizes all findings into a standard format (ASFF — Amazon Security Finding Format)
3. Scores your posture against security standards (CIS, PCI DSS, AWS Foundational)
4. Lets you triage, investigate, and update finding status in one place

![h](/assets/devsecops/week1/Pasted%20image%2020260523110448.png)

---

### Enabling Security Hub

1. Security Hub → Go to Security Hub → Enable Security Hub
2. Enable standards: AWS Foundational Security Best Practices (enable this always), CIS AWS Foundations Benchmark
3. Enable integrations: GuardDuty, Inspector, Config, IAM Access Analyzer

*Security Hub → Security standards page showing three standards — FSBP, CIS v1.4, PCI DSS — with their enable/disable toggles and compliance scores*

![g](/assets/devsecops/week1/Pasted%20image%2020260523111612.png)

---

### Findings in Security Hub

All findings appear in one place regardless of source:

*Security Hub → Findings page filtered to Critical severity, showing findings from multiple sources (GuardDuty, Config, Inspector) with resource ARNs and workflow status*

![h](/assets/devsecops/week1/Pasted%20image%2020260523133756.png)

**Finding workflow states:**
- `NEW` — just came in, not yet reviewed
- `NOTIFIED` — team has been alerted
- `SUPPRESSED` — known issue, intentionally ignored
- `RESOLVED` — fixed

Update finding status to track your response:
1. Select finding → Actions → Update finding
2. Set workflow status → Resolved

---

### Security Standards & Controls

Security Hub checks your environment against hundreds of controls. Each control maps to a specific configuration requirement.

*Security Hub → CIS AWS Foundations Benchmark → expanded showing individual controls with Pass/Fail/Unknown status and affected resources*

![g](/assets/devsecops/week1/Pasted%20image%2020260523134657.png)

**Drilling into a failed control:**
1. Security Hub → Security standards → CIS → find a failed control
2. Click it → see which resources are failing and why
3. Click a resource → see the specific finding with remediation guidance

*Security Hub → a specific failed CIS control showing the description, remediation instructions, and list of non-compliant resources*

![g](/assets/devsecops/week1/Pasted%20image%2020260523134346.png)

---

### Insights

Insights are saved queries that group related findings for analysis.

Built-in insights include:
- AWS resources with the most findings
- AMIs generating the most findings
- EC2 instances with unresolved critical findings

 *Security Hub → Insights page showing the "AWS resources with the most findings" insight with a bar chart of top resources*

You can create custom insights — for example: *"All Critical findings on production-tagged resources that are still NEW after 24 hours"*.

---

## How They Work Together

```
Data Sources
    │
    ▼
GuardDuty ──────────────────────┐
AWS Config ──────────────────── │──► Security Hub ──► EventBridge ──► SNS/Lambda/SIEM
Amazon Inspector ──────────────┘
IAM Access Analyzer ───────────┘
```

**Recommended flow:**
1. GuardDuty detects a threat
2. Finding appears in Security Hub automatically
3. EventBridge rule triggers on High/Critical finding
4. Lambda isolates the affected EC2 / revokes IAM session / notifies security team

---

## Lab — Enable GuardDuty and Trigger a Sample Finding

1. Enable GuardDuty in your test account (if not already)
2. Enable Security Hub → enable FSBP standard
3. Enable the GuardDuty integration in Security Hub: Integrations → GuardDuty → Accept findings
4. GuardDuty → Settings → Generate sample findings
5. Go to Security Hub → Findings — you should see GuardDuty sample findings appear within a few minutes

*Security Hub → Findings filtered by "Product name = GuardDuty" showing the sample findings flowing in from GuardDuty*

6. Click a Critical finding → review the full detail, resource, remediation
7. Change the workflow status to `SUPPRESSED` (since these are test samples)

*Security Hub → finding detail panel showing the workflow status dropdown changed to Suppressed*

---

## Key Takeaways

- GuardDuty is zero-config threat detection — enable it in every account and every region on day one
- Finding names follow a pattern — learn the taxonomy to triage faster
- Security Hub is the single pane of glass — connect all your AWS security services to it
- Use EventBridge to automate response, not just notification
- Security Hub scores tell you your compliance posture at a glance — treat anything below 90% as a priority

---

## References

<div class="references">
<ul>
  <li><a href="https://docs.aws.amazon.com/guardduty/latest/ug/what-is-guardduty.html" target="_blank">AWS GuardDuty User Guide</a></li>
  <li><a href="https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html" target="_blank">AWS Security Hub User Guide</a></li>
  <li><a href="https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-active.html" target="_blank">GuardDuty Finding Types</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
