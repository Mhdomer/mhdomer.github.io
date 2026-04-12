---
layout: post
title: "TryHackMe: The Forgotten Access Key - Defending AWS Walkthrough"
date: 2026-04-12T10:00:00+03:00
categories:
  - TryHackMe
  - Defending AWS
tags:
  - tryhackme
  - defending-aws
  - aws
  - cloud-security
  - iam
  - access-keys
  - credential-hygiene
  - incident-response
  - oidc
  - blue-team
description: Technical walkthrough of The Forgotten Access Key room in TryHackMe's Defending AWS path. Deep dive into IAM access key lifecycle management, auditing stale credentials, programmatic access risks, automated key rotation, and emergency revocation workflows.
toc: true
pin: false
math: false
mermaid: true
image: https://cdn-images.tryhackme.com/room-icons/68baea2454c82afe90fd7020-1781605793152
---

In the previous room, **The Over-Privileged User**, we explored how excessive IAM policy permissions expand an attacker's blast radius. However, an over-privileged policy is only dangerous if an adversary can authenticate as that identity.

One of the most persistent attack vectors against cloud environments is the compromise of **static, long-lived IAM access keys**. Unlike web console logins, which are commonly defended by Multi-Factor Authentication (MFA) and single sign-on (SSO) session timeouts, programmatic access keys bypass console protections entirely. Once an access key is hardcoded into a public repository, embedded in an unencrypted script, or left on a decommissioned laptop, an attacker can authenticate to the AWS API from anywhere in the world.

This room, **The Forgotten Access Key**, examines how to audit stale credentials, identify dormant programmatic access keys, execute emergency credential revocation workflows, and transition infrastructure to modern keyless architectures using **OpenID Connect (OIDC)** and **AWS STS temporary credentials**.

```
+========================================================================================+
|                    THE DANGER OF THE FORGOTTEN ACCESS KEY                              |
+========================================================================================+
|  [KEY TYPE]          --> Long-lived IAM User Access Key (AKIA...)                      |
|  [VULNERABILITY]     --> Created 400+ days ago, unrotated, owner departed company      |
|  [INCIDENT]          --> Key leaked in public GitHub commit; adversary scans account    |
|  [DEFENSIVE ACTION]  --> Inactive status -> CloudTrail audit -> Keyless OIDC migration |
+========================================================================================+
```

---

## 1. Anatomy of AWS Programmatic Credentials

When an IAM user is granted programmatic access, AWS generates a cryptographic credential pair:
1. **Access Key ID:** A 20-character alphanumeric string serving as the public identifier (analogous to a username).
2. **Secret Access Key:** A 40-character string used to cryptographically sign API requests using the **AWS Signature Version 4 (SigV4)** protocol (analogous to a password).

### AKIA vs. ASIA Prefixes

AWS access keys carry identifiable prefixes that immediately reveal their lifecycle and security properties:

| Prefix | Credential Type | Lifespan | Risk Profile |
|---|---|---|---|
| `AKIA...` | **Long-term IAM User Access Key** | Permanent until manually deleted or rotated. | **High Risk:** Persists indefinitely; no automatic expiration; vulnerable to leakage. |
| `ASIA...` | **Temporary AWS STS Session Token** | Ephemeral (15 minutes to 12 hours). | **Low Risk:** Automatically expires; requires a companion session token; bounded exposure. |

```mermaid
flowchart LR
    subgraph LongLived["Long-Lived Key (AKIA...)"]
        A[Generated for IAM User] --> B[Stored on Developer Laptop / Config]
        B --> C[No Expiration Date]
        C --> D[Leaked in Git / Stolen via Malware]
        D -->|Valid Until Manually Revoked| E[Adversary Persistent Access]
    end

    subgraph Ephemeral["Temporary Token (ASIA...)"]
        F[Assumed via IAM Role / OIDC] --> G[AWS STS Issues Session Token]
        G --> H[Valid for 1 Hour]
        H --> I[Automatic Cryptographic Expiry]
        I -->|Expired Token Rejected| J[Adversary Access Neutralized]
    end

    classDef red fill:#7f1d1d,stroke:#ef4444,stroke-width:2px,color:#fff;
    classDef green fill:#064e3b,stroke:#10b981,stroke-width:2px,color:#fff;

    class A,B,C,D,E red;
    class F,G,H,I,J green;
```

---

## 2. Auditing Access Keys and Stale Credentials

The **Center for Internet Security (CIS) AWS Foundations Benchmark** establishes two mandatory controls regarding programmatic access keys:
- **Control 1.13:** Ensure access keys are rotated every **90 days or less**.
- **Control 1.14:** Ensure all unused access keys (not used for 45 or 90 days) are deactivated or deleted.

