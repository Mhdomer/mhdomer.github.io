---
layout: post
title: AWS Firewall Manager and Resource Access Manager — Centralized Policy Enforcement
date: 2026-07-05T10:00:00
categories:
  - SCS-C03 Cloud Security Speciality
  - Governance
tags:
  - aws
  - firewall-manager
  - ram
  - service-catalog
  - waf
  - governance
  - cloud-security
  - scs-c03
author: muhammed
description: SCS-C03 Domain 6 — Firewall Manager policies for WAF, Shield Advanced, and Security Groups; AWS RAM for resource sharing; Service Catalog for secure deployment
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## AWS Firewall Manager

**AWS Firewall Manager** centrally configures and manages security policies across accounts and resources in AWS Organizations.
Instead of configuring WAF, Shield, security groups, and Network Firewall separately in each account, Firewall Manager deploys and enforces policies from a single administrator account.

Firewall Manager requires:
- An AWS Organization
- A designated **Firewall Manager administrator account**
- AWS Config enabled in all accounts (for resource discovery)

```bash
# Set the Firewall Manager administrator account
aws fms associate-admin-account --admin-account 111122223333

# Verify the admin account
aws fms get-admin-account
```

---

## Firewall Manager Policy Types

| Policy Type | What it manages |
|---|---|
| **WAF policy** | AWS WAF Web ACLs on ALB, API GW, CloudFront across the org |
| **Shield Advanced policy** | Automatically protects resources with Shield Advanced |
| **Security group policy** | Audit and enforce security group rules across EC2, ALB, ENIs |
| **Network Firewall policy** | Deploy Network Firewall to VPCs across the org |
| **Route 53 Resolver DNS Firewall policy** | Enforce DNS filtering rules |
| **Third-party firewall policy** | Palo Alto, Fortigate deployed via Marketplace |

---

## WAF Policy — Deploy Across All Accounts

```bash
# Create a Firewall Manager WAF policy
aws fms put-policy \
  --policy '{
    "PolicyName": "org-wide-waf-policy",
    "SecurityServicePolicyData": {
      "Type": "WAFV2",
      "ManagedServiceData": "{\"type\":\"WAFV2\",\"preProcessRuleGroups\":[{\"ruleGroupArn\":null,\"overrideAction\":{\"type\":\"NONE\"},\"managedRuleGroupIdentifier\":{\"versionEnabled\":false,\"version\":null,\"vendorName\":\"AWS\",\"managedRuleGroupName\":\"AWSManagedRulesCommonRuleSet\"},\"ruleGroupType\":\"ManagedRuleGroup\",\"excludeRules\":[]}],\"postProcessRuleGroups\":[],\"defaultAction\":{\"type\":\"ALLOW\"},\"sampledRequestsEnabledForDefaultActions\":true}"
    },
    "ResourceType": "AWS::ElasticLoadBalancingV2::LoadBalancer",
    "ResourceTypeList": ["AWS::ElasticLoadBalancingV2::LoadBalancer", "AWS::ApiGateway::Stage"],
    "IncludeMap": {
      "ORG_UNIT": ["ou-abc123"]
    },
    "RemediationEnabled": true,
    "DeleteUnusedFMManagedResources": false
  }'
```

When `RemediationEnabled: true`, Firewall Manager automatically:
- Creates Web ACLs in accounts that do not have them
- Attaches them to matching resources
- Adds the required managed rule groups

---

## Security Group Policy

Firewall Manager can audit and enforce security group rules across all EC2 instances and ENIs in the organization.

**Common policies:**
- Audit: find security groups with inbound 0.0.0.0/0 on any port except 443
- Content audit: flag SGs that reference a specific port or CIDR that should be blocked
- Usage audit: find unused security groups

