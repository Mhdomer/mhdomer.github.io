---
layout: post
title: "Day 8: AWS Secrets Manager & Parameter Store - Securing Credentials"
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
description: A practical walkthrough of AWS Secrets Manager and SSM Parameter Store - how to store, rotate, and retrieve secrets securely without hardcoding credentials anywhere.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fdevio2023-media.developers.io%2Fwp-content%2Fuploads%2F2023%2F08%2Faws-systems-manager.png&f=1&nofb=1&ipt=a02cd36b5d7af05a4fd3e370e26ac7a893429252720ae25a097e21575a4adce1
---

## The #1 Cause of Cloud Breaches: Hardcoded Secrets

Every junior developer has done it or been tempted to do it:

```javascript
// database.js
const db = mysql.createConnection({
  host: "prod-db.c91823.ap-southeast-1.rds.amazonaws.com",
  user: "admin",
  password: "SuperSecretPassword2026!" // DO NOT DO THIS!
});
```

You tell yourself: *"I will just test it locally, and before I push to GitHub I will remove the password."*

Then at 11:30 PM, you run `git add . && git commit -m "fix db query" && git push origin main`.

Within five minutes, automated GitHub scrapers find your commit, extract the credentials, and attackers begin enumerating your database. Even if you make a new commit deleting the password, the credential remains baked into your repository's git commit history forever!

The fundamental rule of modern cloud development is simple: **Code must never contain credentials.** Secrets must live in a centralized, encrypted secrets store and be fetched dynamically at runtime.

In AWS, we have two primary tools for this: **AWS Secrets Manager** and **AWS Systems Manager Parameter Store**.

---

## Secrets Manager vs Parameter Store: The Big Confusion

When I started, I asked: *"Why does AWS have two different services that store encrypted strings? Why would I pay $0.40 a month for Secrets Manager when Parameter Store is free?"*

Here is the simple mental model:

- **SSM Parameter Store is an Encrypted Digital Notebook:** It stores configuration settings, environment variables, feature flags, license keys, and static API tokens. It is simple, fast, and completely free in the Standard tier.
- **AWS Secrets Manager is a High-Security Digital Safe with an Automated Combination Lock (Rotation):** It does everything Parameter Store does, but with a killer feature: **automatic secret rotation**. It integrates with AWS Lambda and RDS to automatically change database passwords every 30 days without human intervention or downtime.

```
+-------------------------------------------------------------+
|                      Secrets Architecture                   |
+-------------------------------------------------------------+
                 |                                  |
                 v                                  v
    [ SSM Parameter Store ]              [ AWS Secrets Manager ]
    - Config values & env vars           - High-value DB passwords
    - Static API keys                    - Automatic Lambda rotation
    - Free (Standard tier)               - ~$0.40 / secret / month
    - Max 4KB - 8KB                      - Max 65KB
```

---

## AWS Secrets Manager Deep Dive

### What It Stores

Secrets Manager stores JSON payloads up to 65KB, perfect for structured database credentials:

```json
{
  "engine": "postgres",
  "host": "production-db.c91823.ap-southeast-1.rds.amazonaws.com",
  "port": 5432,
  "username": "app_user",
  "password": "n8!vK9#mQ2$pL0@z"
}
```

### Hierarchical Secret Naming

Always structure secret names using hierarchical forward slashes:

```
prod/ecommerce/database
prod/ecommerce/stripe-api-key
staging/ecommerce/database
```

Using paths makes writing least-privilege IAM policies straightforward: you can grant a microservice access to `arn:aws:secretsmanager:*:*:secret:prod/ecommerce/*` without having to list every secret ARN manually.

---

## Retrieving Secrets at Runtime

### 1. From the AWS CLI

```bash
aws secretsmanager get-secret-value \
  --secret-id prod/ecommerce/database \
  --query SecretString \
  --output text
```

### 2. From Python (boto3)

Instead of reading from `.env` files on disk, your application queries Secrets Manager on startup:

```python
import boto3
import json
from botocore.exceptions import ClientError

def get_database_credentials():
    secret_name = "prod/ecommerce/database"
    region_name = "ap-southeast-1"

    session = boto3.session.Session()
    client = session.client(
        service_name="secretsmanager",
        region_name=region_name
    )

    try:
        get_secret_value_response = client.get_secret_value(
            SecretId=secret_name
        )
    except ClientError as e:
        # Handle access denied or missing secret errors cleanly
        raise e

    secret = json.loads(get_secret_value_response["SecretString"])
    return secret["username"], secret["password"]
```

### 3. Least-Privilege IAM Policy

