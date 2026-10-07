---
layout: post
title: "Day 6: AWS WAF & Shield - Application Filtering and DDoS Defense"
date: 2026-03-06 10:00:00 +0800
categories:
  - DevSecOps
  - Week1
tags:
  - AWS
  - WAF
  - Shield
  - DDoS
  - CloudSecurity
author: muhammed
description: A practical walkthrough of AWS WAF for application-layer (L7) protection and AWS Shield for DDoS mitigation - how to safely deploy rules without breaking production.
toc: true
pin: false
math: false
mermaid: false
image: /assets/devsecops/WAF.png
---

## Stopping Attacks at the Edge: WAF vs Shield

Over the last few days, we covered detective controls: GuardDuty catches malicious activity inside your account, and Inspector finds unpatched vulnerabilities on your servers.

But what about stopping an attacker **before** their payload ever reaches your EC2 instances, containers, or databases?

This is where the perimeter defense duo comes in:

- **AWS Shield:** Absorbs volumetric Layer 3 and Layer 4 Distributed Denial of Service (DDoS) floods (SYN floods, UDP reflection) so your network pipes do not get choked.
- **AWS WAF (Web Application Firewall):** Inspects incoming HTTP/HTTPS traffic at Layer 7, filtering out malicious web attacks like SQL Injection (SQLi), Cross-Site Scripting (XSS), path traversal, and aggressive scraping bots.

```
+-------------------------------------------------------------+
|                     Incoming Public Traffic                 |
+-------------------------------------------------------------+
                              |
                              v
       [ AWS Shield: Absorbs L3/L4 Network DDoS Floods ]
                              |
                              v
     [ AWS WAF: Inspects L7 HTTP/HTTPS Payloads & Headers ]
             |                                    |
          Malicious                             Clean
             |                                    |
             v                                    v
       [ 403 Forbidden ]                 [ ALB / CloudFront ]
                                                  |
                                                  v
                                         [ App Servers / DB ]
```

### The Highway vs Baggage Inspection Analogy

To keep their roles crystal clear:

- **Security Groups & Shield (Layer 3/4):** The highway traffic police. They verify license plates, source IPs, and ports. If a million fake cars try to clog the highway entrance at once, Shield absorbs the traffic jam. But the traffic cops never open your trunk to see what you are carrying.
- **AWS WAF (Layer 7):** The airport customs and baggage scanner. WAF opens the HTTP packet, reads the request URI, inspects the headers (User-Agent, cookies), parses the JSON POST body, and searches for malicious payloads like `' OR 1=1 --` or `<script>alert(1)</script>`.

---

## AWS WAF Core Concepts

AWS WAF attaches directly to your public-facing entry points:
- Application Load Balancers (ALB)
- Amazon CloudFront distributions
- Amazon API Gateway
- AWS AppSync (GraphQL APIs)
- Amazon Cognito user pools

![Web ACLs console overview](/assets/devsecops/week1/Pasted%20image%2020260523211001.png)

### 1. Web ACL (Access Control List)

A Web ACL is the container that holds your security rules. You associate a Web ACL with an ALB or a CloudFront CDN distribution.

### 2. Rules and Actions

Each rule inside a Web ACL inspects a specific part of the incoming HTTP request. When a rule matches, WAF can execute one of four actions:
- **Allow:** Passes the request forward to your application.
- **Block:** Drops the connection immediately and returns an HTTP 403 Forbidden error.
- **Count:** Logs that a match occurred, increments metrics, but lets the request pass through untouched (essential for safe testing!).
- **CAPTCHA / Challenge:** Forces the client browser to solve an interactive puzzle or JavaScript silent challenge to weed out automated bots.

---

## AWS Managed Rule Groups: Instant OWASP Protection

Writing comprehensive regex rules from scratch to catch every flavor of SQL injection or remote code execution is difficult and error-prone.

AWS solves this with **AWS Managed Rules (AMR)**: pre-configured, battle-tested rule sets maintained directly by the AWS Threat Research Team.

![WAF managed rule groups overview](/assets/devsecops/week1/Pasted%20image%2020260523212557.png)

