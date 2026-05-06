---
layout: post
title: "Phase 5: Observability & Security Hardening — CloudWatch, Secrets Manager, and Structured Logging"
date: 2026-05-06T10:00:00
categories:
  - MindCraft Cloud Deployment
tags:
  - CICD
  - aws
  - pipeline
  - devsecops
  - monitoring
  - bcrypt
author: muhammed
description: Observability & Security Hardening
toc: true
pin: false
math: false
mermaid: false
image: /assets/thumbnails/finalphase.png
Link: "[[2026-05-06-cicd-prerequisites-walkthrough]]"
Link1:
---



## What This Phase Covers

  

By the end of Phase 3 , the pipeline was green, images were in ECR, and containers were running on EC2. But "running" is not the same as "observable." I had no idea what wa3s happening inside those containers unless I SSM'd in and ran `docker logs` manually.

  

Phase 5 fixes that — and adds the security pieces that turn the project from a demo into something you'd actually trust in production:

  

1. **CloudWatch agent** on every EC2 — container logs shipped automatically, memory and disk metrics collected

2. **Structured JSON logging** in the Express API via winston

3. **AWS Secrets Manager** — secrets never appear in SSM command history

4. **EBS snapshot lifecycle** — automated daily MongoDB backups

5. **npm audit gate** — blocks CRITICAL vulnerabilities in CI

6. **Dockerfile health check fix** — the frontend container was always "unhealthy"

  

---

  

## The Bug I Found: The Container That Lied

  

Before adding anything new, I discovered the frontend Dockerfile had a broken health check:

  

```dockerfile

# WRONG — Next.js has no /api/health route

HEALTHCHECK CMD wget -qO- http://localhost:3000/api/health || exit 1

```

  

Every time the container ran, Docker was marking it `(unhealthy)` within 30 seconds. The app was actually running fine — `Ready in 369ms` appeared in the logs — but Docker's health status said otherwise.

  

The fix is one word:

  

```dockerfile

# Correct — Next.js always returns 200 at /

HEALTHCHECK CMD wget -qO- http://localhost:3000/ || exit 1

```

  