Your ECS task execution role or EC2 instance profile only needs read access to that specific secret path:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowGetAppSecretsOnly",
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:prod/ecommerce/*"
    }
  ]
}
```

---

## How Automatic Secret Rotation Works

If an employee leaves your company or an API key is suspected of being exposed, manually rotating 20 database passwords across 10 microservices is a recipe for broken deployments.

AWS Secrets Manager automates this using an AWS Lambda function:

```
[ Secrets Manager Timer: Every 30 Days ]
                   |
                   v
         [ Trigger Lambda ]
                   |
    1. Create new password on database
    2. Test connection with new password
    3. Update secret in Secrets Manager
    4. Retire old password
```

1. **createSecret:** Lambda generates a new password.
2. **setSecret:** Lambda connects to the database as admin and updates the user's password.
3. **testSecret:** Lambda tests logging in with the new credentials.
4. **finishSecret:** Secrets Manager switches the current secret version to the new password.

When your application re-authenticates, it receives the updated credentials seamlessly.

---

## Direct Injection in ECS & AWS Lambda

You do not even need to write custom `boto3` retrieval code if you are deploying to ECS or Lambda. AWS can inject secrets directly into environment variables at container launch:

### ECS Task Definition Integration

```json
{
  "name": "ecommerce-api",
  "image": "123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/api:v1.2",
  "secrets": [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:prod/ecommerce/database-xYz123:password::"
    }
  ]
}
```

The ECS agent calls Secrets Manager on container boot and injects `DB_PASSWORD` into process memory. The secret is never baked into the Docker image layers!

---

## SSM Parameter Store: The Lightweight Alternative

For configuration data, environment variables, and static secrets where automated rotation is not needed, **Systems Manager Parameter Store** is the ideal tool.

### Parameter Types

| Type | How It Works | Best Use Case |
| :--- | :--- | :--- |
| `String` | Plaintext string | API endpoints, log levels, environment names |
| `StringList` | Comma-separated strings | Allowed CORS domains, subnet lists |
| `SecureString`| Encrypted at rest with AWS KMS | API tokens, private signing keys |

Always select `SecureString` for sensitive credentials!

### Retrieving Parameters via CLI

```bash
# Retrieve a single encrypted secret with automatic KMS decryption
aws ssm get-parameter \
  --name /prod/ecommerce/stripe-key \
  --with-decryption \
  --query Parameter.Value \
  --output text

# Fetch all configuration parameters for an app in one recursive call
aws ssm get-parameters-by-path \
  --path /prod/ecommerce/ \
  --with-decryption \
  --recursive
```

---

## Junior Pitfalls to Avoid

1. **The In-Memory Caching Bug:** If your application queries Secrets Manager once at boot and caches the database password in memory forever, what happens when Secrets Manager automatically rotates the password at 3:00 AM? Your app will keep using the old password, fail authentication, and crash! Always implement retry logic: if a DB connection fails with authentication errors, fetch the fresh secret from Secrets Manager before failing.
2. **Using Secrets Manager for Every Single Setting:** At $0.40 per secret per month, storing 500 minor config settings in Secrets Manager costs $200/month. Use Parameter Store (free) for general configs, and reserve Secrets Manager for credentials requiring automated rotation.
3. **Leaving Hardcoded Secrets in Git History:** If you accidentally commit a secret, changing the code in a new commit does NOT delete it. You must revoke the secret immediately in AWS, and purge git history using `git-filter-repo` or BFG Repo-Cleaner.
4. **Granting Wildcard Permissions:** Never write `"Resource": "arn:aws:secretsmanager:*:*:secret:*"` in an IAM policy. Always restrict secret access to specific application path prefixes.

---

## Key Takeaways

- Never hardcode passwords or API keys in source code, Dockerfiles, or git repositories.
- Use AWS Secrets Manager when you need automated secret rotation (especially for RDS databases).
- Use SSM Parameter Store with `SecureString` for static configuration values and general API keys.
- Inject secrets directly into container memory via ECS task definition `secrets` blocks to avoid storing secrets in image layers.
- Always implement re-fetch logic in applications to handle seamless password rotation without restarts.

---

## References

<div class="references">
<ul>
  <li><a href="https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html" target="_blank">AWS Secrets Manager User Guide</a></li>
  <li><a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html" target="_blank">AWS Systems Manager Parameter Store Guide</a></li>
  <li><a href="https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html" target="_blank">Rotating AWS Secrets Manager Secrets</a></li>
  <li><a href="https://aws.amazon.com/blogs/compute/specifying-sensitive-data-in-aws-lambda-using-aws-systems-manager-parameter-store-and-aws-secrets-manager/" target="_blank">AWS Compute Blog: Injecting Secrets into Lambda</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
