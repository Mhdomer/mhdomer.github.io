---
layout: post
title: YAML & GitHub Actions Walkthrough
date: 2026-05-08T13:00:00
categories:
  - General Notes & Materials
tags:
  - yaml
  - github-actions
  - cicd
  - devops
  - automation
author: muhammed
description: Full YAML syntax guide and GitHub Actions walkthrough — triggers, jobs, steps, secrets, matrix builds, reusable workflows, and real-world CI/CD pipelines
excerpt: Full YAML syntax guide and GitHub Actions walkthrough — triggers, jobs, steps, secrets, matrix builds, reusable workflows, and real-world CI/CD pipelines
toc: true
pin: false
math: false
mermaid: false
image: /assets/notes-img/github.png
Link: "[[]]"
Link1:
img:
---
{% raw %}
# YAML & GitHub Actions Walkthrough

---

## Part 1 — YAML

YAML (YAML Ain't Markup Language) is a human-readable data serialisation format. It's the config language for GitHub Actions, Kubernetes, Docker Compose, Ansible, and more. Getting it wrong silently breaks things, so understanding it properly matters.

---

### Basic Rules

- **Indentation is structure** — use spaces only, never tabs
- **2 spaces** per indent level is the convention (4 also works, just be consistent)
- **Case sensitive** — `Name` and `name` are different keys
- **`#`** starts a comment
- A YAML file can contain multiple documents separated by `---`

---

### Scalars — Strings, Numbers, Booleans, Null

```yaml
# Strings — quotes are optional unless the value contains special characters
name: Mohamed
greeting: "Hello, World!"
path: 'C:\Users\user'          # single quotes: no escape sequences
multiword: this is fine too    # no quotes needed for plain text

# Avoid ambiguity — quote these
version: "1.0"                 # unquoted 1.0 is a float
yes_string: "yes"              # unquoted yes/no/true/false = boolean
port: "8080"                   # unquoted integers stay integers — pick a side

# Numbers
count: 42
price: 3.14
hex: 0xFF
scientific: 1.2e10

# Booleans
debug: true
enabled: false
# These also parse as booleans in YAML 1.1 (avoid using them as strings):
# yes, no, on, off, True, False, TRUE, FALSE

# Null
value: null
empty:              # also null
explicit: ~         # also null
```

---

### Strings — Multiline

```yaml
# Literal block scalar |  — preserves newlines
script: |
  #!/bin/bash
  echo "Hello"
  echo "World"
# Result: "#!/bin/bash\necho \"Hello\"\necho \"World\"\n"

# Folded block scalar >  — newlines become spaces (good for long sentences)
description: >
  This is a very long description
  that spans multiple lines but
  will be joined into one paragraph.
# Result: "This is a very long description that spans multiple lines but will be joined into one paragraph.\n"

# Chomp modifiers
literal_strip: |-        # | strip trailing newline
  line one
  line two

literal_keep: |+         # | keep all trailing newlines
  line one
  line two


# Inline — escape sequences work in double-quoted strings
escaped: "first line\nsecond line\ttabbed"
```

---

### Collections — Lists and Maps

```yaml
# List (sequence) — block style
fruits:
  - apple
  - banana
  - cherry

# List — flow style (inline)
fruits: [apple, banana, cherry]

# Map (mapping) — block style
person:
  name: Mohamed
  age: 22
  city: Dubai

# Map — flow style (inline)
person: {name: Mohamed, age: 22}

# List of maps
users:
  - name: Alice
    role: admin
  - name: Bob
    role: developer

# Nested maps
database:
  primary:
    host: db-primary.internal
    port: 5432
  replica:
    host: db-replica.internal
    port: 5432

# Mixed nesting
config:
  servers:
    - name: web-1
      ip: 10.0.0.1
      tags: [web, nginx]
    - name: db-1
      ip: 10.0.0.2
      tags: [db, postgres]
```

---

### Anchors and Aliases — DRY in YAML

Anchors (`&`) define a reusable block. Aliases (`*`) reference it. `<<` merges a map.

```yaml
# Define anchor
defaults: &defaults
  timeout: 30
  retries: 3
  log_level: info

# Merge anchor into another map
development:
  <<: *defaults
  debug: true

production:
  <<: *defaults
  log_level: warn

# development becomes:
# timeout: 30
# retries: 3
# log_level: info
# debug: true
```

GitHub Actions uses anchors extensively to avoid repeating `env:` blocks.

---

### Common Gotchas

```yaml
# Colon in a value — must quote
url: "https://example.com"     # unquoted colon breaks parsing
message: "Error: not found"    # same issue

# Value starting with { or [ — must quote
json: '{"key": "value"}'
list: '[1, 2, 3]'

# Indentation after a block scalar
key: |
  line one
next_key: value    # this must be at the same indent as 'key'

# Empty string vs null
empty_string: ""   # empty string
null_value:        # null (no value at all)

# Octal integers in YAML 1.1
mode: 0755         # parsed as octal 493 in old parsers — quote it: "0755"
```

---

## Part 2 — GitHub Actions

GitHub Actions is a CI/CD platform built into GitHub. Workflows are YAML files stored in `.github/workflows/`.

---

## Anatomy of a Workflow

```
.github/
└── workflows/
    ├── ci.yml          # runs on every push/PR
    ├── deploy.yml      # runs on merge to main
    └── nightly.yml     # runs on a schedule
```

**Top-level structure:**

```yaml
name: CI Pipeline          # shown in GitHub UI
run-name: "Build ${{ github.sha }}"   # optional — shown per-run

on: ...                    # when to trigger

env:                       # workflow-level env vars
  NODE_VERSION: "20"

jobs:
  job-name:                # one or more jobs
    runs-on: ubuntu-latest
    steps:
      - name: Step name
        run: echo "hello"
```

---

## Triggers (`on:`)

### Push and pull_request

```yaml
on:
  push:
    branches:
      - main
      - "release/**"      # glob patterns
    paths:
      - "src/**"          # only trigger if these paths changed
      - "!docs/**"        # ! = exclude
    tags:
      - "v*"              # trigger on version tags

  pull_request:
    branches:
      - main
    types:                # default: opened, synchronize, reopened
      - opened
      - synchronize
      - reopened
      - ready_for_review
```

### Manual trigger

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        required: true
        default: "staging"
        type: choice
        options: [staging, production]
      dry_run:
        description: "Dry run?"
        type: boolean
        default: false
```

Run it from: **GitHub repo → Actions → select workflow → Run workflow**

### Scheduled trigger

```yaml
on:
  schedule:
    - cron: "0 2 * * *"       # daily at 2 AM UTC
    - cron: "0 9 * * MON"     # every Monday at 9 AM
```

### Other triggers

```yaml
on:
  release:
    types: [published]        # when a GitHub release is published

  issues:
    types: [opened, labeled]  # on issue events

  workflow_call:              # called from another workflow (reusable)

  repository_dispatch:        # external HTTP trigger via API
    types: [deploy]
```

---

## Jobs

```yaml
jobs:
  build:
    name: Build and Test         # display name
    runs-on: ubuntu-latest       # runner

    # Run only on main branch
    if: github.ref == 'refs/heads/main'

    # Timeout (default 360 min)
    timeout-minutes: 30

    # Environment (for secrets and protection rules)
    environment: production

    # Outputs — pass data to dependent jobs
    outputs:
      version: ${{ steps.get-version.outputs.version }}

    steps:
      - ...
```

### Runner options

```yaml
runs-on: ubuntu-latest      # Linux (most common)
runs-on: ubuntu-22.04       # specific version
runs-on: windows-latest     # Windows
runs-on: macos-latest       # macOS
runs-on: self-hosted        # your own runner
runs-on: [self-hosted, linux, gpu]   # labeled self-hosted
```

### Job dependencies

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "running tests"

  build:
    needs: test                  # waits for test to pass
    runs-on: ubuntu-latest
    steps:
      - run: echo "building"

  deploy:
    needs: [test, build]         # waits for both
    runs-on: ubuntu-latest
    steps:
      - run: echo "deploying"
```

---

## Steps

Each step is either a shell command (`run`) or an action (`uses`).

```yaml
steps:
  # Shell command
  - name: Run tests
    run: npm test

  # Multiline shell command
  - name: Build and lint
    run: |
      npm ci
      npm run lint
      npm run build

  # Use an action from the marketplace
  - name: Checkout code
    uses: actions/checkout@v4

  # Action with inputs
  - name: Setup Node.js
    uses: actions/setup-node@v4
    with:
      node-version: "20"
      cache: "npm"

  # Step-level env vars
  - name: Deploy
    run: ./deploy.sh
    env:
      API_KEY: ${{ secrets.API_KEY }}
      ENV: production

  # Conditional step
  - name: Notify on failure
    if: failure()
    run: curl -X POST ${{ secrets.SLACK_WEBHOOK }} -d '{"text":"Build failed"}'

  # Continue even if step fails
  - name: Run optional check
    run: ./optional-check.sh
    continue-on-error: true

  # Set output for use in later steps
  - name: Get version
    id: get-version
    run: echo "version=$(cat VERSION)" >> $GITHUB_OUTPUT

  - name: Use version
    run: echo "Deploying version ${{ steps.get-version.outputs.version }}"
```

---

## Contexts and Expressions

### Syntax

```yaml
${{ expression }}
```

### Key contexts

```yaml
# github context — event info
${{ github.sha }}               # commit SHA
${{ github.ref }}               # refs/heads/main
${{ github.ref_name }}          # main
${{ github.event_name }}        # push, pull_request, etc.
${{ github.actor }}             # username who triggered
${{ github.repository }}        # owner/repo
${{ github.run_id }}            # unique run ID
${{ github.run_number }}        # incrementing run number

# env context
${{ env.NODE_VERSION }}

# secrets context
${{ secrets.MY_SECRET }}

# steps context — outputs from earlier steps
${{ steps.step-id.outputs.value }}
${{ steps.step-id.outcome }}    # success, failure, skipped

# jobs context (in later jobs)
${{ needs.build.outputs.version }}
${{ needs.build.result }}       # success, failure, skipped, cancelled

# runner context
${{ runner.os }}                # Linux, Windows, macOS
```

### Expressions

```yaml
# Conditionals
if: github.ref == 'refs/heads/main'
if: github.event_name == 'pull_request'
if: contains(github.ref, 'release')
if: startsWith(github.ref, 'refs/tags/v')
if: failure()                   # step/job failed
if: always()                    # run regardless of outcome
if: cancelled()
if: success()                   # default

# Functions
contains(github.ref, 'main')
startsWith(github.ref, 'refs/tags')
endsWith(github.actor, 'bot')
format('Hello {0}', github.actor)
join(matrix.os, ',')
toJSON(github.event)
fromJSON('{"key":"value"}')
```

---

## Secrets and Variables

### Setting secrets

Go to: **Repo → Settings → Secrets and variables → Actions → New repository secret**

Levels:
- **Repository secrets** — available to that repo only
- **Environment secrets** — only available when job targets that environment
- **Organisation secrets** — shared across repos

### Using secrets

```yaml
steps:
  - name: Deploy
    env:
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    run: ./deploy.sh
```

Secrets are **masked** in logs — GitHub replaces the value with `***`.

### Variables (non-secret config)

```yaml
# Repo → Settings → Secrets and variables → Variables
${{ vars.APP_ENV }}
${{ vars.REGISTRY_URL }}
```

---

## Matrix Builds

Run the same job across multiple configurations in parallel.

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: ["18", "20", "22"]
      fail-fast: false       # don't cancel all if one fails
      max-parallel: 4        # max concurrent jobs

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
```

This creates 3 × 3 = **9 jobs** running in parallel.

### Include and exclude

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: ["18", "20"]
    include:
      - os: ubuntu-latest
        node: "22"         # add an extra combination
        experimental: true
    exclude:
      - os: windows-latest
        node: "18"         # skip this specific combination
```

---

## Caching

Speed up workflows by caching dependencies between runs.

```yaml
- name: Cache node_modules
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

- name: Install dependencies
  run: npm ci
```

The cache key includes a hash of `package-lock.json` — a new key is created when dependencies change.

**`setup-*` actions often have built-in caching:**

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: "20"
    cache: "npm"           # handles caching automatically
```

---

## Artifacts

Artifacts persist files between jobs or let you download build outputs.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 7

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/
      - run: ./deploy.sh dist/
```

---

## Environments and Deployment Protection

Environments add protection rules (required reviewers, wait timers) before deployment.

```yaml
jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com       # shown as link in GitHub UI
    steps:
      - run: ./deploy.sh
```

Go to **Repo → Settings → Environments** to configure:
- Required reviewers (must approve before job runs)
- Wait timer
- Deployment branch rules (only deploy from `main`)

---

## Reusable Workflows

Call one workflow from another — like a function.

```yaml
# .github/workflows/deploy.yml — the reusable workflow
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      deploy_key:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh ${{ inputs.environment }}
        env:
          DEPLOY_KEY: ${{ secrets.deploy_key }}
```

```yaml
# .github/workflows/ci.yml — calls the reusable workflow
jobs:
  deploy-staging:
    uses: ./.github/workflows/deploy.yml
    with:
      environment: staging
    secrets:
      deploy_key: ${{ secrets.STAGING_KEY }}

  deploy-prod:
    needs: deploy-staging
    uses: ./.github/workflows/deploy.yml
    with:
      environment: production
    secrets:
      deploy_key: ${{ secrets.PROD_KEY }}
```

---

## Composite Actions

Package multiple steps into a reusable action stored in your repo.

```yaml
# .github/actions/setup-app/action.yml
name: Setup App
description: Install dependencies and configure environment
inputs:
  node-version:
    description: Node.js version
    default: "20"
runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: npm
    - run: npm ci
      shell: bash
    - run: cp .env.example .env
      shell: bash
```

```yaml
# Use it in any workflow
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-app
    with:
      node-version: "20"
  - run: npm test
```

---

## Real-World CI/CD Pipeline

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ── 1. Lint and Test ──────────────────────────────────────────
  test:
    name: Lint & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: npm

      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  # ── 2. Build Docker Image ─────────────────────────────────────
  build:
    name: Build Image
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write

    outputs:
      image: ${{ steps.meta.outputs.tags }}
      digest: ${{ steps.push.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix=sha-
            type=semver,pattern={{version}}

      - name: Build and push
        id: push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ── 3. Deploy to Staging ──────────────────────────────────────
  deploy-staging:
    name: Deploy → Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.myapp.com

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging
        run: |
          echo "Deploying ${{ needs.build.outputs.image }}"
          # kubectl set image deployment/app app=${{ needs.build.outputs.image }}
        env:
          KUBECONFIG_DATA: ${{ secrets.STAGING_KUBECONFIG }}

  # ── 4. Deploy to Production (manual approval) ─────────────────
  deploy-prod:
    name: Deploy → Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://myapp.com

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to production
        run: echo "Deploying to production"
        env:
          KUBECONFIG_DATA: ${{ secrets.PROD_KUBECONFIG }}
```

---

## Useful Actions from the Marketplace

| Action | Use |
|---|---|
| `actions/checkout@v4` | Clone the repository |
| `actions/setup-node@v4` | Install Node.js |
| `actions/setup-python@v5` | Install Python |
| `actions/setup-go@v5` | Install Go |
| `actions/cache@v4` | Cache files between runs |
| `actions/upload-artifact@v4` | Upload files |
| `actions/download-artifact@v4` | Download files |
| `docker/login-action@v3` | Log in to a container registry |
| `docker/build-push-action@v5` | Build and push Docker images |
| `docker/metadata-action@v5` | Generate image tags and labels |
| `hashicorp/setup-terraform@v3` | Install Terraform |
| `aws-actions/configure-aws-credentials@v4` | Configure AWS credentials |
| `azure/login@v2` | Log in to Azure |
| `google-github-actions/auth@v2` | Authenticate to GCP |

---

## Quick Reference

### YAML

| Pattern | Syntax |
|---|---|
| String | `key: value` |
| Quoted string | `key: "value: with colon"` |
| Multiline literal | `key: \|` then indented block |
| Multiline folded | `key: >` then indented block |
| List | `- item` under key, or `[a, b, c]` |
| Map | `key: value` pairs, or `{k: v}` |
| Anchor | `&name` on a block |
| Alias | `*name` to reference |
| Merge | `<<: *anchor` |

### GitHub Actions

| Concept | Notes |
|---|---|
| Workflow file | `.github/workflows/*.yml` |
| Manual trigger | `on: workflow_dispatch` |
| Scheduled | `on: schedule: cron: "..."` |
| Job dependency | `needs: [job1, job2]` |
| Conditional | `if: github.ref == 'refs/heads/main'` |
| Secret | `${{ secrets.NAME }}` |
| Step output | `echo "key=value" >> $GITHUB_OUTPUT` |
| Step output use | `${{ steps.id.outputs.key }}` |
| Matrix | `strategy: matrix: os: [...]` |
| Reusable workflow | `on: workflow_call` + `uses: ./.github/workflows/x.yml` |
| Cache | `actions/cache@v4` with `key` |
| Artifact | `actions/upload-artifact@v4` + `download-artifact` |
{% endraw %}
