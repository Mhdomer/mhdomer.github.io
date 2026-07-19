---
layout: post
title: AWS CodePipeline and CodeBuild — CI/CD on AWS
date: 2026-06-09T10:00:00
categories:
  - AWS Learning Path
  - DevOps
tags:
  - aws
  - codepipeline
  - codebuild
  - codedeploy
  - cicd
  - devops
  - cloud
author: muhammed
description: A full walkthrough of AWS CI/CD tools — CodePipeline stages and actions, CodeBuild buildspec, CodeDeploy deployment strategies, and end-to-end pipeline examples
toc: true
pin: false
math: false
mermaid: false
image: /assets/notes-img/codepipeline.png
Link: "[[]]"
Link1:
img:
---

## AWS CI/CD Services

AWS provides a suite of managed services for building, testing, and deploying applications:

| Service | Role |
|---|---|
| **CodePipeline** | Orchestrates the CI/CD pipeline — connects source, build, test, deploy stages |
| **CodeBuild** | Managed build service — compiles code, runs tests, produces artifacts |
| **CodeDeploy** | Deploys applications to EC2, ECS, Lambda, or on-premises |
| **CodeCommit** | Managed Git repository (largely replaced by GitHub/GitLab in practice) |
| **CodeArtifact** | Managed artifact repository for npm, Maven, pip, NuGet packages |

---

## CodePipeline

**CodePipeline** is a fully managed continuous delivery service that models, visualises, and automates your release process.
A pipeline consists of a series of **stages**, each containing one or more **actions**.

### Pipeline Concepts

| Term | Meaning |
|---|---|
| **Pipeline** | The overall workflow — source → build → test → deploy |
| **Stage** | A logical phase (Source, Build, Test, Deploy) |
| **Action** | A task within a stage (checkout from GitHub, run CodeBuild, deploy to ECS) |
| **Artifact** | Files passed between stages (source code, compiled binary, Docker image digest) |
| **Artifact store** | S3 bucket where CodePipeline stores artifacts between stages |
| **Transition** | The link between stages — can be disabled to pause the pipeline |

### Action Types

| Category | Providers |
|---|---|
| **Source** | GitHub, GitLab, Bitbucket, S3, ECR, CodeCommit |
| **Build** | CodeBuild, Jenkins |
| **Test** | CodeBuild, AWS Device Farm |
| **Deploy** | CodeDeploy, ECS, S3, CloudFormation, Elastic Beanstalk, Service Catalog |
| **Invoke** | Lambda, Step Functions |
| **Approval** | Manual approval with SNS notification |

> 📸 **SCREENSHOT:** CodePipeline → Pipelines → select pipeline.
> Show the visual pipeline with Source, Build, and Deploy stages connected by arrows, each showing the action name and status (Succeeded/In Progress/Failed).

### Creating a Pipeline

```bash
# Create a pipeline via JSON definition
aws codepipeline create-pipeline --cli-input-json file://pipeline.json

# Start a pipeline execution manually
aws codepipeline start-pipeline-execution --name my-app-pipeline

# Get pipeline state (stage and action status)
aws codepipeline get-pipeline-state --name my-app-pipeline

# List pipelines
aws codepipeline list-pipelines --output table
```

### Example Pipeline — GitHub to ECS Fargate

```json
{
  "pipeline": {
    "name": "web-app-pipeline",
    "roleArn": "arn:aws:iam::123456789012:role/codepipeline-role",
    "artifactStore": {
      "type": "S3",
      "location": "my-pipeline-artifacts-bucket"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [{
          "name": "GitHub",
          "actionTypeId": {
            "category": "Source",
            "owner": "ThirdParty",
            "provider": "GitHub",
            "version": "1"
          },
          "configuration": {
            "Owner": "myorg",
            "Repo": "web-app",
            "Branch": "main",
            "OAuthToken": "{% raw %}{{resolve:secretsmanager:github-token}}{% endraw %}"
          },
          "outputArtifacts": [{"name": "SourceCode"}]
        }]
      },
      {
        "name": "Build",
        "actions": [{
          "name": "CodeBuild",
          "actionTypeId": {
            "category": "Build",
            "owner": "AWS",
            "provider": "CodeBuild",
            "version": "1"
          },
          "configuration": {
            "ProjectName": "web-app-build"
          },
          "inputArtifacts": [{"name": "SourceCode"}],
          "outputArtifacts": [{"name": "BuildOutput"}]
        }]
      },
      {
        "name": "Approve",
        "actions": [{
          "name": "ManualApproval",
          "actionTypeId": {
            "category": "Approval",
            "owner": "AWS",
            "provider": "Manual",
            "version": "1"
          },
          "configuration": {
            "NotificationArn": "arn:aws:sns:...:prod-approvals",
            "CustomData": "Approve deployment to production?"
          }
        }]
      },
      {
        "name": "Deploy",
        "actions": [{
          "name": "ECS",
          "actionTypeId": {
            "category": "Deploy",
            "owner": "AWS",
            "provider": "ECS",
            "version": "1"
          },
          "configuration": {
            "ClusterName": "prod-cluster",
            "ServiceName": "web-service",
            "FileName": "imagedefinitions.json"
          },
          "inputArtifacts": [{"name": "BuildOutput"}]
        }]
      }
    ]
  }
}
```

