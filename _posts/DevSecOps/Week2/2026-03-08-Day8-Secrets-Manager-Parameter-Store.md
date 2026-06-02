---
layout: post
title: "Week 2 — Day 8: AWS Secrets Manager & Parameter Store"
date: 2026-03-08 10:00:00 +0800
categories:
  - DevSecOps
  - Week2
tags:
  - AWS
  - SecretsManager
  - ParameterStore
  - CloudSecurity
  - DevSecOps
author: muhammed
description: A full walkthrough of AWS Secrets Manager and SSM Parameter Store — how to store, rotate, and retrieve secrets securely without hardcoding credentials anywhere.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fdevio2023-media.developers.io%2Fwp-content%2Fuploads%2F2023%2F08%2Faws-systems-manager.png&f=1&nofb=1&ipt=a02cd36b5d7af05a4fd3e370e26ac7a893429252720ae25a097e21575a4adce1
---

## The Problem With Hardcoded Credentials

Hardcoded credentials in source code, environment variables in plaintext, or `.env` files committed to git — these are among the most common causes of cloud breaches. The fix is centralized secrets management.

AWS offers two services for this:
- **Secrets Manager** — for secrets that need automatic rotation (DB passwords, API keys)
- **SSM Parameter Store** — for configuration values and simpler secrets (no auto-rotation needed)

---

## AWS Secrets Manager

### What It Stores

- Database credentials (RDS, Redshift, DocumentDB)
- API keys and OAuth tokens
- SSH keys
- Any arbitrary text or binary up to 65KB

### Creating a Secret

1. Secrets Manager → Store a new secret
2. Secret type: **Credentials for Amazon RDS database** (or "Other type of secret" for custom)
3. Enter username and password
4. Select the RDS instance (Secrets Manager links the secret to the DB for rotation)
5. Name: `prod/myapp/db-password`
6. Store

 — *Secrets Manager → Store a new secret wizard showing the secret type selection (RDS credentials) and the username/password fields*

**Naming convention:** Use a path structure like `env/app/secret-name` — it helps with IAM policies and organization.

---

### Retrieving a Secret

**From CLI:**
```bash
aws secretsmanager get-secret-value \
  --secret-id prod/myapp/db-password \
  --query SecretString \
  --output text
```

**From Python (boto3):**
```python
import boto3
import json

client = boto3.client('secretsmanager', region_name='ap-southeast-1')

response = client.get_secret_value(SecretId='prod/myapp/db-password')
secret = json.loads(response['SecretString'])

db_user = secret['username']
db_pass = secret['password']
```

 — *Secrets Manager → a secret detail page showing the secret name, ARN, description, rotation status, and the "Retrieve secret value" button*

The application never stores the password — it calls Secrets Manager at runtime. The IAM role on the EC2/Lambda must have `secretsmanager:GetSecretValue` permission on that specific secret ARN.

---

### IAM Policy for Secret Access

```json
{
  "Effect": "Allow",
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "arn:aws:secretsmanager:ap-southeast-1:123456789:secret:prod/myapp/*"
}
```

Scope to the exact path — don't allow `*` on all secrets.

---

### Automatic Secret Rotation

Secrets Manager can automatically rotate secrets on a schedule using a Lambda function.

**For RDS (supported natively):**
1. Secrets Manager → select your secret → Edit rotation
2. Enable automatic rotation
3. Rotation schedule: every 30 days
4. Rotation function: use the AWS-provided template for your DB engine

 — *Secrets Manager → Edit rotation panel showing "Automatic rotation" toggled on, rotation schedule set to 30 days, and the Lambda function ARN selected*

**What happens during rotation:**
1. Lambda creates a new password on the database
2. Updates the secret in Secrets Manager
3. Tests the new credentials
4. Marks rotation complete

Your application using `get_secret_value` automatically gets the new password on the next call — no redeployment needed.

---

### Secrets Manager in ECS and Lambda

**ECS task definition — inject secret as environment variable:**
```json
{
  "secrets": [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:123456789:secret:prod/myapp/db-password"
    }
  ]
}
```

**Lambda — reference secret as environment variable:**
Same pattern — reference the secret ARN in the Lambda environment variables config.

 — *ECS task definition JSON editor showing the "secrets" block with a secret ARN reference for DB_PASSWORD*

---

## SSM Parameter Store