```bash
# Create a security group usage audit policy (find unused SGs)
aws fms put-policy \
  --policy '{
    "PolicyName": "unused-sg-audit",
    "SecurityServicePolicyData": {
      "Type": "SECURITY_GROUPS_USAGE_AUDIT",
      "ManagedServiceData": "{\"type\":\"SECURITY_GROUPS_USAGE_AUDIT\",\"deleteUnusedSecurityGroups\":false,\"coalesceRedundantSecurityGroups\":false}"
    },
    "ResourceType": "AWS::EC2::SecurityGroup",
    "IncludeMap": {"ORG_UNIT": ["ou-abc123"]},
    "RemediationEnabled": false
  }'

# Get Firewall Manager compliance report for a policy
aws fms list-compliance-status --policy-id policy-id-here
```

---

## Shield Advanced Policy

```bash
# Deploy Shield Advanced protection automatically to all ALBs and CloudFront distributions
aws fms put-policy \
  --policy '{
    "PolicyName": "shield-advanced-policy",
    "SecurityServicePolicyData": {
      "Type": "SHIELD_ADVANCED"
    },
    "ResourceType": "AWS::ElasticLoadBalancingV2::LoadBalancer",
    "ResourceTypeList": [
      "AWS::ElasticLoadBalancingV2::LoadBalancer",
      "AWS::CloudFront::Distribution",
      "AWS::Route53::HostedZone"
    ],
    "IncludeMap": {"ORG_UNIT": ["ou-prod-123"]},
    "RemediationEnabled": true
  }'
```

---

## Firewall Manager Notifications

Firewall Manager integrates with **SNS** to send notifications when a policy violation is detected.

```bash
# Configure SNS topic for Firewall Manager notifications
aws fms put-notification-channel \
  --sns-topic-arn arn:aws:sns:eu-west-1:123456789012:fms-violations \
  --sns-role-name arn:aws:iam::123456789012:role/fms-notification-role
```

---

## AWS Resource Access Manager (RAM)

**AWS RAM** allows you to share AWS resources across accounts within your organization — without creating copies.
Instead of duplicating resources, one account owns the resource and shares it to other accounts who use it as if they owned it.

### Resources That Can Be Shared with RAM

| Resource | Common Use Case |
|---|---|
| **VPC subnets** | Share private subnets with workload accounts — all VMs in a central VPC |
| **Transit Gateway** | Share TGW with spoke accounts for hub-and-spoke networking |
| **Route 53 Resolver rules** | Share DNS forwarding rules centrally |
| **License Manager configurations** | Track software licenses across accounts |
| **AWS Network Firewall policies** | Apply shared firewall policies |
| **Prefix lists** | Share managed IP lists (partner CIDRs, office IPs) |

```bash
# Share a VPC subnet with another account
aws ram create-resource-share \
  --name shared-private-subnets \
  --resource-arns arn:aws:ec2:eu-west-1:123456789012:subnet/subnet-private1 \
  --principals arn:aws:organizations::123456789012:ou/o-abc/ou-abc-123 \
  --allow-external-principals false

# List shares that have been shared with your account
aws ram get-resource-share-associations \
  --association-type PRINCIPAL \
  --resource-share-status ACTIVE

# Accept a resource share (if outside org auto-accept)
aws ram accept-resource-share-invitation \
  --resource-share-invitation-arn arn:aws:ram:eu-west-1:123456789012:resource-share-invitation/abc123
```

### Shared VPC Architecture

```
Networking account
    │ owns VPC + subnets
    │ RAM shares subnets to workload accounts
    ▼
Workload Account A (sees subnets, launches EC2 into them)
Workload Account B (sees subnets, launches RDS into them)
```

Benefits: centralized VPC management, shared Transit Gateway, no VPC peering needed between workload accounts.

---

## AWS Service Catalog

**Service Catalog** allows organizations to create and manage portfolios of approved CloudFormation templates.
End users (developers, teams) can self-provision pre-approved resources without needing IAM permissions to create them directly.
This enforces compliance — teams can only deploy what the security team has pre-approved.

### Key Concepts