---

## CodeBuild

**CodeBuild** is a fully managed build service.
It compiles source code, runs tests, and produces deployable artifacts.
You don't manage servers — CodeBuild provisions a fresh environment for each build, runs it, and tears it down.

### Build Environments

Each build runs in an isolated Docker container.
You choose the base image and compute size.

| Compute Type | vCPU | Memory |
|---|---|---|
| `BUILD_GENERAL1_SMALL` | 2 | 3 GB |
| `BUILD_GENERAL1_MEDIUM` | 4 | 7 GB |
| `BUILD_GENERAL1_LARGE` | 8 | 15 GB |
| `BUILD_GENERAL1_XLARGE` | 36 | 72 GB |

AWS provides managed images for common runtimes (Amazon Linux 2023, Ubuntu, Windows Server).
You can also bring your own Docker image from ECR.

### buildspec.yml

The **buildspec** file defines the build commands.
It lives at the root of your repository.

```yaml
version: 0.2

env:
  variables:
    AWS_REGION: eu-west-1
  parameter-store:
    DB_HOST: /prod/db/host
  secrets-manager:
    API_KEY: prod/api-key:api_key

phases:
  install:
    runtime-versions:
      nodejs: 20
    commands:
      - npm install -g typescript

  pre_build:
    commands:
      - echo "Running pre-build checks"
      - npm ci
      - npm run lint
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY

  build:
    commands:
      - echo "Building application"
      - npm run build
      - npm test
      - docker build -t web-app:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - docker tag web-app:$CODEBUILD_RESOLVED_SOURCE_VERSION $ECR_REGISTRY/web-app:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - docker push $ECR_REGISTRY/web-app:$CODEBUILD_RESOLVED_SOURCE_VERSION

  post_build:
    commands:
      - echo "Build completed — creating imagedefinitions.json"
      - printf '[{"name":"web","imageUri":"%s"}]' $ECR_REGISTRY/web-app:$CODEBUILD_RESOLVED_SOURCE_VERSION > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
    - appspec.yaml
    - taskdef.json
  discard-paths: yes

cache:
  paths:
    - node_modules/**/*

reports:
  test-results:
    files:
      - "test-results/junit.xml"
    file-format: JUNIT
```

### Build Phases

| Phase | Purpose |
|---|---|
| `install` | Install build tools, runtimes |
| `pre_build` | Authenticate, install dependencies, lint |
| `build` | Compile, test, package, build Docker image |
| `post_build` | Push artifacts, generate deployment manifests |

If any phase fails, subsequent phases are skipped (except `finally` blocks).

### Environment Variables

CodeBuild has built-in environment variables available in every build:

| Variable | Value |
|---|---|
| `CODEBUILD_BUILD_ID` | Build ID (e.g. `web-app:abc-123`) |
| `CODEBUILD_BUILD_NUMBER` | Sequential build number |
| `CODEBUILD_RESOLVED_SOURCE_VERSION` | Full commit SHA |
| `CODEBUILD_SOURCE_VERSION` | Branch name or PR number |
| `CODEBUILD_BUILD_SUCCEEDING` | `1` if build succeeded so far, `0` if it failed |

```bash
# Create a CodeBuild project
aws codebuild create-project \
  --name web-app-build \
  --source Type=GITHUB,Location=https://github.com/myorg/web-app \
  --artifacts Type=S3,Location=my-artifacts-bucket \
  --environment \
    Type=LINUX_CONTAINER,\
    ComputeType=BUILD_GENERAL1_MEDIUM,\
    Image=aws/codebuild/standard:7.0,\
    PrivilegedMode=true \
  --service-role arn:aws:iam::123456789012:role/codebuild-role

# Start a build manually
aws codebuild start-build --project-name web-app-build

# List builds for a project
aws codebuild list-builds-for-project --project-name web-app-build

# Get build details (logs location, phase status, artifacts)
aws codebuild batch-get-builds --ids web-app-build:abc123
```

> 📸 **SCREENSHOT:** CodeBuild → Build projects → select project → Build history.
> Show the build list with Build run ID, Status (Succeeded/Failed), Duration, and the Phase details section showing each phase and its timing.

### Caching

Caching speeds up builds by reusing previously downloaded dependencies.

| Cache Type | Stores In | Use For |
|---|---|---|
| **S3** | S3 bucket | `node_modules`, Maven `.m2`, pip packages |
| **Local** | Build host (ephemeral) | Docker layers (when `PrivilegedMode: true`) |

```yaml
cache:
  paths:
    - node_modules/**/*         # npm packages
    - .m2/**/*                  # Maven packages
    - /root/.cache/pip/**/*     # Python pip packages
```

---

## CodeDeploy

**CodeDeploy** automates application deployments to EC2 instances, ECS services, Lambda functions, or on-premises servers.

### Deployment Types