### Step 1: List Access Keys for an IAM User

```bash
aws iam list-access-keys --user-name former-contractor
```

Output:
```json
{
    "AccessKeyMetadata": [
        {
            "UserName": "former-contractor",
            "AccessKeyId": "AKIAIOSFODNN7EXAMPLE",
            "Status": "Active",
            "CreateDate": "2024-11-10T12:00:00Z"
        }
    ]
}
```

The `CreateDate` timestamp reveals this key was generated more than 500 days ago, directly violating the 90-day rotation policy.

### Step 2: Determine When the Key Was Last Used

Query the AWS IAM API to determine if the key is actively in use, dormant, or being leveraged by an attacker:

```bash
aws iam get-access-key-last-used --access-key-id AKIAIOSFODNN7EXAMPLE
```

Output:
```json
{
    "UserName": "former-contractor",
    "AccessKeyLastUsed": {
        "LastUsedDate": "2026-04-11T23:14:02Z",
        "ServiceName": "s3",
        "Region": "us-east-1"
    }
}
```

**Critical Forensic Indicator:** Although the contractor left the organization over a year ago, the access key was used within the last 24 hours to access Amazon S3 from region `us-east-1`.

---

## 3. Account-Wide Auditing: The IAM Credential Report

Auditing access keys user-by-user across an enterprise with hundreds of identities is inefficient. AWS provides the **Credential Report**, an account-wide downloadable CSV that catalogs the state of all passwords, MFA devices, and access keys.

```mermaid
sequenceDiagram
    autonumber
    actor Auditor as Security Auditor
    participant IAM as AWS IAM API

    Auditor->>IAM: aws iam generate-credential-report
    IAM-->>Auditor: State: COMPLETE
    Auditor->>IAM: aws iam get-credential-report
    IAM-->>Auditor: Returns Base64-Encoded CSV Content
    Auditor->>Auditor: Decodes Base64 to Plaintext CSV
    Auditor->>Auditor: Filters by access_key_1_last_rotated > 90 days
```

### Generating and Decoding the Credential Report via CLI

1. Request report generation:
   ```bash
   aws iam generate-credential-report
   ```

2. Retrieve and decode the CSV data:
   ```bash
   aws iam get-credential-report --query 'Content' --output text | base64 --decode > /tmp/credential_report.csv
   ```

3. Parse the report with Python or awk to identify unrotated active keys:
   ```bash
   awk -F',' '$9=="true" {print $1, $8, $10, $11}' /tmp/credential_report.csv
   ```

Columns inspected include:
- `user`: Identity name.
- `access_key_1_active`: Whether Key 1 is enabled (`true`/`false`).
- `access_key_1_last_rotated`: Creation or last rotation date.
- `access_key_1_last_used_date`: Date and time when Key 1 made an API call.

---

## 4. Emergency Incident Response: Compromised Access Key Revocation

When an access key is leaked, speed is essential. Adversaries use automated scanners that monitor public GitHub commits, scraping exposed keys within **under 60 seconds** to provision cryptocurrency miners or exfiltrate sensitive data.

Follow this standard four-phase containment workflow:

```mermaid
flowchart TD
    A[Alert: Exposed Access Key Detected] --> B[Phase 1: Immediate Deactivation<br>Set Status to Inactive]
    B --> C[Phase 2: Invalidate Active Sessions<br>Attach Explicit Deny Policy]
    C --> D[Phase 3: Forensic Audit<br>Query CloudTrail for Key Activity]
    D --> E[Phase 4: Key Deletion & Migration<br>Delete Key and Transition to OIDC/Role]

    classDef alert fill:#7f1d1d,stroke:#ef4444,stroke-width:2px,color:#fff;
    classDef step fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#fff;

    class A alert;
    class B,C,D,E step;
```

### Phase 1: Deactivate the Access Key Immediately

Do not delete the key immediately. Deactivating it stops all new API calls while preserving the key ID for CloudTrail log correlation:

```bash
aws iam update-access-key \
  --user-name former-contractor \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --status Inactive
```

Verify that the status is now `Inactive`:

```bash
aws iam list-access-keys --user-name former-contractor --query 'AccessKeyMetadata[*].[AccessKeyId,Status]' --output table
```

### Phase 2: Invalidate Active STS Sessions

If the attacker used the compromised access key to assume an IAM role, deactivating the access key does not instantly revoke already-issued temporary STS tokens. 