### Essential Managed Rule Groups

| Managed Rule Group Name | What It Defends Against |
| :--- | :--- |
| `AWSManagedRulesCommonRuleSet` | The core baseline covering the OWASP Top 10 (XSS, path traversal, command injection) |
| `AWSManagedRulesKnownBadInputsRuleSet` | Exploits with predictable signatures (Log4j / Log4Shell, Spring4Shell, Shellshock) |
| `AWSManagedRulesAmazonIpReputationList` | Known malicious IPs identified by AWS threat intelligence feeds |
| `AWSManagedRulesSQLiRuleSet` | Deep inspection of request parameters and bodies for database injection syntax |
| `AWSManagedRulesLinuxRuleSet` | Linux-specific attacks (e.g. attempting to read `/etc/passwd` or `/proc/self/environ`) |
| `AWSManagedRulesBotControlRuleSet` | Identifies and throttles automated web scrapers and crawlers |

---

## Writing High-Impact Custom WAF Rules

In addition to managed rules, you can create custom rules tailored to your application's unique threat model.

### 1. Rate-Based Rules (Anti-Brute Force)

A common junior mistake is leaving login or password-reset endpoints open to unlimited password guessing.

A rate-based rule tracks request frequency per client IP over a sliding 5-minute window. If any single IP sends more than 100 requests to `/api/v1/auth/login` within 5 minutes, WAF temporarily blocks that IP address:

```json
{
  "Name": "RateLimitLoginAPI",
  "Priority": 10,
  "Action": {
    "Block": {}
  },
  "Statement": {
    "RateBasedStatement": {
      "Limit": 100,
      "AggregateKeyType": "IP",
      "ScopeDownStatement": {
        "ByteMatchStatement": {
          "SearchString": "/api/v1/auth/login",
          "FieldToMatch": {
            "UriPath": {}
          },
          "TextTransformations": [
            {
              "Priority": 0,
              "Type": "LOWERCASE"
            }
          ],
          "PositionalConstraint": "EXACTLY"
        }
      }
    }
  },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "RateLimitLoginAPI"
  }
}
```

### 2. Geo-Blocking Rules

If your SaaS application is strictly targeted at users in Singapore and Malaysia, and you have zero legitimate business elsewhere, you can block or challenge incoming traffic originating from geographical regions known for high volumes of malicious scanning.

---

## The Golden Rule: Always Use "Count" Mode First

One of the most dangerous mistakes a junior engineer can make is deploying new WAF rules directly in **Block** mode in a production environment:

> *"I turned on the Common Rule Set in Block mode, and suddenly 15% of our legitimate customers started calling customer support because their checkout requests were getting blocked with HTTP 403 Forbidden!"*

Legitimate client requests (such as rich text inputs, markdown editors, or custom headers) frequently resemble attack payloads.

### The Safe Production Deployment Workflow

1. **Step 1:** Add the managed rule group to your Web ACL, but override the action to **Count**.
2. **Step 2:** Let the rule run in Count mode for 3 to 7 days during regular peak business hours.
3. **Step 3:** Review Amazon CloudWatch metrics and sample logs. Check which URI paths are triggering matches: are they real malicious scanners or your own mobile application?
4. **Step 4:** Add rule exclusions or scope-down statements for legitimate endpoints if needed.
5. **Step 5:** Once you have verified zero false positives, flip the rule action from Count to **Block**.

---

## AWS Shield Standard vs Shield Advanced

A common point of fear for beginners navigating the AWS console:

> *"Is AWS Shield going to charge me thousands of dollars?"*

Let's clear this up completely:

### AWS Shield Standard (Free & Automatic)

- **Cost:** $0.00 (Completely free, enabled automatically for every AWS customer).
- **Protection:** Defends against standard Layer 3 and Layer 4 volumetric attacks (SYN floods, UDP reflection, ICMP floods).
- **Where it runs:** Operates automatically at the edge on CloudFront, Route 53, and Elastic Load Balancing.
- **For 95% of businesses:** Shield Standard combined with AWS WAF is more than enough to handle common DDoS attacks.