### Secrets Manager vs Parameter Store

| | Secrets Manager | Parameter Store |
|--|-----------------|-----------------|
| Cost | ~$0.40/secret/month | Free (Standard tier) |
| Auto rotation | Yes (built-in) | No |
| Cross-account | Yes | Limited |
| Max size | 65KB | 4KB (Standard), 8KB (Advanced) |
| Best for | DB passwords, API keys needing rotation | Config values, feature flags, non-rotating secrets |

Use **Secrets Manager** when you need rotation. Use **Parameter Store** for everything else.

---

### Parameter Types

| Type | Description |
|------|-------------|
| `String` | Plaintext value |
| `StringList` | Comma-separated list |
| `SecureString` | Encrypted with KMS |

Always use `SecureString` for sensitive values.

---

### Creating Parameters

1. Systems Manager → Parameter Store → Create parameter
2. Name: `/prod/myapp/api-key`
3. Tier: Standard
4. Type: SecureString
5. KMS key: use your CMK or the default `aws/ssm` key
6. Value: paste your secret value
7. Create

 — *SSM Parameter Store → Create parameter form showing the name "/prod/myapp/api-key", type SecureString, and KMS key selection*

---

### Retrieving Parameters

**CLI:**
```bash
# Get a single parameter (decrypted)
aws ssm get-parameter \
  --name /prod/myapp/api-key \
  --with-decryption \
  --query Parameter.Value \
  --output text

# Get all parameters under a path
aws ssm get-parameters-by-path \
  --path /prod/myapp/ \
  --with-decryption \
  --recursive
```

**Python:**
```python
import boto3

ssm = boto3.client('ssm', region_name='ap-southeast-1')

response = ssm.get_parameter(
    Name='/prod/myapp/api-key',
    WithDecryption=True
)
api_key = response['Parameter']['Value']
```

 — *Terminal showing the output of the get-parameter CLI command returning the decrypted value*

---

### Parameter Store in ECS and Lambda

**ECS task definition:**
```json
{
  "secrets": [
    {
      "name": "API_KEY",
      "valueFrom": "arn:aws:ssm:ap-southeast-1:123456789:parameter/prod/myapp/api-key"
    }
  ]
}
```

The same `secrets` block in ECS task definitions works for both Secrets Manager and Parameter Store — just change the ARN format.

---

## Lab — Store and Retrieve a DB Password

**Objective:** Store a fake DB password in Secrets Manager, retrieve it in a Python script.

1. Secrets Manager → Store a new secret → Other type of secret
2. Key: `username`, value: `dbadmin`
3. Key: `password`, value: `S3cr3tP@ssword!`
4. Name: `lab/testapp/db-creds` → Store

 — *Secrets Manager → the newly created secret showing its name, ARN, and "Last retrieved" timestamp*

5. Create a Python script locally:

```python
import boto3
import json

client = boto3.client('secretsmanager', region_name='ap-southeast-1')

response = client.get_secret_value(SecretId='lab/testapp/db-creds')
creds = json.loads(response['SecretString'])

print(f"User: {creds['username']}")
print(f"Pass: {creds['password']}")
```

6. Run the script — confirm it retrieves the values without them being in the code

 — *Terminal showing the Python script output printing the username and password retrieved from Secrets Manager*

7. Go back to Secrets Manager → enable rotation for the secret using the Lambda template
8. Trigger rotation manually: Actions → Rotate secret immediately
9. Re-run the script — it should return the new rotated password

 — *Secrets Manager → secret detail showing rotation enabled, last rotated timestamp updated, and rotation status "Successful"*

---

## Key Takeaways

- Never hardcode credentials — retrieve them at runtime from Secrets Manager or Parameter Store
- Use Secrets Manager when you need automatic rotation — especially for DB passwords
- Use Parameter Store for non-rotating config values — it's free at the Standard tier
- Scope IAM permissions to specific secret paths — never `secretsmanager:GetSecretValue` on `*`
- ECS and Lambda can inject secrets directly via the `secrets` block — no custom code needed for basic cases

---

## References

<div class="references">
<ul>
  <li><a href="https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html" target="_blank">AWS Secrets Manager User Guide</a></li>
  <li><a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html" target="_blank">SSM Parameter Store Guide</a></li>
  <li><a href="https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html" target="_blank">Rotating Secrets in Secrets Manager</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)


- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