To immediately invalidate all active STS sessions, attach an inline revocation policy denying all actions issued prior to the current timestamp:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RevokeAllExistingSessions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "DateLessThan": {
          "aws:TokenIssueTime": "2026-04-12T10:00:00Z"
        }
      }
    }
  ]
}
```

### Phase 3: CloudTrail Forensic Investigation

Investigate all API calls initiated by the compromised key to determine what actions the attacker performed:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAIOSFODNN7EXAMPLE \
  --query 'Events[*].[EventTime,EventName,SourceIPAddress,Username]' \
  --output table
```

### Phase 4: Final Key Deletion

Once forensics and operational validation are complete, delete the key permanently:

```bash
aws iam delete-access-key \
  --user-name former-contractor \
  --access-key-id AKIAIOSFODNN7EXAMPLE
```

---

## 5. Modern Architecture: Eliminating Static Keys with OIDC

The most effective way to protect against compromised access keys is to **stop using static access keys entirely**.

For automated systems such as CI/CD pipelines (GitHub Actions, GitLab CI, Jenkins), AWS supports **OpenID Connect (OIDC)** identity federation. Instead of storing long-lived `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` secrets in GitHub repository settings, GitHub Actions authenticates directly using short-lived tokens.

```mermaid
sequenceDiagram
    autonumber
    participant GH as GitHub Actions Runner
    participant OIDC as GitHub OIDC Provider
    participant STS as AWS STS (Security Token Service)
    participant IAM as AWS IAM Role

    GH->>OIDC: Requests OIDC JSON Web Token (JWT)
    OIDC-->>GH: Issues signed JWT token
    GH->>STS: Calls sts:AssumeRoleWithWebIdentity (passes JWT)
    STS->>STS: Validates JWT signature with GitHub OIDC
    STS->>IAM: Checks Role Trust Policy (repo, branch conditions)
    STS-->>GH: Issues Temporary Credentials (ASIA..., valid 1 hour)
    GH->>GH: Executes Deployment via Temporary Tokens
```

### Example OIDC Role Trust Policy

This trust policy allows only the `main` branch of a specific GitHub repository to assume the deployment role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:initech-corp/web-app:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

---

## 6. Hands-On Lab Walkthrough & Incident Containment

In this lab, a high-severity security alert indicates that a stale access key belonging to an inactive employee (`dev-alice`) was detected executing unauthorized API calls from an unknown IP address.

### Step 1: Identify the Stale Credential and Extract Flag 1

List all IAM users in the account:

```bash
aws iam list-users --query 'Users[*].[UserName,UserId]' --output table
```

Inspect the access keys associated with `dev-alice`:

```bash
aws iam list-access-keys --user-name dev-alice
```

Output:
```json
{
    "AccessKeyMetadata": [
        {
            "UserName": "dev-alice",
            "AccessKeyId": "AKIAJXXFORGOTTENKEY88",
            "Status": "Active",
            "CreateDate": "2024-03-15T09:12:00Z"
        }
    ]
}
```

Check when the key was last used:

```bash
aws iam get-access-key-last-used --access-key-id AKIAJXXFORGOTTENKEY88
```

Output reveals recent unauthorized access:
```json
{
    "UserName": "dev-alice",
    "AccessKeyLastUsed": {
        "LastUsedDate": "2026-04-12T08:15:30Z",
        "ServiceName": "secretsmanager",
        "Region": "us-east-1"
    }
}
```

Inspect the user tags on `dev-alice` to locate the investigation flag:

```bash
aws iam list-user-tags --user-name dev-alice
```

Output:
```json
{
    "Tags": [
        {
            "Key": "IncidentNote",
            "Value": "THM{ST4L3_K3Y_L34K_D3T3CT3D}"
        }
    ]
}
```

**Question:** What is the forensic flag found during the investigation of the compromised stale access key?
**Answer:** `THM{ST4L3_K3Y_L34K_D3T3CT3D}`

---

### Step 2: Query CloudTrail to Uncover Attacker Activity

Search CloudTrail event logs to identify what secrets the adversary accessed:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAJXXFORGOTTENKEY88 \
  --query 'Events[*].[EventTime,EventName,SourceIPAddress]' \
  --output table
```

Output:
```
-------------------------------------------------------------------------------------
|                                    LookupEvents                                   |
-------------------------------------------------------------------------------------
|  2026-04-12T08:15:30Z  |  GetSecretValue  |  198.51.100.99                        |
|  2026-04-12T08:14:10Z  |  ListSecrets     |  198.51.100.99                        |
|  2026-04-12T08:13:00Z  |  DescribeAccount |  198.51.100.99                        |
-------------------------------------------------------------------------------------
```

The adversary used the key from source IP `198.51.100.99` to list secrets and call `GetSecretValue`.

---

### Step 3: Deactivate the Stale Access Key

Execute the emergency deactivation command to immediately cut off the adversary's API access:

```bash
aws iam update-access-key \
  --user-name dev-alice \
  --access-key-id AKIAJXXFORGOTTENKEY88 \
  --status Inactive
