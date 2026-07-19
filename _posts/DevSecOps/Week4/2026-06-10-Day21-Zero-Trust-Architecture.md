---
layout: post
title: "Week 4 — Day 21: Zero Trust Architecture"
date: 2026-06-10 10:00:00 +0800
categories:
  - DevSecOps
  - Week4
tags:
  - ZeroTrust
  - AWS
  - CloudSecurity
  - Architecture
  - DevSecOps
author: muhammed
description: A full walkthrough of Zero Trust architecture principles and how AWS implements them — IAM Identity Center, VPC Lattice, PrivateLink, and moving from perimeter security to identity-based access.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fwww.xenonstack.com%2Fhubfs%2Fzero-trust-security-architecture.png&f=1&nofb=1&ipt=96a39dd851293aa9b892a75bc2f90730eb868087c683b1103a72f36af306aac8
---

## The Problem With Perimeter Security

Traditional security assumed: **inside the network = trusted, outside = untrusted**. Build a strong firewall, and everything inside is safe.

This model broke when:
- Remote work dissolved the "inside"
- Cloud workloads span multiple providers and regions
- Breaches showed that attackers move laterally once they're "inside" — they don't stop at the perimeter
- SaaS tools mean sensitive data flows outside the perimeter constantly

**Zero Trust answer:** Never trust, always verify. Every request — regardless of where it comes from — must be authenticated, authorized, and continuously validated.

---

## Zero Trust Principles

NIST SP 800-207 defines Zero Trust around three core tenets:

### 1. Verify Explicitly
Every access request must be authenticated and authorized using all available data points:
- Identity (who is making the request?)
- Device health (is the device patched and managed?)
- Location (is this an expected location?)
- Service/workload (what is being accessed?)
- Data classification (how sensitive is the resource?)

### 2. Use Least Privilege Access
- Grant just-in-time, just-enough access
- Time-limit sessions
- Continuously analyze and trim excess permissions

### 3. Assume Breach
- Minimize blast radius — segment everything
- Encrypt all traffic, even on internal networks
- Use analytics to detect anomalies
- Never assume a previous authentication is still valid

> `[SCREENSHOT]` — *NIST Zero Trust architecture diagram or the AWS Zero Trust whitepaper diagram showing the policy engine, policy administrator, and policy enforcement points*

---

## Zero Trust vs Traditional Perimeter

| | Perimeter Security | Zero Trust |
|--|-------------------|-----------|
| Trust model | Trust by location | Trust by identity + context |
| Internal traffic | Assumed trusted | Verified every time |
| Lateral movement | Easy once inside | Blocked by micro-segmentation |
| VPN dependency | Required | Not required |
| User experience | Painful VPN | Seamless (SSO) |
| Breach impact | Entire network | Isolated blast radius |

---

## AWS Zero Trust Building Blocks

AWS doesn't have a single "Zero Trust" service. It's implemented by combining several services:

```
Identity        → IAM Identity Center (SSO), IAM roles, MFA
Device          → AWS Verified Access, device posture checks
Network         → VPC, Security Groups, PrivateLink, VPC Lattice
Workload        → Service-to-service auth via IAM roles, mTLS
Data            → KMS encryption, S3 bucket policies, Macie
Visibility      → CloudTrail, GuardDuty, Security Hub
```

---

## IAM Identity Center (SSO)

IAM Identity Center is AWS's implementation of centralized identity for human users across multiple AWS accounts.

**How it works:**
1. Connect your identity provider (Azure AD, Okta, Google Workspace, or built-in)
2. Define permission sets (collections of IAM policies)
3. Assign users/groups to accounts with specific permission sets
4. Users log in via SSO → get temporary credentials → no long-lived IAM users

> `[SCREENSHOT]` — *IAM Identity Center dashboard showing the connected identity source, the list of AWS accounts managed, and the permission sets defined*

**Setting up IAM Identity Center:**
1. AWS Console → IAM Identity Center → Enable
2. Choose identity source: External IdP (recommended) or Identity Center directory
3. Create permission sets: e.g., `ReadOnly`, `Developer`, `SecurityAuditor`
4. Assign users from your IdP to accounts with appropriate permission sets

> `[SCREENSHOT]` — *IAM Identity Center → Permission sets page showing defined sets (ReadOnly, Developer, SecurityAuditor) with their managed policies listed*

