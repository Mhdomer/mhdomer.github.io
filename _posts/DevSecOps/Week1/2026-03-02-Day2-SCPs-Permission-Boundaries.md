---
layout: post
title: "Day 2: AWS Organizations & SCPs - Multi-Account Guardrails"
date: 2026-03-02 10:00:00 +0800
categories:
  - DevSecOps
  - Week1
tags:
  - AWS
  - IAM
  - SCP
  - Organizations
  - CloudSecurity
author: muhammed
description: A practical walkthrough of AWS Organizations, Service Control Policies (SCPs), and IAM Permission Boundaries - org-level guardrails that prevent privilege escalation.
toc: true
pin: false
math: false
mermaid: false
image: https://v1.maturitymodel.security.aws.dev/en/scps1.png
---

## The Question That Baffled Me as a Beginner

After learning IAM on Day 1, you feel like you have a handle on AWS access control: you write JSON policies, attach them to roles, and enforce least privilege.

But then in any real enterprise or team environment, you run into this question:

> *"What happens if a developer or compromised credential in our development account has `AdministratorAccess`? Can they turn off CloudTrail logging to cover their tracks, delete all backups, or spin up 50 expensive GPU instances in an unmonitored region?"*

In a single AWS account, someone with `AdministratorAccess` can do whatever they want. They can change policies, delete logs, and bypass individual IAM rules.

This is where **AWS Organizations** and **Service Control Policies (SCPs)** enter the picture. They introduce account-level guardrails that **cannot be overridden by anyone inside that account**, not even an administrator with full admin rights.

Today, we will break down how AWS Organizations work, how SCPs enforce boundaries across multiple accounts, how Permission Boundaries prevent privilege escalation, and the practical gotchas you must know as a junior security engineer.

---

## AWS Organizations: The Big Picture

Before diving into SCPs, we need to understand how multi-account AWS environments are structured.

AWS Organizations allows you to group multiple AWS accounts under a single **Management (Root) Account**. Inside the organization, you group accounts into **Organizational Units (OUs)**.

![AWS Organizations hierarchy overview](/assets/devsecops/week1/Pasted%20image%2020260521134554.png)

### Typical Multi-Account Structure

In production, you never run everything inside one massive AWS account. Instead, accounts are segregated by purpose:

```
Root (Management Account)
├── OU: Core / Security
│   ├── Log Archive Account (Centralized S3 logs)
│   └── Security Tooling Account (GuardDuty, Security Hub)
├── OU: Workloads
│   ├── Production Account
│   ├── Staging Account
│   └── Development Account
└── OU: Sandbox (Isolated experimental accounts with tight budget caps)
```

![Root and OUs structure console view](/assets/devsecops/week1/Pasted%20image%2020260521134050.png)

SCPs are attached to the Root, to specific OUs, or to individual accounts. Policies automatically flow downward: any policy attached to the Root applies to every account across the entire organization.

---

## The Landlord Analogy: How SCPs Actually Work

The cleanest mental model for understanding SCPs is the relationship between an apartment building landlord and a tenant:

- **The Landlord (AWS Organization Management Account):** Owns the building and sets the master building rules in the lease: *"No pets, no tearing down structural walls, and no smoking inside."*
- **The Tenant (Account Administrator):** Rents Apartment 4B (the Development Account). Inside their apartment, the tenant can decorate however they want, buy furniture, and give house keys to friends (IAM policies).
- **The Conflict:** If the tenant writes a note on their fridge granting themselves permission to keep a pet tiger, does that matter? No. The landlord's building lease overrules whatever the tenant decides inside their apartment.

![SCP evaluation boundary diagram](/assets/devsecops/week1/Pasted%20image%2020260521134857.png)

### The Two-Key Lock Mental Model

For an API call to succeed inside any member account, AWS requires **two matching keys**:

1. **Key 1 (The SCP Ceiling):** Does the SCP permit the action (or at least not explicitly deny it)?
2. **Key 2 (The IAM Policy Door):** Does the identity's IAM policy explicitly grant the action?

```
+-------------------------------------------------------------+
|                     Effective Access Check                  |
+-------------------------------------------------------------+
                              |
                              v
        [ Does the SCP permit this action? ]
             |                      |
            Yes                     No  --> [ DENIED ]
             |
             v
   [ Does the IAM Policy allow this action? ]
             |                      |
            Yes                     No  --> [ DENIED ]
             |
             v
        [ ALLOWED ]
```

### What SCPs Do NOT Do