| Type | Behaviour | Downtime |
|---|---|---|
| **In-place (EC2)** | Stops the app, installs new version, restarts | Brief downtime |
| **Blue/Green (EC2)** | New instances launched alongside old, traffic shifted, old terminated | Zero downtime |
| **Blue/Green (ECS)** | New task set launched, traffic shifted, old task set terminated | Zero downtime |
| **Lambda** | Traffic shifted from old version to new version | Zero downtime |

### Deployment Configuration

Controls how fast traffic is shifted during a deployment:

| Config | Behaviour |
|---|---|
| `CodeDeployDefault.AllAtOnce` | Shifts all traffic at once — fastest, highest risk |
| `CodeDeployDefault.HalfAtATime` | Half of instances at a time |
| `CodeDeployDefault.OneAtATime` | One instance at a time — slowest, lowest risk |
| **Canary** | Shift X% first, wait N minutes, then shift the rest (e.g. 10% for 5 min) |
| **Linear** | Shift X% every N minutes until complete (e.g. 10% every 1 minute) |

### AppSpec File

The **appspec.yaml** file tells CodeDeploy how to deploy your application.
It defines the deployment steps and lifecycle hooks.

**For EC2 deployments:**

```yaml
version: 0.0
os: linux

files:
  - source: /build/app
    destination: /opt/app

hooks:
  BeforeInstall:
    - location: scripts/stop_server.sh
      timeout: 60
      runas: root
  AfterInstall:
    - location: scripts/install_deps.sh
      timeout: 120
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 60
  ValidateService:
    - location: scripts/validate.sh
      timeout: 60
```

**Lifecycle hook order for EC2 in-place:**

```
ApplicationStop → DownloadBundle → BeforeInstall → Install →
AfterInstall → ApplicationStart → ValidateService
```

**For ECS deployments:**

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:...:task-definition/web-app:6"
        LoadBalancerInfo:
          ContainerName: "web"
          ContainerPort: 8080
        PlatformVersion: "LATEST"

Hooks:
  - BeforeAllowTraffic: "arn:aws:lambda:...:function:validate-health"
  - AfterAllowTraffic: "arn:aws:lambda:...:function:smoke-test"
```

---

## End-to-End Pipeline — GitHub to ECS with CodeDeploy Blue/Green

Here is a complete pipeline for deploying a containerised application:

```
GitHub (push to main)
    │
    ▼
CodePipeline — Source Stage
    │  (checkout code, store in S3 artifact)
    ▼
CodePipeline — Build Stage (CodeBuild)
    │  1. Run tests
    │  2. Build Docker image
    │  3. Push to ECR
    │  4. Generate imagedefinitions.json + taskdef.json + appspec.yaml
    ▼
CodePipeline — Approve Stage
    │  (SNS notification → manual approval in console)
    ▼
CodePipeline — Deploy Stage (CodeDeploy Blue/Green)
    │  1. Register new ECS task definition
    │  2. Create new task set (Green)
    │  3. Run BeforeAllowTraffic Lambda (health check)
    │  4. Shift 10% traffic to Green (canary)
    │  5. Wait 5 minutes
    │  6. Shift 100% traffic to Green
    │  7. Run AfterAllowTraffic Lambda (smoke test)
    │  8. Terminate Blue task set
    ▼
Production serving new version
```

---

## CodeArtifact

**CodeArtifact** is a managed artifact repository for software packages.
It acts as a proxy to public repositories (npmjs.com, PyPI, Maven Central) — packages are cached in your account after the first download.

Use it to:
- Cache public packages internally (avoids dependency on public internet)
- Publish private packages for internal use
- Enforce approved package lists

```bash
# Configure npm to use CodeArtifact
aws codeartifact login \
  --tool npm \
  --domain my-domain \
  --domain-owner 123456789012 \
  --repository my-npm-repo

# Now npm install uses CodeArtifact as the registry
npm install lodash
```

---

## Quick Reference

```bash
# CodePipeline
aws codepipeline list-pipelines
aws codepipeline get-pipeline-state --name my-pipeline
aws codepipeline start-pipeline-execution --name my-pipeline
aws codepipeline stop-pipeline-execution --pipeline-name my-pipeline --pipeline-execution-id abc123 --abandon

# Disable/enable a stage transition (pause before prod)
aws codepipeline disable-stage-transition \
  --pipeline-name my-pipeline \
  --stage-name Deploy \
  --transition-type Inbound \
  --reason "Waiting for maintenance window"

aws codepipeline enable-stage-transition \
  --pipeline-name my-pipeline \
  --stage-name Deploy \
  --transition-type Inbound

# CodeBuild
aws codebuild list-projects
aws codebuild start-build --project-name my-build
aws codebuild list-builds-for-project --project-name my-build --sort-order DESCENDING
aws codebuild batch-get-builds --ids my-build:abc123

# CodeDeploy
aws deploy list-applications
aws deploy list-deployments --application-name web-app --deployment-group-name prod
aws deploy get-deployment --deployment-id d-ABC123XYZ
aws deploy stop-deployment --deployment-id d-ABC123XYZ --auto-rollback-enabled
```