**User experience:** Users visit the AWS access portal URL → select account and role → get console access or CLI credentials (valid for 1-8 hours). No long-lived access keys.

---

## AWS Verified Access

Verified Access provides Zero Trust network access to internal applications — replacing VPN for browser-based apps.

**Traditional VPN:** User connects VPN → gets access to entire internal network → risk of lateral movement.

**Verified Access:** User request goes through Verified Access → policy engine checks identity (from IdP) + device posture (from Jamf, CrowdStrike, etc.) → only allows access to the specific app → no network-level access granted.

> `[SCREENSHOT]` — *AWS Verified Access console showing a Verified Access instance with attached endpoints (internal apps), trust providers (IdP + device management), and access policies*

**Policy example — allow only managed devices from engineering group:**
```
permit(principal, action, resource)
when {
  context.identity.groups.contains("engineering") &&
  context.device.trust_score >= 80
};
```

---

## VPC PrivateLink

PrivateLink exposes services privately within AWS without routing traffic over the public internet — even across accounts.

**Without PrivateLink:**
```
App in VPC A → public internet → SaaS/service → back to AWS
```

**With PrivateLink:**
```
App in VPC A → PrivateLink endpoint → service VPC (private, no internet traversal)
```

> `[SCREENSHOT]` — *VPC → Endpoints page showing a list of VPC endpoints — Interface endpoints for specific AWS services (S3, Secrets Manager, ECR) and their status (Available)*

**Creating a VPC endpoint for Secrets Manager (no public internet for secret retrieval):**
1. VPC → Endpoints → Create endpoint
2. Service name: `com.amazonaws.ap-southeast-1.secretsmanager`
3. VPC: select your VPC
4. Subnets: select private subnets
5. Security group: allow HTTPS (443) from your VPC CIDR
6. Create

After this, your Lambda/EC2 calls to Secrets Manager stay entirely within the AWS network.

> `[SCREENSHOT]` — *VPC Endpoint detail page showing the Secrets Manager endpoint with its DNS names, VPC, subnets, and the security group attached*

---

## VPC Lattice — Service-to-Service Zero Trust

VPC Lattice is AWS's service mesh for service-to-service communication with built-in auth. It handles:
- Service discovery across VPCs and accounts
- Auth policies (IAM-based) between services
- Traffic management (routing, retries, timeouts)
- Observability (access logs per service)

**Traditional internal service mesh:**
- Service A calls Service B over an internal IP
- No authentication — any service on the network can call any other
- Lateral movement risk

**With VPC Lattice:**
- Every call requires an IAM identity
- Policy: Service A can call Service B on specific paths only
- Service C cannot call Service B unless explicitly allowed

> `[SCREENSHOT]` — *VPC Lattice console showing a service network with multiple services attached, each with their auth policies and access log settings*

**Auth policy example — allow only specific service account:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789:role/payment-service-role"
      },
      "Action": "vpc-lattice-svcs:Invoke",
      "Resource": "arn:aws:vpc-lattice:...:service/svc-payment/path/api/charge"
    }
  ]
}
```

Only the `payment-service-role` can invoke the `/api/charge` path — nothing else on the network can call it.

---

## Micro-Segmentation with Security Groups

Security Groups are the enforcement point for Zero Trust networking inside AWS. Applied correctly, they prevent lateral movement.

**Anti-pattern — flat network:**
```hcl
# All instances can talk to all instances in the VPC
resource "aws_security_group_rule" "allow-all-internal" {
  cidr_blocks = ["10.0.0.0/8"]
  from_port   = 0
  to_port     = 65535
  protocol    = "-1"
}
```

**Zero Trust pattern — reference security groups directly:**
```hcl
# Web tier can only receive from the ALB
resource "aws_security_group_rule" "web-from-alb" {
  source_security_group_id = aws_security_group.alb.id
  from_port   = 8080
  to_port     = 8080
  protocol    = "tcp"
}

# App tier can only receive from the web tier
resource "aws_security_group_rule" "app-from-web" {
  source_security_group_id = aws_security_group.web.id
  from_port   = 3000
  to_port     = 3000
  protocol    = "tcp"
}