1. **SCPs DO NOT grant permissions on their own:** If an SCP says `Allow *`, that does not mean a developer can do everything. It only means the SCP is not blocking them. The user still needs an IAM policy that allows the action.
2. **SCPs NEVER apply to the Management Account:** The Root/Management account is completely immune to SCPs. This is why you should never run workloads or daily tasks inside the management account.
3. **SCPs DO NOT affect AWS Service-Linked Roles:** Internal AWS service-linked roles needed for infrastructure automation bypass SCP restrictions.

---

## SCP Strategies: Deny-List vs Allow-List

AWS provides two approaches for writing SCPs:

### 1. Deny-List Strategy (Recommended Baseline)

By default, AWS attaches an AWS-managed SCP called `FullAWSAccess` to all accounts, which allows everything (`Allow *:*`). You then attach custom SCPs with explicit `"Effect": "Deny"` blocks for dangerous actions.

- **Pros:** Safe to roll out, zero risk of accidentally breaking legitimate applications or pipelines.
- **Cons:** If AWS launches a brand new service, accounts can use it by default until you write a deny rule.

### 2. Allow-List Strategy

You remove `FullAWSAccess` and explicitly list only the specific services permitted in each OU (e.g. only S3, EC2, and RDS in the Dev OU).

- **Pros:** Maximum security and strict boundary control.
- **Cons:** Very high operational overhead. If a developer needs DynamoDB and it is not on the allow list, their deployment fails until an org admin updates the SCP.

---

## High-Impact SCP Examples

Here are the essential SCP guardrails every cloud security engineer should implement:

### 1. Protect Audit Logs: Deny Disabling CloudTrail

This policy ensures that nobody (even someone with full administrative permissions in a member account) can delete audit trails or stop logging.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDisablingCloudTrail",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:DeleteTrail",
        "cloudtrail:StopLogging",
        "cloudtrail:UpdateTrail"
      ],
      "Resource": "*"
    }
  ]
}
```

### 2. Prevent Rogue Accounts: Deny Leaving the Organization

An attacker who compromises a member account might attempt to detach that account from your AWS Organization so your central guardrails and billing monitoring disappear.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLeavingOrganization",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    }
  ]
}
```

### 3. Region Restriction: Block Unapproved AWS Regions

If your team only operates in Singapore (`ap-southeast-1`) and US East (`us-east-1`), you should block all other regions. Attackers frequently launch cryptocurrency miners or backdoor infrastructure in unmonitored regions like `af-south-1` or `eu-north-1` because security teams rarely monitor them.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnapprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "organizations:*",
        "route53:*",
        "cloudfront:*",
        "support:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "ap-southeast-1",
            "us-east-1"
          ]
        }
      }
    }
  ]
}
```

> **Important Note:** Notice the NotAction block! Global services like IAM, Route 53, and CloudFront operate out of us-east-1. If you block those globally without exempting them, your console and identity services will break.

![Region restriction rule console view](/assets/devsecops/week1/Pasted%20image%2020260521140413.png)

---

## IAM Permission Boundaries (Identity-Level Guardrails)

While SCPs operate at the **account level**, Permission Boundaries operate at the **identity level** (a specific user or role).

### The Real-World Problem It Solves

Imagine a senior engineer asks you:

> *"Our DevOps team needs to create IAM roles for their microservices in their deployment pipelines. But we do not want to give them full IAM admin access, because they could create an admin role and escalate their own privileges."*

This is the classic privilege escalation trap: if a user can call `iam:CreateRole` and attach policies to it, they can grant that role `AdministratorAccess` and assume it.

### The Solution: Permission Boundary

A **Permission Boundary** is an advanced IAM feature where you attach a managed policy to a user or role that acts as a ceiling. Even if the identity has an attached policy allowing `*:*`, their effective access can never exceed what the boundary specifies.

```
+-------------------------------------------------------------+
|                 Permission Boundary Evaluation              |
+-------------------------------------------------------------+
                              |
                              v
        [ Attached IAM Policy: What they want to do ]
                              |
                              v
    [ Permission Boundary: Maximum permissions allowed ]
                              |
                              v
    [ Result: Intersection of Policy AND Boundary ]
```

### Enforcing Boundaries with IAM Conditions

You allow developers to create roles, but with a strict condition: the role creation API call will fail unless they attach your approved permission boundary!

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowRoleCreationWithBoundaryOnly",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:AttachRolePolicy"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DevTeamPermissionBoundary"
        }
      }
    }
  ]
}
```

If a developer attempts to create a role without passing `DevTeamPermissionBoundary`, AWS returns `AccessDenied`.

