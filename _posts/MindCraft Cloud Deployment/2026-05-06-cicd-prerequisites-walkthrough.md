---
layout: post
title: "Phase 4: CI/CD Pipeline with GitHub Actions — Complete Walkthrough"
date: 2026-05-06T10:00:00
categories:
  - MindCraft Cloud Deployment
tags:
  - CICD
  - aws
  - pipeline
  - devsecops
author: muhammed
description: CI/CD Pipeline with GitHub Actions
toc: true
pin: false
math: false
mermaid: false
image: /assets/thumbnails/phase4.png
Link: "[[2026-03-24-Introduction-about-Application]]"
Link1:
---

---

  

Phase 3 is the CI/CD pipeline. This post covers the complete picture — the

infrastructure setup that makes the pipeline possible, and the three workflow files that

run on every push.

  

This is Phase 3. Not a separate phase, not a "nice to have" — the prerequisites and

the workflows together are what makes Phase 3 complete. The Terraform code (Phase 2)

provisions empty EC2 instances. Phase 3 is the system that gets the application onto

those instances automatically on every `git push`.

  

Before writing a single line of GitHub Actions YAML, three things need to exist in AWS

and GitHub. Most tutorials skip this part — they hand you admin credentials and call it

a day. That's the wrong way to do it, and this post explains why and what to do instead.

  

The three prerequisites:

  

1. A dedicated IAM user for CI/CD — scoped to exactly the permissions the pipeline needs

2. Two ECR repositories — where the Docker images will live

3. GitHub Secrets — where the credentials are stored so workflows can use them

  

Everything done here from the terminal. Zero console clicking.

  

---

  

## Why a Dedicated CI IAM User

  

Your personal AWS credentials have broad permissions. You created the VPC, the EC2

instances, the S3 bucket — you need wide access. A GitHub Actions pipeline does not.

  

If you put your personal access key in GitHub Secrets, you have created a credential

that:

  

- Has far more permissions than the pipeline needs