# DB can only receive from the app tier
resource "aws_security_group_rule" "db-from-app" {
  source_security_group_id = aws_security_group.app.id
  from_port   = 5432
  to_port     = 5432
  protocol    = "tcp"
}
```

> `[SCREENSHOT]` — *AWS Console → Security Groups → the app tier security group showing inbound rules with source as the web tier security group ID (sg-xxxxx), not a CIDR range*

Now even if the web tier is compromised, the attacker can't reach the DB directly — they'd have to go through the app tier.

---

## Encrypting Internal Traffic (mTLS)

Zero Trust requires encrypting traffic even inside the network — assume the network is hostile.

**mTLS (mutual TLS):** Both client and server present certificates. Neither can impersonate the other.

In ECS/EKS, use AWS Certificate Manager Private CA to issue internal certificates:
1. ACM Private CA → Create CA (root or subordinate)
2. Issue certificates for each service
3. Configure your service to present and validate certificates

For Kubernetes, use cert-manager with ACM integration or a service mesh (Istio, Linkerd) to handle mTLS automatically between all pods.

> `[SCREENSHOT]` — *ACM Private CA console showing a private root CA in ACTIVE state, with the issued certificate count and the CA's ARN*

---

## Zero Trust Maturity Model

CISA defines a Zero Trust maturity model with three stages:

| Stage | Description |
|-------|-------------|
| Traditional | Implicit trust, perimeter-focused, static policies |
| Advanced | Identity-based access, some automation, limited visibility |
| Optimal | Fully dynamic policies, continuous validation, ML-driven anomaly detection |

**Where to start for AWS environments:**

1. **Enable IAM Identity Center** — eliminate long-lived IAM users for humans
2. **Enforce MFA everywhere** — use SCPs to require MFA for sensitive actions
3. **Replace CIDR rules with SG references** — micro-segment by tier, not IP range
4. **Create VPC endpoints** for AWS services — keep traffic off the internet
5. **Enable GuardDuty + Security Hub** — continuous verification of what's happening
6. **Rotate credentials aggressively** — short-lived tokens via Secrets Manager + IAM Identity Center

---

## Lab — Implement Micro-Segmentation with Security Groups

**Objective:** Replace a permissive `0.0.0.0/0` internal rule with security group references.

1. Identify any security group rules using broad CIDR ranges for internal traffic:

```bash
aws ec2 describe-security-groups \
  --query "SecurityGroups[*].{ID:GroupId,Rules:IpPermissions[?IpRanges[?CidrIp=='0.0.0.0/0']]}" \
  --output table
```

> `[SCREENSHOT]` — *Terminal showing the aws ec2 describe-security-groups output listing security groups with 0.0.0.0/0 rules*

2. For each overly permissive rule, replace the CIDR with the source security group ID in Terraform
3. Plan + apply the change
4. Verify connectivity still works between the intended tiers
5. Verify a "lateral" connection attempt is blocked — try connecting from web tier directly to DB port

> `[SCREENSHOT]` — *Terminal showing a telnet/nc connection attempt from web tier to DB port timing out — confirming lateral movement is blocked*

---

## Key Takeaways

- Zero Trust is not a product — it's a strategy implemented by combining multiple controls
- "Assume breach" means designing your network so a compromised component can't reach everything else
- IAM Identity Center eliminates long-lived human credentials — the single biggest Zero Trust win in AWS
- Security group references (not CIDRs) are the practical implementation of micro-segmentation in AWS
- VPC endpoints keep traffic to AWS services off the public internet
- VPC Lattice brings IAM-based auth to service-to-service communication — eliminating implicit internal trust

---

## References

<div class="references">
<ul>
  <li><a href="https://csrc.nist.gov/publications/detail/sp/800-207/final" target="_blank">NIST SP 800-207 — Zero Trust Architecture</a></li>
  <li><a href="https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html" target="_blank">AWS IAM Identity Center</a></li>
  <li><a href="https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html" target="_blank">AWS PrivateLink</a></li>
  <li><a href="https://docs.aws.amazon.com/vpc/latest/lattice/what-is-vpc-lattice.html" target="_blank">AWS VPC Lattice</a></li>
  <li><a href="https://www.cisa.gov/zero-trust-maturity-model" target="_blank">CISA Zero Trust Maturity Model</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
