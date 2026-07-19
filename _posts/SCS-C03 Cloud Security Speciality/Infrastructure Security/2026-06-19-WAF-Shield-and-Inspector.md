---
layout: post
title: AWS WAF, Shield, and Amazon Inspector — Edge Protection and Vulnerability Scanning
date: 2026-06-19T10:00:00
categories:
  - SCS-C03 Cloud Security Speciality
  - Infrastructure Security
tags:
  - aws
  - waf
  - shield
  - inspector
  - cloud-security
  - scs-c03
author: muhammed
description: SCS-C03 Domain 3 — WAF rule groups, managed rules, rate-based rules, Shield Advanced, and Inspector EC2/ECR/Lambda vulnerability scanning
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## AWS WAF — Web Application Firewall

**AWS WAF** filters HTTP/HTTPS traffic at Layer 7 before it reaches your application.
It can be attached to **CloudFront**, **Application Load Balancer**, **API Gateway**, or **AppSync**.
WAF evaluates every request against a set of rules and either allows, blocks, or counts it.

---

## WAF Core Concepts

| Component | Description |
|---|---|
| **Web ACL** | The top-level container — a set of rules applied to a resource |
| **Rule** | A condition + action (allow / block / count / CAPTCHA) |
| **Rule group** | A reusable collection of rules |
| **Statement** | The condition inside a rule (IP match, geo match, string match, regex, size) |
| **Default action** | What happens to requests that match no rule — allow or block |

### Rule Actions

| Action | Effect |
|---|---|
| `Allow` | Forward the request |
| `Block` | Return 403 Forbidden |
| `Count` | Log the request, do not block (useful for testing new rules) |
| `CAPTCHA` | Challenge the user with a CAPTCHA |
| `Challenge` | Silent browser challenge (JavaScript-based bot detection) |

---

## Rule Types

### IP Set Rules

```bash
# Create an IP set (blocklist)
aws wafv2 create-ip-set \
  --name blocked-ips \
  --scope REGIONAL \
  --ip-address-version IPV4 \
  --addresses "1.2.3.4/32" "5.6.7.8/32"

# Create an IP set (allowlist — e.g. your office)
aws wafv2 create-ip-set \
  --name office-allowlist \
  --scope CLOUDFRONT \
  --ip-address-version IPV4 \
  --addresses "203.0.113.0/24"
```

### Geo Match Rules

Block or allow requests based on country of origin.

```json
{
  "Name": "block-high-risk-countries",
  "Priority": 10,
  "Action": {"Block": {}},
  "Statement": {
    "GeoMatchStatement": {
      "CountryCodes": ["KP", "IR", "SY"]
    }
  },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "BlockHighRiskCountries"
  }
}
```

### Rate-Based Rules

Automatically block IPs that send more than N requests in a 5-minute window.
Essential for DDoS mitigation at the application layer.

```json
{
  "Name": "rate-limit-login",
  "Priority": 5,
  "Action": {"Block": {}},
  "Statement": {
    "RateBasedStatement": {
      "Limit": 100,
      "AggregateKeyType": "IP",
      "ScopeDownStatement": {
        "ByteMatchStatement": {
          "SearchString": "/login",
          "FieldToMatch": {"UriPath": {}},
          "TextTransformations": [{"Priority": 0, "Type": "LOWERCASE"}],
          "PositionalConstraint": "STARTS_WITH"
        }
      }
    }
  }
}
```

---

## Managed Rule Groups

AWS and AWS Marketplace partners publish pre-built rule groups.
The most important ones for the exam:

| Rule Group | What It Blocks |
|---|---|
| `AWSManagedRulesCommonRuleSet` | OWASP Top 10 — SQLi, XSS, command injection, path traversal |
| `AWSManagedRulesKnownBadInputsRuleSet` | Exploits, malformed requests, known attack signatures |
| `AWSManagedRulesSQLiRuleSet` | SQL injection patterns specifically |
| `AWSManagedRulesLinuxRuleSet` | Linux-specific attacks (path traversal, /etc/passwd, LFI) |
| `AWSManagedRulesAmazonIpReputationList` | IPs known for scanning, botnets, malicious activity |
| `AWSManagedRulesBotControlRuleSet` | Automated bot traffic, scrapers, credential stuffers |