The ALB health check (separate from Docker's) was already checking `/` and reporting `healthy`. So the ALB was routing traffic correctly, but `docker ps` would have shown `(unhealthy)` to anyone monitoring it. Fixed.

  

---

  

## CloudWatch: Seeing Inside the Instances

  

### The Problem With EC2 Defaults

  

CloudWatch gives you CPU utilization for EC2 by default. That's it. Memory and disk are not exposed — the hypervisor can't see inside the OS. And container logs don't flow anywhere automatically.

  

The solution is the CloudWatch agent: a small daemon that runs on the EC2 instance, reads from `/var/lib/docker/containers/**/*-json.log`, and ships those logs to CloudWatch Log Groups. It also collects memory and disk metrics.

  

### Installing the Agent via User Data

  

Every EC2 instance in this project uses `user_data` to bootstrap on first boot. I extended that script to install and configure the agent:

  

```bash

dnf install -y docker amazon-cloudwatch-agent

  

cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'CWEOF'

{

  "logs": {

    "logs_collected": {

      "files": {

        "collect_list": [{

          "file_path": "/var/lib/docker/containers/**/*-json.log",

          "log_group_name": "/mindcraft/api",

          "log_stream_name": "{instance_id}",

          "timezone": "UTC"

        }]

      }

    }

  },

  "metrics": {

    "append_dimensions": { "InstanceId": "${aws:InstanceId}" },

    "metrics_collected": {

      "cpu":  { "measurement": ["cpu_usage_idle", "cpu_usage_user"], "metrics_collection_interval": 60, "totalcpu": true },

      "disk": { "measurement": ["used_percent"], "metrics_collection_interval": 60, "resources": ["/"] },

      "mem":  { "measurement": ["mem_used_percent"], "metrics_collection_interval": 60 }

    }

  }

}

CWEOF

  

/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \

  -a fetch-config -m ec2 -s \

  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

```

  

One Terraform note: the agent config JSON contains `${aws:InstanceId}` — a CW agent template variable. In a Terraform heredoc, `${...}` is Terraform interpolation syntax. To prevent Terraform from trying to resolve it, escape the dollar sign: `$${aws:InstanceId}`. Terraform outputs `${aws:InstanceId}` literally in the generated shell script, and the CW agent resolves it at runtime.

  

### Three Separate user_data Scripts

  

Each EC2 tier ships logs to its own log group (`/mindcraft/web`, `/mindcraft/api`, `/mindcraft/db`). This means three separate `user_data` locals in `terraform/modules/ec2/main.tf` — identical Docker setup, different `log_group_name`.

  

### CloudWatch Terraform Module

  

Created `terraform/modules/cloudwatch/` with:

  

```hcl

# Log groups — 7-day retention to keep costs low

resource "aws_cloudwatch_log_group" "web" {

  name              = "/mindcraft/web"

  retention_in_days = 7

}

  

# SNS topic for alerts

resource "aws_sns_topic" "alerts" {

  name = "mindcraft-alerts"

}

  

resource "aws_sns_topic_subscription" "email" {

  topic_arn = aws_sns_topic.alerts.arn

  protocol  = "email"

  endpoint  = var.alert_email

}

  

# CPU alarm — web tier

resource "aws_cloudwatch_metric_alarm" "web_cpu" {

  alarm_name          = "mindcraft-web-cpu-high"

  comparison_operator = "GreaterThanThreshold"

  evaluation_periods  = 2

  metric_name         = "CPUUtilization"

  namespace           = "AWS/EC2"

  period              = 300

  statistic           = "Average"

  threshold           = 80

  alarm_actions       = [aws_sns_topic.alerts.arn]

  ok_actions          = [aws_sns_topic.alerts.arn]

  dimensions          = { InstanceId = var.web_instance_id }

}

```

  

Two CPU alarms (web + API). Threshold 80%, evaluation across 2 periods of 5 minutes — so you get alerted only after 10 sustained minutes of high CPU, not a brief spike.

  

### IAM: The Agent Already Had Permission

  

The EC2 IAM role already had `CloudWatchAgentServerPolicy` attached (Phase 2 setup). No new IAM changes needed for CloudWatch — it was pre-positioned.

  

---

  

## Structured Logging: Winston in Express

  

### Why Morgan Isn't Enough

  

Morgan is an HTTP request logger. It logs `GET /api/users 200 12ms` and that's it. It has no concept of severity levels, no JSON format, and no way to add structured fields to application logs.

  

In production, you want JSON so CloudWatch Logs Insights can query across fields:

  

```json

{ "level": "error", "method": "POST", "path": "/api/auth/login", "status": 401, "message": "Invalid password" }

```

  

### The Implementation

  

Added `winston` to `server/package.json`:

  

```json

"winston": "^3.17.0"

```

  

Created `server/logger.js`:

  

```javascript

import winston from 'winston';

  

const isProd = process.env.NODE_ENV === 'production';

  

export const logger = winston.createLogger({

  level: isProd ? 'http' : 'debug',

  format: isProd

    ? winston.format.json()

    : winston.format.combine(

        winston.format.colorize(),

        winston.format.simple()

      ),

  transports: [new winston.transports.Console()],

});

  

// Pipes morgan HTTP logs through winston in production

export const morganStream = {

  write: (message) => logger.http(message.trim()),

};

```

  

In production: JSON output, `http` level threshold (so HTTP access logs appear). In dev: colorized human-readable output, `debug` level. The same logger handles both environments.

  

In `server/index.js`:

```javascript

// Morgan output piped through winston in production

app.use(morgan(isProd ? 'combined' : 'dev', { stream: isProd ? morganStream : undefined }));

  

// Error handler uses logger instead of console.error

logger.error(`${req.method} ${req.path}`, { status: err.status, message: err.message });

  

// Startup

logger.info(`MindCraft API running on port ${PORT}`, { env: process.env.NODE_ENV });

```

  

One design decision: `level: 'http'` in production. Winston's default level ordering means that if you set `level: 'info'`, HTTP-level messages are silenced (http = priority 3, below info = priority 2). Setting `level: 'http'` ensures access logs flow while keeping verbose/debug silent.

  

---

  

## AWS Secrets Manager: Secrets Off the Command Line

  

### The Problem With Environment Variables in SSM Commands

  

The previous deploy.yml ran containers like this:

  

```bash

docker run -d --name mindcraft-api --env MONGODB_URI="mongodb://..." ...

```

  

That connection string lives in the SSM command history in plaintext. Anyone with `ssm:ListCommands` on the account can read it.

  

### The Better Pattern

  

Instead of passing secrets as GitHub Actions secrets into the SSM command directly, the EC2 instance fetches them from Secrets Manager at deploy time:

  

```bash

MONGODB_URI=$(aws secretsmanager get-secret-value \

  --secret-id mindcraft/mongodb-uri \

  --query SecretString --output text --region $region)

JWT_SECRET=$(aws secretsmanager get-secret-value \

  --secret-id mindcraft/jwt-secret \

  --query SecretString --output text --region $region)

GEMINI_API_KEY=$(aws secretsmanager get-secret-value \

  --secret-id mindcraft/gemini-api-key \

  --query SecretString --output text --region $region)

docker run -d --name mindcraft-api -p 3001:3001 --restart unless-stopped \

  --env NODE_ENV=production \

  --env MONGODB_URI="$MONGODB_URI" \

  --env JWT_SECRET="$JWT_SECRET" \

  --env GEMINI_API_KEY="$GEMINI_API_KEY" \

  $ecr/mindcraft-api:latest

```

  

The secrets are fetched on the EC2 instance using the instance's IAM role — no credentials required, and the secret values never appear in the SSM command document sent from GitHub Actions.

  

### Secrets Live Outside Terraform

  

An early version of this project managed the secret resources in Terraform. The problem: `terraform destroy` deleted the secrets, meaning secret values had to be re-entered after every apply/destroy cycle — even though the actual values (JWT key, Gemini API key) never change.

  

The fix: secrets are created once via CLI and never touched by Terraform again.

  

```bash

aws secretsmanager create-secret \

  --name mindcraft/jwt-secret \

  --secret-string "your-jwt-secret" \

  --region ap-southeast-1

  

aws secretsmanager create-secret \

  --name mindcraft/mongodb-uri \

  --secret-string "mongodb://10.0.5.10:27017/mindcraft" \

  --region ap-southeast-1

  

aws secretsmanager create-secret \

  --name mindcraft/gemini-api-key \

  --secret-string "your-gemini-key" \

  --region ap-southeast-1

```

  

The DB instance has a fixed private IP (`10.0.5.10`, pinned in the EC2 Terraform resource) so the MongoDB URI is always the same — no update needed after re-apply.

  

Terraform manages the IAM role that grants access. The secrets themselves persist across any number of destroy/apply cycles. This is the correct boundary: Terraform owns infrastructure, not application secrets.

  

### IAM Scope

  

The EC2 IAM role can only read secrets in the `mindcraft/*` namespace:

  

```hcl

resource "aws_iam_role_policy" "secrets_read" {

  policy = jsonencode({

    Statement = [{

      Effect   = "Allow"

      Action   = ["secretsmanager:GetSecretValue"]

      Resource = "arn:aws:secretsmanager:*:*:secret:mindcraft/*"

    }]

  })

}

```

  

### Frontend: Build-Time Variable

  

`NEXT_PUBLIC_API_URL` is different — it's baked into the JavaScript bundle at build time (Next.js's `NEXT_PUBLIC_` prefix convention). It can't be injected at `docker run` time.

  