---

## SCP vs Permission Boundary vs IAM Policy

Here is a quick reference table to keep their roles crystal clear:

| Feature | Service Control Policy (SCP) | Permission Boundary | IAM Policy |
| :--- | :--- | :--- | :--- |
| **Enforcement Scope** | Account or Organizational Unit | Single IAM User or IAM Role | User, Group, or Role |
| **Who Manages It** | Org Admin (Management Account) | IAM Security Admin | IAM Admin or DevOps Team |
| **Can Overrule Account Admin?**| Yes | No | No |
| **Grants Permissions?** | No (Only sets ceiling) | No (Only sets ceiling) | Yes |
| **Primary Goal** | Multi-account governance | Prevent privilege escalation | Define functional job permissions |

---

## Hands-on Lab: Attaching and Testing an SCP

Let's walk through creating a baseline security SCP and verifying that it blocks unauthorized modifications.

### Step 1: Create the SCP in AWS Organizations

1. Log into your AWS Organization Management Account.
2. Navigate to **AWS Organizations -> Policies -> Service control policies**.
3. Click **Create policy**.
4. Name the policy `baseline-security-controls` and enter the combined JSON policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDisablingAuditLogs",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:DeleteTrail",
        "cloudtrail:StopLogging",
        "cloudtrail:UpdateTrail"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyLeavingOrganization",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    }
  ]
}
```

![Attaching SCP in console](/assets/devsecops/week1/Pasted%20image%2020260521140703.png)

### Step 2: Attach the SCP to the Target OU

Navigate to the target OU (e.g. `Development`) under **AWS Organizations**, click the **Policies** tab, select **Service Control Policies**, and attach `baseline-security-controls`.

![Attaching baseline policy to OU](/assets/devsecops/week1/Pasted%20image%2020260521171544.png)

### Step 3: Test and Verify the Guardrail

Log into one of the member accounts inside that OU with full administrator credentials and run the following AWS CLI command in your terminal:

```bash
aws cloudtrail stop-logging --name production-audit-trail
```

The command immediately fails with an explicit denial:

```
An error occurred (AccessDeniedException) when calling the StopLogging operation: 
User: arn:aws:iam::123456789012:user/admin-user is not authorized to perform: 
cloudtrail:StopLogging on resource: arn:aws:cloudtrail:ap-southeast-1:123456789012:trail/production-audit-trail 
with an explicit deny in a service control policy.
```

![Access Denied CLI](/assets/devsecops/week1/Pasted%20image%2020260521173218.png)

Even though our user has the full `AdministratorAccess` IAM policy attached, the SCP intervened and killed the request!

![CloudTrail stop-logging denied console view](/assets/devsecops/week1/Pasted%20image%2020260521173545.png)

---

## Junior Pitfalls to Avoid

1. **Testing SCPs in the Management Account:** The Management Account is immune to SCPs. If you attach an SCP to the Root and test it using your management account credentials, nothing will be blocked. Always test in a dedicated member account!
2. **Forgetting Global Services in Region Restrictions:** If you write a region restriction SCP, always exempt global services (IAM, Route 53, CloudFront, STS, Support) using `NotAction`. Otherwise, users will not even be able to open the IAM console.
3. **Removing `FullAWSAccess` Too Early:** Never delete the default `FullAWSAccess` SCP until you have tested and verified exhaustive allow lists. Doing so prematurely can cause widespread production outages.
4. **Confusing Permission Boundaries with Managed Policies:** Remember that a boundary does not grant rights; it only caps them. Both an allow policy AND a boundary are required for an action to work.

---

## Key Takeaways

- SCPs set the organizational ceiling: no member account administrator can bypass an explicit Deny in an SCP.
- SCPs do not grant permissions; they only restrict what IAM policies can allow.
- Multi-account AWS architecture relies on SCPs for central guardrails (protecting CloudTrail, blocking unapproved regions, preventing leaving the org).
- Permission Boundaries allow safe delegation of IAM role creation without opening doors to privilege escalation.
- Always use deny-list SCPs for critical baseline security controls.

---

## References

<div class="references">
<ul>
  <li><a href="https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html" target="_blank">AWS Service Control Policies (SCPs) Documentation</a></li>
  <li><a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html" target="_blank">IAM Permission Boundaries Guide</a></li>
  <li><a href="https://github.com/ScaleSec/terraform_aws_scp" target="_blank">ScaleSec AWS SCP Examples Repository</a></li>
  <li><a href="https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html" target="_blank">AWS Best Practices for Multi-Account Environments</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