```bash
# Create a Web ACL with managed rules
aws wafv2 create-web-acl \
  --name prod-web-acl \
  --scope REGIONAL \
  --default-action Allow={} \
  --rules '[
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 1,
      "OverrideAction": {"None": {}},
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "CommonRuleSet"
      }
    }
  ]' \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=prod-web-acl

# Associate with an ALB
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:eu-west-1:123456789012:regional/webacl/prod-web-acl/abc123 \
  --resource-arn arn:aws:elasticloadbalancing:eu-west-1:123456789012:loadbalancer/app/prod-alb/abc123
```

> 📸 **SCREENSHOT:** WAF → Web ACLs → select ACL → Rules tab.
> Show the rule priority list with managed rule groups and custom rules, each with their action (Count/Block/Allow override).

---

## WAF Logging

Enable WAF logging to S3, CloudWatch Logs, or Kinesis Data Firehose.
Log all requests — sampled logging only captures a subset.

```bash
aws wafv2 put-logging-configuration \
  --logging-configuration '{
    "ResourceArn": "arn:aws:wafv2:eu-west-1:123456789012:regional/webacl/prod-web-acl/abc123",
    "LogDestinationConfigs": ["arn:aws:s3:::my-waf-logs"],
    "LoggingFilter": {
      "DefaultBehavior": "KEEP",
      "Filters": [{
        "Behavior": "DROP",
        "Conditions": [{"ActionCondition": {"Action": "ALLOW"}}],
        "Requirement": "MEETS_ALL"
      }]
    }
  }'
```

---

## AWS Shield

**Shield Standard** is automatically enabled for all AWS accounts at no cost.
It protects against the most common layer 3 and 4 DDoS attacks (SYN floods, UDP reflection, etc.).

**Shield Advanced** is a paid service (~$3,000/month per organisation) that adds:

| Feature | Details |
|---|---|
| **Layer 7 DDoS mitigation** | Automatic application-layer attack mitigation via WAF |
| **DDoS Response Team (DRT)** | 24/7 access to AWS security experts during attacks |
| **Cost protection** | AWS credits you for EC2/ELB/CloudFront bill spikes caused by DDoS |
| **Attack diagnostics** | Detailed reports on detected and mitigated attacks |
| **Health-based detection** | Uses Route 53 health checks to detect attacks faster |
| **Proactive engagement** | DRT contacts you when they detect an attack on your protected resources |

### Protected Resources (Shield Advanced)

- CloudFront distributions
- Route 53 hosted zones
- Application and Network Load Balancers
- Elastic IPs (EC2 instances, NAT Gateways)
- Global Accelerator accelerators

```bash
# Enable Shield Advanced (account-level, 1-year commitment)
aws shield create-subscription

# Protect a resource
aws shield create-protection \
  --name prod-alb-protection \
  --resource-arn arn:aws:elasticloadbalancing:eu-west-1:123456789012:loadbalancer/app/prod-alb/abc123

# Enable proactive engagement (DRT will contact you)
aws shield update-proactive-engagement --proactive-engagement-status ENABLED
```

**Exam tip:** Shield Advanced + WAF + CloudFront is the standard defence-in-depth pattern for DDoS protection.
Shield handles volumetric (L3/L4), WAF handles application-layer (L7).

---

## Amazon Inspector

**Amazon Inspector** automatically discovers and scans your workloads for software vulnerabilities and unintended network exposure.

### What Inspector Scans