| Term | Meaning |
|---|---|
| **Portfolio** | A collection of products shared with a set of users/groups |
| **Product** | A CloudFormation template — defines what can be deployed |
| **Provisioned product** | A deployed instance of a product (a live CloudFormation stack) |
| **Launch constraint** | An IAM role used to deploy the product — users don't need direct IAM permissions |
| **Tag options** | Required tags applied to all resources provisioned from a product |

```bash
# Create a portfolio
aws servicecatalog create-portfolio \
  --display-name "Security-Approved Infrastructure" \
  --provider-name "Security Team" \
  --description "Pre-approved templates that meet security requirements"

# Create a product (CloudFormation template)
aws servicecatalog create-product \
  --name "Encrypted S3 Bucket" \
  --product-type CLOUD_FORMATION_TEMPLATE \
  --owner "Security Team" \
  --provisioning-artifact-parameters '{
    "Name": "v1.0",
    "Description": "S3 bucket with KMS encryption and access logging",
    "Info": {"LoadTemplateFromURL": "https://s3.amazonaws.com/templates/encrypted-s3.yaml"},
    "Type": "CLOUD_FORMATION_TEMPLATE"
  }'

# Associate product with portfolio
aws servicecatalog associate-product-with-portfolio \
  --product-id prod-abc123 \
  --portfolio-id port-abc123

# Grant access to a portfolio (IAM role)
aws servicecatalog associate-principal-with-portfolio \
  --portfolio-id port-abc123 \
  --principal-arn arn:aws:iam::123456789012:role/developer-role \
  --principal-type IAM
```

### Service Catalog + Organizations

Share portfolios across the entire organization — developers in any account can deploy from pre-approved templates.

```bash
# Share portfolio with the entire org
aws servicecatalog enable-aws-organizations-access

aws servicecatalog create-portfolio-share \
  --portfolio-id port-abc123 \
  --organization-node Type=ORGANIZATION,Value=o-abc123
```

---

## IaC Security — CloudFormation Guard and cfn-lint

For teams deploying CloudFormation directly (not via Service Catalog), enforce security policies at the template level.

**CloudFormation Guard (cfn-guard)** validates templates against policy rules written in a Guard DSL.

```bash
# Install cfn-guard
pip install cloudformation-guard

# Write a rule (enforce S3 bucket encryption)
# rule s3_bucket_must_have_encryption {
#   AWS::S3::Bucket {
#     Properties.BucketEncryption exists
#   }
# }

# Validate a template
cfn-guard validate \
  --data template.yaml \
  --rules s3-security-rules.guard

# cfn-lint — lint for CloudFormation best practices
pip install cfn-lint
cfn-lint template.yaml
```

---

## Exam Key Points

- **Firewall Manager**: one admin account manages WAF, Shield, security groups, Network Firewall across the entire org — requires AWS Config in all accounts
- **Firewall Manager remediation**: when enabled, automatically creates and attaches policies to non-compliant resources; when disabled, generates findings only
- **RAM**: share resources without copying — subnets, TGW, resolver rules; resources in the sharing account, accessed from other accounts
- **Shared VPC with RAM**: workload accounts launch into centrally managed subnets — reduces VPC count, simplifies networking
- **Service Catalog**: developers self-provision from pre-approved templates; launch constraints provide the IAM role so developers don't need direct CloudFormation permissions
- **CloudFormation Guard**: validates templates against security policies before deployment — shift-left security for IaC

---

## Quick Reference

```bash
# Firewall Manager
aws fms list-policies --output table
aws fms get-policy --policy-id policy-id-here
aws fms list-compliance-status --policy-id policy-id-here

# RAM
aws ram list-resource-shares --resource-owner SELF
aws ram list-resource-shares --resource-owner OTHER-ACCOUNTS
aws ram get-resource-shares --resource-owner SELF --name shared-private-subnets

# Service Catalog
aws servicecatalog list-portfolios
aws servicecatalog search-products --output table
aws servicecatalog list-provisioned-products --output table
```