### AWS Shield Advanced (Enterprise Tier)

- **Cost:** ~$3,000 / month per organization (with a 1-year commitment).
- **Key Features:**
  - 24/7 direct access to the specialized **AWS DDoS Response Team (DRT)** who can write custom mitigation rules on your behalf during an active attack.
  - **Financial Cost Protection:** If an attack causes your EC2 auto-scaling groups to scale up to 100 instances to handle massive traffic, AWS provides billing credits to reimburse the spike!
  - Automated application-layer DDoS mitigation and detailed real-time metrics.

---

## Hands-on Lab: Protecting an ALB with WAF

Let's walk through creating a Web ACL with the SQL Injection rule set and testing it safely:

### Step 1: Create the Web ACL

1. In the console, search for **WAF & Shield -> Web ACLs**.
2. Select your target region and click **Create web ACL**.
3. Name: `demo-app-waf`.
4. Click **Add rules -> Add managed rule groups**.
5. Locate `AWSManagedRulesSQLiRuleSet`, toggle it on, and set its action to **Count** for testing.
6. Set the Default Web ACL Action to **Allow**.
7. In the association step, attach the Web ACL to your test Application Load Balancer.

### Step 2: Test with a Simulated SQL Injection Payload

Send a curl request containing a classic SQL injection query parameter:

```bash
curl -i "http://your-test-alb-dns.amazonaws.com/products?search=1'+OR+'1'='1"
```

Because our rule is running in **Count** mode, the request returns HTTP 200 OK from your application.

### Step 3: Inspect Sampled Requests

1. In the AWS WAF console, click your Web ACL and open the **Sampled requests** tab.
2. Filter by the last 15 minutes.
3. You will see your curl request logged with the matching rule: `SQLi_QUERYARGUMENTS` marked with action `COUNT`.
4. Once verified, edit the rule group, change the override from Count to **Block**, and re-run the curl command:

```
HTTP/1.1 403 Forbidden
Server: awselb/2.0
Content-Type: text/html
Content-Length: 134

<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
</body>
</html>
```

The request was intercepted and killed at the edge before ever touching your backend code!

---

## Junior Pitfalls to Avoid

1. **Deploying Directly in Block Mode:** Always test rules in Count mode first. WAF false positives break user experience faster than code bugs.
2. **Forgetting Rate-Limiting on Sensitive Routes:** Attackers can brute force authentication endpoints without using malicious SQL payloads. Always protect `/login`, `/register`, and `/forgot-password` with rate-based rules.
3. **Subscribing to Shield Advanced by Accident:** Shield Standard is free. Do not click "Subscribe to Shield Advanced" unless your enterprise specifically approves the $3,000/month cost!
4. **Ignoring WAF Logging Costs:** Storing full WAF request bodies in Amazon CloudWatch Logs or S3 at scale can become expensive. Configure logging filters to only record dropped (blocked) requests in high-traffic environments.

---

## Key Takeaways

- Shield handles network Layer 3/4 floods; WAF inspects application Layer 7 HTTP payloads.
- Shield Standard is completely free and automatically protects every AWS account.
- AWS Managed Rule Groups provide instant protection against OWASP Top 10 vulnerabilities, known exploits, and bad bot traffic.
- Never deploy new rules directly to Block mode in production: run in Count mode first to inspect real-world traffic patterns.
- Rate-based rules are the simplest and most effective defense against credential stuffing and brute force attacks.

---

## References

<div class="references">
<ul>
  <li><a href="https://docs.aws.amazon.com/waf/latest/developerguide/what-is-aws-waf.html" target="_blank">AWS WAF Developer Guide</a></li>
  <li><a href="https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-list.html" target="_blank">AWS Managed Rule Groups Reference</a></li>
  <li><a href="https://docs.aws.amazon.com/waf/latest/developerguide/ddos-overview.html" target="_blank">AWS Shield DDoS Mitigation Whitepaper</a></li>
  <li><a href="https://aws.amazon.com/blogs/security/how-to-deploy-aws-waf-with-terraform/" target="_blank">AWS Blog: Deploying WAF via Terraform</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