| Resource | What It Checks |
|---|---|
| **EC2 instances** | OS packages, software vulnerabilities (CVEs), network reachability |
| **ECR container images** | Vulnerabilities in OS packages and programming language packages (on push or continuously) |
| **Lambda functions** | Vulnerabilities in function code packages and Lambda layers |

Inspector uses the **National Vulnerability Database (NVD)** and vendor advisories.
It also uses **EPSS (Exploit Prediction Scoring System)** to prioritise findings by likelihood of exploitation.

> 📸 **SCREENSHOT:** Inspector → Dashboard.
> Show the critical and high finding counts, the coverage summary (EC2 instances, ECR images, Lambda functions), and the top 5 most critical findings with their EPSS scores.

---

## Inspector Finding Severity

Inspector uses a **combined score** based on CVSS base score and EPSS.

| Severity | Score Range |
|---|---|
| **Critical** | 9.0–10.0 |
| **High** | 7.0–8.9 |
| **Medium** | 4.0–6.9 |
| **Low** | 0.1–3.9 |

**EPSS score** (0–1) indicates probability of exploitation within 30 days.
Inspector sorts findings by risk — a medium CVSS score with a high EPSS score might need more urgent attention than a high CVSS with low EPSS.

---

## Inspector Multi-Account Setup

Enable Inspector in the **delegated administrator account**.
Inspector automatically scans all instances and images in member accounts.
Findings are aggregated in the admin account.

```bash
# Enable Inspector
aws inspector2 enable --resource-types EC2 ECR LAMBDA

# Enable as org admin
aws inspector2 enable-delegated-admin-account \
  --delegated-admin-account-id 111122223333

# Check coverage (what's being scanned)
aws inspector2 list-coverage \
  --filter-criteria '{"ec2InstanceTags": [{"comparison": "NOT_EQUALS", "key": "Inspector", "value": "excluded"}]}'
```

---

## Inspector + Security Hub + ECR Integration

- Inspector findings flow automatically into **Security Hub**
- ECR image scanning: Inspector scans every image on push and continuously re-evaluates against new CVEs — no manual trigger needed
- Inspector findings trigger **EventBridge** events for automated response

```bash
# Suppress a specific CVE across all resources (known false positive)
aws inspector2 create-filter \
  --name "suppress-cve-2021-44228-false-positive" \
  --action SUPPRESS \
  --filter-criteria '{
    "vulnerabilityId": [{"comparison": "EQUALS", "value": "CVE-2021-44228"}],
    "resourceTags": [{"comparison": "EQUALS", "key": "env", "value": "sandbox"}]
  }'
```

---

## WAF vs Shield vs Inspector — Exam Distinction

| | WAF | Shield | Inspector |
|---|---|---|---|
| **Protects against** | Application-layer attacks (SQLi, XSS, bots) | DDoS (volumetric + application) | Software vulnerabilities (CVEs) |
| **Layer** | L7 (HTTP) | L3/L4 + L7 (Advanced) | N/A — offline scanning |
| **Sits in front of** | CloudFront, ALB, API GW | CloudFront, ALB, Route 53, EIP | EC2, ECR, Lambda |
| **Requires agents** | No | No | No (agentless) |

---

## Quick Reference

```bash
# WAF
aws wafv2 list-web-acls --scope REGIONAL
aws wafv2 get-web-acl --name prod-web-acl --scope REGIONAL --id abc123
aws wafv2 get-sampled-requests --web-acl-arn ARN --rule-metric-name CommonRuleSet --scope REGIONAL --time-window StartTime=1718000000,EndTime=1718100000 --max-items 100

# Shield
aws shield list-protections
aws shield describe-attacks --start-time StartTime=2026-06-01T00:00:00Z,EndTime=2026-06-19T00:00:00Z
aws shield describe-subscription

# Inspector
aws inspector2 list-findings --filter-criteria '{"severity": [{"comparison": "EQUALS", "value": "CRITICAL"}]}'
aws inspector2 get-findings-report-status
aws inspector2 list-coverage
```