Fixed in `Dockerfile.frontend`:

```dockerfile

ARG NEXT_PUBLIC_API_URL

ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL

RUN npm run build

```

  

And in `deploy.yml`:

```yaml

- name: Build frontend image

  run: |

    docker build -f Dockerfile.frontend \

      --build-arg NEXT_PUBLIC_API_URL=${{ secrets.NEXT_PUBLIC_API_URL }} \

      -t $ECR_REGISTRY/mindcraft-frontend:${{ github.sha }} \

      .

```

  

---

  

## EBS Snapshot Lifecycle: Automated MongoDB Backups

  

MongoDB data lives on a separate 20GB gp3 EBS volume (`/dev/sdf`). Without snapshots, a terminated instance means lost data.

  

AWS Data Lifecycle Manager (DLM) automates this:

  

```hcl

resource "aws_dlm_lifecycle_policy" "mongodb_snapshot" {

  description        = "Daily MongoDB EBS snapshot, 7-day retention"

  execution_role_arn = aws_iam_role.dlm.arn

  state              = "ENABLED"

  

  policy_details {

    resource_types = ["VOLUME"]

  

    schedule {

      name = "Daily"

      create_rule {

        interval      = 24

        interval_unit = "HOURS"

        times         = ["02:00"]

      }

      retain_rule { count = 7 }

      copy_tags = true

    }

  

    target_tags = {

      Name = "mindcraft-mongodb-data"

    }

  }

}

```

  