```

Verify that the key is inactive:

```bash
aws iam list-access-keys --user-name dev-alice --query 'AccessKeyMetadata[*].[AccessKeyId,Status]' --output table
```

Output confirms:
```
------------------------------------------------
|                ListAccessKeys                |
------------------------------------------------
|  AKIAJXXFORGOTTENKEY88  |  Inactive          |
------------------------------------------------
```

---

### Step 4: Delete the Deactivated Access Key

With the threat contained, delete the compromised credential permanently:

```bash
aws iam delete-access-key \
  --user-name dev-alice \
  --access-key-id AKIAJXXFORGOTTENKEY88
```

Verify deletion:

```bash
aws iam list-access-keys --user-name dev-alice
```

Output confirms the key array is now completely empty (`[]`).

---

### Step 5: Run Compliance Evaluator and Submit Final Flag

Run the automated incident response evaluation script to verify containment and extract the completion flag:

```bash
python3 /home/ubuntu/scripts/verify_key_remediation.py --user dev-alice
```

Output:
```
======================================================================
               INCIDENT CONTAINMENT & KEY HYGIENE AUDIT
======================================================================
[+] Checking Status of Compromised Access Key........... [REVOKED/DELETED]
[+] Validating CloudTrail Incident Scope Identification. [VERIFIED]
[+] Testing Stale Key Removal Across All Users.......... [CLEAN]
[+] Checking Credential Report Generation Status........ [PASSED]

[***] CONTAINMENT SUCCESSFUL! ACCESS KEY THREAT NEUTRALIZED! [***]
Completion Flag: THM{R0T4T3_0R_R3V0K3_K3Y_S3CUR1TY}
======================================================================
```

**Question:** What is the final completion flag returned after neutralizing the stale key?
**Answer:** `THM{R0T4T3_0R_R3V0K3_K3Y_S3CUR1TY}`

---

## 7. Complete Questions and Answers Summary

| Task | Question | Answer |
|---|---|---|
| **Task 1** | What prefix designates a permanent, long-lived IAM access key? | `AKIA` |
| **Task 1** | What prefix designates a temporary, short-lived session token issued by AWS STS? | `ASIA` |
| **Task 2** | According to the CIS AWS Foundations Benchmark, what is the maximum recommended rotation period for access keys? | `90 days` |
| **Task 3** | Which AWS CLI command retrieves the timestamp, service, and region where an access key was last active? | `aws iam get-access-key-last-used` |
| **Task 4** | What downloadable account-wide report provides a full inventory of passwords, MFA, and access key statuses? | `Credential Report` |
| **Task 5** | What identity federation protocol allows GitHub Actions to deploy to AWS without storing static access keys? | `OIDC` (or `OpenID Connect`) |
| **Task 6** | What was the forensic challenge flag discovered in dev-alice's user tags? | `THM{ST4L3_K3Y_L34K_D3T3CT3D}` |
| **Task 7** | What is the final completion flag returned after neutralizing the stale key? | `THM{R0T4T3_0R_R3V0K3_K3Y_S3CUR1TY}` |

---

## 8. Defensive Engineering & Production Hardening Checklist

- [ ] **Enforce 90-Day Key Rotation:** Implement AWS Config managed rule `access-keys-rotated` to automatically flag keys older than 90 days as non-compliant.
- [ ] **Automated Stale Key Deactivation:** Deploy an EventBridge scheduled rule running a Lambda function that queries the IAM Credential Report weekly, automatically deactivating keys unused for 45 days.
- [ ] **Migrate CI/CD Pipelines to OIDC:** Replace all repository-stored AWS access keys in GitHub Actions, GitLab CI, and CircleCI with OIDC role assumption.
- [ ] **Enable Amazon GuardDuty Credential Anomaly Detection:** Ensure GuardDuty is active to detect `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration` and `AnomalousBehavior:IAMUser/ConsoleLoginWithAccessKey`.
- [ ] **Require MFA for API Operations:** For administrative IAM users who must maintain programmatic keys, enforce policy conditions requiring active MFA session tokens (`"Condition": {"Bool": {"aws:MultiFactorAuthPresent": "true"}}`).

---

*Looking for more hands-on cloud defense guides? Check out the rest of the [Defending AWS Series](/categories/defending-aws/) or explore my [AWS Security Labs](/categories/aws-security-labs/) for advanced threat simulation and detection engineering tutorials.*

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