- Is stored in a system you do not fully control (GitHub's secret storage)

- If leaked, can do anything your account can do — including deleting your S3 state

  bucket, terminating all your instances, or racking up charges

  

The fix is least privilege: create a separate IAM user with only the permissions the

pipeline actually needs, and nothing else. If that credential leaks, the blast radius

is contained.

  

---

  

## Step 1 — Create the IAM User

  

```bash

aws iam create-user --user-name mindcraft-ci

```

  

Output:

```json

{

    "User": {

        "UserName": "mindcraft-ci",

        "UserId": "AIDAUYNSCF4JJHELQ54WC",

        "Arn": "arn:aws:iam::327327821586:user/mindcraft-ci"

    }

}

```

  

---

  

## Step 2 — Attach a Scoped Policy

  

The policy below gives `mindcraft-ci` exactly four capabilities and nothing else:

  

```json

{

  "Version": "2012-10-17",

  "Statement": [

    {

      "Effect": "Allow",

      "Action": ["ecr:GetAuthorizationToken"],

      "Resource": "*"

    },

    {

      "Effect": "Allow",

      "Action": [

        "ecr:BatchCheckLayerAvailability",

        "ecr:GetDownloadUrlForLayer",

        "ecr:BatchGetImage",

        "ecr:InitiateLayerUpload",

        "ecr:UploadLayerPart",

        "ecr:CompleteLayerUpload",

        "ecr:PutImage",

        "ecr:DescribeRepositories"

      ],

      "Resource": "arn:aws:ecr:ap-southeast-1:327327821586:repository/mindcraft-*"

    },

    {

      "Effect": "Allow",

      "Action": [

        "ssm:SendCommand",

        "ssm:GetCommandInvocation",

        "ssm:DescribeInstanceInformation"

      ],

      "Resource": "*"

    },

    {

      "Effect": "Allow",

      "Action": ["ec2:DescribeInstances"],

      "Resource": "*"

    },

    {

      "Effect": "Allow",

      "Action": ["s3:GetObject", "s3:PutObject"],

      "Resource": "arn:aws:s3:::mindcraft-tfstate-327327821586/*"

    }

  ]

}

```

  

**Breaking down what each statement allows and why:**

  

**`ecr:GetAuthorizationToken` on `*`** — this is how Docker authenticates to ECR. It

exchanges your AWS credentials for a short-lived Docker login token. The `*` resource

is required here — this action does not support resource-level restrictions. It only

returns a token; it cannot create, delete, or modify anything.

  

**ECR image actions on `mindcraft-*` repositories only** — these are the permissions to

push Docker image layers. The resource ARN is scoped to `repository/mindcraft-*` — this

user cannot push to any other ECR repository in the account, even if one exists.

  

**SSM `SendCommand` + `GetCommandInvocation`** — this is how the deploy step runs

`docker pull && docker restart` on the EC2 instances without SSH. `SendCommand` sends

the script; `GetCommandInvocation` polls the result. No port 22 involved.

  

**`ec2:DescribeInstances`** — the deploy workflow uses tags to find instance IDs

dynamically (they change every `terraform apply`). This read-only action is what lets

the pipeline ask "which instance has tag Project=mindcraft and Tier=web?"

  

**S3 read/write on the tfstate bucket only** — the CI pipeline may need to read

Terraform state to get output values. Scoped to the specific bucket, not all S3.

  

```bash

aws iam put-user-policy \

  --user-name mindcraft-ci \

  --policy-name mindcraft-ci-policy \

  --policy-document file://mindcraft-ci-policy.json

```

  

---

  

## Step 3 — Generate Access Keys

  

```bash

aws iam create-access-key --user-name mindcraft-ci

```

  

Output:

```json

{

    "AccessKey": {

        "UserName": "mindcraft-ci",

        "AccessKeyId": "AKIAUYNSCF4JHNWPH64Y",

        "Status": "Active",

        "SecretAccessKey": "..."

    }

}

```

  

The `SecretAccessKey` is only shown once. It goes straight into GitHub Secrets —

never into a file, never into the terminal history, never into the repo.

  

---

  

## Step 4 — Create the ECR Repositories

  

ECR (Elastic Container Registry) is AWS's Docker registry — the same concept as Docker

Hub, but private and running inside your AWS account. The pipeline will build the

Docker images locally in the GitHub Actions runner, then push them here. The EC2

instances will pull from here at deploy time.

  

Two repositories — one per service:

  

```bash

aws ecr create-repository \

  --repository-name mindcraft-frontend \

  --region ap-southeast-1

  

aws ecr create-repository \

  --repository-name mindcraft-api \

  --region ap-southeast-1

```

  

The repository URIs:

  

```

327327821586.dkr.ecr.ap-southeast-1.amazonaws.com/mindcraft-frontend

327327821586.dkr.ecr.ap-southeast-1.amazonaws.com/mindcraft-api

```

  

The format is always: `<account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>`

  

Why two repositories and not one? Each service has its own build cycle, its own

image size, its own tags. Keeping them separate means you can deploy the API without

rebuilding the frontend, and vice versa. The Trivy security scan also runs per image —

a critical CVE in the frontend image should not block an API deploy.

  

---

  

## Step 5 — Store Credentials in GitHub Secrets

  

GitHub Secrets are encrypted values stored at the repository level. Workflows can

reference them as `${{ secrets.SECRET_NAME }}` — they are injected into the runner

environment at job start and are never visible in logs.

  

Installing the GitHub CLI:

  

```bash

winget install GitHub.cli

gh auth login   # authenticate once via browser

```

  

Setting all four secrets in one go:

  

```bash

gh secret set AWS_ACCESS_KEY_ID     --body "AKIAUYNSCF4JHNWPH64Y"

gh secret set AWS_SECRET_ACCESS_KEY --body "<secret-key>"

gh secret set AWS_REGION            --body "ap-southeast-1"

gh secret set AWS_ACCOUNT_ID        --body "327327821586"

```

  

Why `AWS_ACCOUNT_ID` as a separate secret? The ECR registry URI is

`<account-id>.dkr.ecr.<region>.amazonaws.com`. Rather than hardcoding the account ID

in the workflow YAML (which would be committed to the repo), we reference it as a

secret so the workflow file contains no account-specific values.

  

Verification:

  

```bash

gh secret list

```

  

```

AWS_ACCESS_KEY_ID      2026-04-30T11:22:26Z

AWS_ACCOUNT_ID         2026-04-30T11:22:28Z

AWS_REGION             2026-04-30T11:22:27Z

AWS_SECRET_ACCESS_KEY  2026-04-30T11:22:26Z

```

  

---

  

## One More Thing — EC2 Instances Need ECR Read Access

  

The deploy job tells the EC2 instance to run `docker pull` from ECR. The instance

authenticates using its IAM Instance Profile — not the `mindcraft-ci` credentials.

  

The original Terraform EC2 module attached two policies to the instance role:

- `AmazonSSMManagedInstanceCore` — for SSM Session Manager access

- `CloudWatchAgentServerPolicy` — for sending logs to CloudWatch

  

It was missing the third: `AmazonEC2ContainerRegistryReadOnly` — without it, `docker pull`

from ECR fails with an authorization error. One line added to the EC2 module:

  

```hcl

resource "aws_iam_role_policy_attachment" "ecr_read" {

  role       = aws_iam_role.ec2.name

  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"

}

```

  

This is read-only — the EC2 instances can pull images but cannot push. Only the

`mindcraft-ci` user (running in GitHub Actions) has push permissions.

  

---

  

## What the Pipeline Can Now Do

  

With these four secrets in place, a GitHub Actions workflow can:

  

1. **Authenticate to AWS** — using `aws-actions/configure-aws-credentials@v4` with the

   `mindcraft-ci` key pair

2. **Log in to ECR** — using `aws-actions/amazon-ecr-login@v2`, which calls

   `ecr:GetAuthorizationToken` and configures Docker

3. **Push images to ECR** — `docker push` to either `mindcraft-frontend` or

   `mindcraft-api`

4. **Deploy via SSM** — `aws ssm send-command` to run `docker pull` on the EC2

   instances, discovered by tag

  

It cannot: create or delete infrastructure, access other S3 buckets, terminate

instances, or do anything outside those five policy statements.

  

---

  

## The Three Workflow Files

  

Three files live in `.github/workflows/`. Each has a different trigger and a different

purpose.

  

---

  

### `ci.yml` — Build Check on Every Push

  

```yaml

on:

  push:

    branches: ["**"]

  pull_request:

    branches: [main]

```

  

Runs on every push to every branch. Installs dependencies and runs `next build`. If

the build fails, the push is marked red — fast feedback before anything reaches `main`.

  

This is the first gate. It doesn't touch AWS, doesn't build Docker images. Just: does

the code compile?

  

---

  

### `security.yml` — Trivy Scan on PRs and Main

  

```yaml

on:

  pull_request:

    branches: [main]

  push:

    branches: [main]

```

  

Builds both Docker images locally in the GitHub Actions runner and runs Trivy against

them. Any CRITICAL CVE in either image fails the workflow with exit code 1 — the PR

cannot merge, the deployment cannot proceed.

  

```yaml

- name: Scan frontend — fail on CRITICAL

  uses: aquasecurity/trivy-action@v0.36.0

  with:

    image-ref: mindcraft-frontend:scan

    exit-code: "1"

    ignore-unfixed: true

    severity: CRITICAL

```

  

`ignore-unfixed: true` means CVEs with no available fix are skipped — flagging

unfixable vulnerabilities in CI just creates noise. Only actionable findings block

the build.

  

---

  

### `deploy.yml` — Scan, Push, Deploy on Merge to Main

  

This is the main event. It runs on every push to `main` (i.e., every merged PR) and

has two jobs:

  

**Job 1: `scan-and-push`**

  

1. Authenticates to AWS using the `mindcraft-ci` credentials from GitHub Secrets

2. Logs in to ECR via `aws-actions/amazon-ecr-login@v2`

3. Builds the frontend Docker image, tags it with the git commit SHA and `latest`

4. Runs Trivy against the built image — fails on CRITICAL before anything is pushed

5. Pushes both tags to ECR only if the scan passed

6. Repeats for the API image

  

Tagging with `github.sha` means every image in ECR is traceable to the exact commit

that produced it. Rolling back means pulling a specific SHA tag.

  

**Job 2: `deploy`** (runs after `scan-and-push` succeeds)

  

Discovers instances by tag — not hardcoded IDs:

  

```bash

aws ec2 describe-instances \

  --filters \

    "Name=tag:Project,Values=mindcraft" \

    "Name=tag:Tier,Values=web" \

    "Name=instance-state-name,Values=running" \

  --query "Reservations[0].Instances[0].InstanceId" \

  --output text

```

  

Because `terraform destroy` removes all instances and `terraform apply` creates new

ones with different IDs, hardcoding instance IDs in the workflow would break every

time. Tag-based discovery means the pipeline works regardless of when the

infrastructure was last provisioned.

  

Then, for each instance, it sends an SSM command:

  

```bash

# Construct commands as JSON using jq (avoids shell escaping issues)

COMMANDS=$(jq -n \

  --arg ecr "$ECR_REGISTRY" \

  --arg region "$AWS_REGION" \

  '[

    ("aws ecr get-login-password --region " + $region + " | docker login --username AWS --password-stdin " + $ecr),

    ("docker pull " + $ecr + "/mindcraft-frontend:latest"),

    "docker stop mindcraft-frontend 2>/dev/null || true",

    "docker rm   mindcraft-frontend 2>/dev/null || true",

    ("docker run -d --name mindcraft-frontend -p 3000:3000 --restart unless-stopped " + $ecr + "/mindcraft-frontend:latest")

  ]')

  

CMD_ID=$(aws ssm send-command \

  --document-name "AWS-RunShellScript" \

  --instance-ids "$WEB_ID" \

  --parameters "commands=$COMMANDS" \

  --query "Command.CommandId" --output text)

```

  

SSM runs the script on the EC2 instance. The pipeline polls every 10 seconds for up

to 4 minutes until it gets `Success` or `Failed`. No SSH. No open port 22.

  

**Graceful skip when infrastructure is down:**

  

```yaml

- name: Skip deploy — no running instances

  if: steps.web.outputs.id == 'None' || steps.web.outputs.id == ''

  run: |

    echo "No running instances found. Run terraform apply first."

```

  

If `terraform destroy` was run, the discovery step returns `None`. The deploy step

is skipped cleanly instead of failing — useful for the apply/destroy portfolio pattern

where infrastructure only runs during demo sessions.

  

---

  

## What Actually Happened — First Pipeline Run

  

The workflows were committed and pushed to main. GitHub Actions triggered immediately.

Here's exactly what happened.

  

### Bug: Trivy Action Version Did Not Exist

  

The first run failed before any scanning happened:

  

```

Error: Unable to resolve action `aquasecurity/trivy-action@0.28.0`,

unable to find version `0.28.0`

```

  

The version `0.28.0` was specified in both `security.yml` and `deploy.yml` — it does

not exist. The correct latest release is `v0.36.0` (note the `v` prefix, which the

action requires). Fixed in both files and pushed. The lesson: always verify action

versions against the actual GitHub releases page before committing.

  

### Second Run — All Green

  

After the version fix, all workflows completed successfully:

  

**`ci.yml`** — `npm ci` + `next build` passed. Build time: ~2 minutes.

  

**`security.yml`** — Built both Docker images in the runner, ran Trivy against each.

Zero CRITICAL CVEs in either image. Both scans passed.

  

**`deploy.yml` — `scan-and-push` job** — Built both images again (runners are

stateless — each job starts fresh), ran Trivy, then pushed to ECR:

  

```

327327821586.dkr.ecr.ap-southeast-1.amazonaws.com/mindcraft-frontend:cc3e2ea

327327821586.dkr.ecr.ap-southeast-1.amazonaws.com/mindcraft-frontend:latest

327327821586.dkr.ecr.ap-southeast-1.amazonaws.com/mindcraft-api:cc3e2ea

327327821586.dkr.ecr.ap-southeast-1.amazonaws.com/mindcraft-api:latest

```

  

Images are tagged with the git commit SHA (`cc3e2ea`) and `latest`. Every image in

ECR is traceable to the exact commit that produced it.

  

**`deploy.yml` — `deploy` job** — Ran the instance discovery step:

  

```bash

aws ec2 describe-instances \

  --filters "Name=tag:Project,Values=mindcraft" \

            "Name=tag:Tier,Values=web" \

            "Name=instance-state-name,Values=running" \

  --query "Reservations[0].Instances[0].InstanceId" \

  --output text

# → None

```

  

No running instances — `terraform destroy` was run after the last test session.

The deploy step hit the graceful skip condition:

  

```yaml

- name: Skip deploy — no running instances

  if: steps.web.outputs.id == 'None' || steps.web.outputs.id == ''

  run: |

    echo "No running instances found. Run terraform apply first."

```

  

The job completed green. This is correct behaviour — the pipeline should not fail

just because infrastructure is currently down. When `terraform apply` runs next, the

same workflow will find the instances by tag and deploy automatically.

  

### What Is Live Right Now

  

- Both Docker images are in ECR, tagged with the commit SHA and `latest`

- The pipeline is proven: build → scan → push works end-to-end

- The deploy path works: instance discovery by tag, SSM command construction, graceful

  skip — all verified logic

- The only missing piece is live EC2 instances to receive the deploy

  

The full end-to-end deploy (containers running on EC2, ALB serving traffic) happens

the next time `terraform apply` provisions the infrastructure and a push to main

triggers the deploy job.

  

---

  

## Phase 3 Complete

  

With the three workflow files committed, Phase 3 is done:

  

```

Push to main

    ├── ci.yml        — build check (all branches)

    ├── security.yml  — Trivy scan (PRs + main)

    └── deploy.yml

          ├── scan-and-push — Trivy → ECR push (images tagged with git SHA)

          └── deploy        — tag discovery → SSM → docker pull + restart

```

  

The pipeline badge in the README goes green. Deploying the application is `git push`.

The EC2 instances pull from ECR using their instance role — no credentials stored on

the servers.

  

Phase 4 is observability and security hardening: CloudWatch dashboards, structured

logging, secrets in AWS Secrets Manager instead of environment variables, and HTTPS

end-to-end.

  

Source: [github.com/Mhdomer/mindcraft-aws-migration](https://github.com/Mhdomer/mindcraft-aws-migration)




---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.comMhdomer)  
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/) 
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