DLM matches volumes by the `Name` tag, takes a snapshot at 02:00 UTC daily, and deletes snapshots older than 7. Zero manual steps.

  

---

  

## npm Audit Gate in CI

  

Added a `npm-audit` job to `security.yml` that runs before Trivy:

  

```yaml

npm-audit:

  name: npm audit

  runs-on: ubuntu-latest

  steps:

    - uses: actions/checkout@v4

    - uses: actions/setup-node@v4

      with:

        node-version: "20"

        cache: "npm"

    - name: Audit frontend dependencies

      run: |

        npm ci

        npm audit --audit-level=critical

    - name: Audit API dependencies

      working-directory: server

      run: |

        npm install

        npm audit --audit-level=critical

```

  

`--audit-level=critical` — only fail on CRITICAL severity. This mirrors the Trivy policy (ignore-unfixed, CRITICAL only). HIGH vulnerabilities are surfaced in the audit output but don't block the build — transitive HIGH vulns in build tools are common and often unfixable.

  

---

  

## What's Left

  

- **HTTPS** — requires a registered domain and ACM certificate. Deferred: the ALB is ready (HTTP listener on 80), adding HTTPS is two Terraform resources once you have a domain.

- **MongoDB TLS** — certificate management between containers adds complexity. Deferred.

- **CloudWatch dashboard** — the log groups and metrics are flowing; a Terraform `aws_cloudwatch_dashboard` resource with JSON widget config is the remaining step.

  

---

  

## The New Terraform Module Count

  

Phase 2 provisioned 35 resources. After Phase 5:

- `terraform/modules/cloudwatch/` — 7 resources (3 log groups + SNS topic + subscription + 2 CPU alarms)

- `terraform/modules/secrets/` — 3 resources (3 Secrets Manager secrets)

- `terraform/modules/ec2/main.tf` additions — DLM role + policy attachment + lifecycle policy (3 resources)

  

Total: ~48 resources on next apply.

  

---

  

## What Actually Happened

  

The first time I ran `terraform plan` after adding the cloudwatch module, I hit a subtle Terraform escape issue. The CloudWatch agent config JSON contains `${aws:InstanceId}` — a CW agent template variable. Inside a Terraform heredoc, `${` triggers Terraform interpolation and the plan fails with:

  

```

Error: Invalid template control keyword

```

  

Fix: escape to `$${aws:InstanceId}` in the Terraform template. Terraform writes `${aws:InstanceId}` literally into the generated shell script; the CW agent resolves it at runtime. One character, completely non-obvious if you haven't seen it before.



---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.comMhdomer)  
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/) 
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
