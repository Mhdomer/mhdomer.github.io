---
layout: post
title: AWS CloudFormation and CDK — Infrastructure as Code
date: 2026-06-06T10:00:00
categories:
  - AWS Learning Path
  - DevOps
tags:
  - aws
  - cloudformation
  - cdk
  - iac
  - devops
  - cloud
author: muhammed
description: A full walkthrough of AWS CloudFormation and CDK — template structure, parameters, outputs, intrinsic functions, change sets, nested stacks, and CDK constructs
toc: true
pin: false
math: false
mermaid: false
image: /assets/notes-img/cloudformation.png
Link: "[[]]"
Link1:
img:
---

## What is Infrastructure as Code?

**Infrastructure as Code (IaC)** means defining your cloud infrastructure in code files — version-controlled, repeatable, auditable.
Instead of clicking through the console, you write a template or program that describes the desired state, and the tool provisions it for you.

AWS has two IaC tools:

| Tool | Approach | Language |
|---|---|---|
| **CloudFormation** | Declarative — describe WHAT you want | YAML or JSON |
| **CDK** | Imperative — write code that generates CloudFormation | TypeScript, Python, Java, Go, .NET |

Both ultimately create and manage **CloudFormation stacks**.
CDK is a layer on top of CloudFormation, not a separate system.

---

## CloudFormation

### Template Structure

A CloudFormation template is a YAML (or JSON) file with up to 10 sections.
Only `Resources` is required — everything else is optional.

```yaml
AWSTemplateFormatVersion: "2010-09-09"   # always this value if used
Description: "Production web application stack"

Parameters:
  # Input values passed at deploy time

Mappings:
  # Static lookup tables (region → AMI ID, env → instance size)

Conditions:
  # Boolean expressions (IsProd: !Equals [!Ref Env, prod])

Resources:
  # AWS resources to create — required section

Outputs:
  # Values to export for other stacks or display after deployment
```

---

## Parameters

Parameters make templates reusable — callers pass values at deploy time instead of hardcoding them.

```yaml
Parameters:
  EnvironmentName:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
    Description: "Environment name — controls instance size and Multi-AZ"

  InstanceType:
    Type: String
    Default: t3.medium
    AllowedValues: [t3.micro, t3.medium, m6g.large, m6g.xlarge]

  DBPassword:
    Type: AWS::SSM::Parameter::Value<String>
    Default: /prod/db/password   # pulls from SSM Parameter Store at deploy time
    NoEcho: true

  VpcCidr:
    Type: String
    Default: "10.0.0.0/16"
    AllowedPattern: "^(\\d{1,3}\\.){3}\\d{1,3}/\\d{1,2}$"
    ConstraintDescription: "Must be a valid CIDR block"
```

```bash
# Deploy with parameter overrides
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name prod-web \
  --parameter-overrides EnvironmentName=prod InstanceType=m6g.large
```

---

## Resources

Resources are the core of every template — they define the AWS infrastructure to create.

```yaml
Resources:
  # VPC
  AppVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentName}-vpc"

  # Security Group
  WebServerSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Web server security group"
      VpcId: !Ref AppVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0

  # RDS Instance
  Database:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot     # take a snapshot before deleting this resource
    UpdateReplacePolicy: Snapshot
    Properties:
      DBInstanceClass: !FindInMap [InstanceSizes, !Ref EnvironmentName, DB]
      Engine: postgres
      EngineVersion: "16.2"
      AllocatedStorage: 100
      StorageType: gp3
      StorageEncrypted: true
      MasterUsername: admin
      MasterUserPassword: !Ref DBPassword
      MultiAZ: !If [IsProd, true, false]
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups:
        - !Ref DatabaseSG
      BackupRetentionPeriod: 7
      DeletionProtection: !If [IsProd, true, false]
```

### DeletionPolicy

Controls what happens to a resource when its stack is deleted:

| Value | Behaviour |
|---|---|
| `Delete` | Default — resource is deleted with the stack |
| `Retain` | Resource is kept but disassociated from the stack |
| `Snapshot` | Take a snapshot then delete (supported by RDS, EBS, ElastiCache) |

Always set `DeletionPolicy: Retain` or `Snapshot` on stateful resources (databases, S3 buckets with data).

---

## Intrinsic Functions

CloudFormation provides built-in functions to reference values, transform strings, and build dynamic configurations.

```yaml
# !Ref — reference a parameter, resource, or pseudo-parameter
VpcId: !Ref AppVPC                          # returns the VPC ID

# !GetAtt — get a specific attribute of a resource
ALBDnsName: !GetAtt ApplicationLoadBalancer.DNSName

# !Sub — string substitution (${Variable} syntax)
Name: !Sub "${EnvironmentName}-${AWS::Region}-alb"
# also supports inline mappings:
Name: !Sub
  - "${Env}-${Region}-alb"
  - Env: !Ref EnvironmentName
    Region: !Ref AWS::Region

# !Join — join a list of values with a delimiter
AllowedCidrs: !Join [",", ["10.0.0.0/8", "172.16.0.0/12"]]

# !Select — select an item from a list by index
FirstSubnet: !Select [0, !Ref SubnetIds]

# !Split — split a string into a list
SubnetList: !Split [",", !Ref SubnetIdsParam]

# !FindInMap — look up a value in a Mappings section
InstanceType: !FindInMap [InstanceSizes, !Ref EnvironmentName, Web]

# !If — conditional value
MultiAZ: !If [IsProd, true, false]

# !Equals, !And, !Or, !Not — condition functions
Conditions:
  IsProd: !Equals [!Ref EnvironmentName, prod]
  IsNotDev: !Not [!Equals [!Ref EnvironmentName, dev]]
  IsProdOrStaging: !Or
    - !Equals [!Ref EnvironmentName, prod]
    - !Equals [!Ref EnvironmentName, staging]

# Pseudo-parameters (built-in, always available)
# AWS::AccountId, AWS::Region, AWS::StackName, AWS::StackId, AWS::NoValue
```

---

## Mappings

Mappings are static lookup tables — useful for region-to-AMI mappings or environment-to-size mappings.

```yaml
Mappings:
  InstanceSizes:
    dev:
      Web: t3.micro
      DB: db.t3.micro
    staging:
      Web: t3.medium
      DB: db.t3.medium
    prod:
      Web: m6g.large
      DB: db.r7g.large

  RegionAMIs:
    eu-west-1:
      AL2023: ami-0abc123
    us-east-1:
      AL2023: ami-0def456

# Usage
InstanceType: !FindInMap [InstanceSizes, !Ref EnvironmentName, Web]
ImageId: !FindInMap [RegionAMIs, !Ref AWS::Region, AL2023]
```

---

## Outputs and Cross-Stack References

**Outputs** expose values from a stack — display them after deployment or share them with other stacks.

```yaml
Outputs:
  VpcId:
    Description: "VPC ID for use by other stacks"
    Value: !Ref AppVPC
    Export:
      Name: !Sub "${EnvironmentName}-VpcId"

  ALBDnsName:
    Description: "Load balancer DNS name"
    Value: !GetAtt ApplicationLoadBalancer.DNSName
    Export:
      Name: !Sub "${EnvironmentName}-ALBDns"
```

In another stack, import with `!ImportValue`:

```yaml
Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      SubnetId: !ImportValue prod-PrivateSubnet1a
```

> Cross-stack references create a dependency — you cannot delete the exporting stack while another stack imports its values.

---

## Change Sets

A **change set** shows you exactly what changes CloudFormation will make before you apply them.
Always use change sets in production — never `update-stack` directly without reviewing first.

```bash
# Create a change set
aws cloudformation create-change-set \
  --stack-name prod-web \
  --template-body file://template.yaml \
  --change-set-name v2-migration \
  --parameters ParameterKey=EnvironmentName,ParameterValue=prod

# Review the change set
aws cloudformation describe-change-set \
  --stack-name prod-web \
  --change-set-name v2-migration

# Execute the change set
aws cloudformation execute-change-set \
  --stack-name prod-web \
  --change-set-name v2-migration

# Delete a change set without executing
aws cloudformation delete-change-set \
  --stack-name prod-web \
  --change-set-name v2-migration
```

> 📸 **SCREENSHOT:** CloudFormation → Stacks → select stack → Change Sets tab → select change set.
> Show the Changes table with Action (Add/Modify/Remove), Resource Type, and Replacement (True/False/Conditional).

---

## Nested Stacks

Large templates become hard to manage.
**Nested stacks** let you break a template into reusable modules — a parent stack that references child stacks via `AWS::CloudFormation::Stack`.

```yaml
# Parent template
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/network.yaml
      Parameters:
        EnvironmentName: !Ref EnvironmentName
        VpcCidr: !Ref VpcCidr

  AppStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/app.yaml
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VpcId
        SubnetIds: !GetAtt NetworkStack.Outputs.PrivateSubnetIds
```

---

## Stack Sets

**CloudFormation StackSets** deploy the same template across multiple accounts and regions in one operation.
Used for landing zone baselines — deploy security controls, logging, IAM roles across all accounts in an AWS Organization.

```bash
# Deploy a stack set across all accounts in an OU
aws cloudformation create-stack-set \
  --stack-set-name baseline-security \
  --template-body file://security-baseline.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false

aws cloudformation create-stack-instances \
  --stack-set-name baseline-security \
  --deployment-targets OrganizationalUnitIds=ou-abc123 \
  --regions eu-west-1 us-east-1
```

---

## Drift Detection

**Drift** occurs when someone manually changes a resource outside of CloudFormation (console, CLI, SDK).
Drift detection compares the current live state against the template-expected state.

```bash
# Start drift detection
aws cloudformation detect-stack-drift --stack-name prod-web

# Check results
aws cloudformation describe-stack-resource-drifts \
  --stack-name prod-web \
  --stack-resource-drift-status-filters MODIFIED DELETED
```

---

## AWS CDK

**CDK (Cloud Development Kit)** lets you define infrastructure using real programming languages.
Instead of YAML, you write TypeScript, Python, Go, Java, or .NET — with type safety, IDE autocomplete, loops, conditions, and reusable constructs.

CDK synthesises into CloudFormation templates under the hood.

### Construct Levels

| Level | Name | What it gives you |
|---|---|---|
| **L1** | CloudFormation Resources (`Cfn*`) | Direct 1:1 mapping to CloudFormation resource types |
| **L2** | Higher-level constructs | Opinionated defaults, helper methods, sensible security |
| **L3** | Patterns | Complete solutions (e.g. `ApplicationLoadBalancedFargateService`) |

Always prefer L2 or L3 — they handle the boilerplate and best practices for you.

### CDK Project Structure

```bash
# Install CDK CLI
npm install -g aws-cdk

# Create a new CDK app
cdk init app --language typescript
```

```
my-cdk-app/
├── bin/
│   └── my-cdk-app.ts    # entry point — creates the App and Stacks
├── lib/
│   └── my-stack.ts      # your stack definition
├── cdk.json             # CDK configuration
└── package.json
```

### Example — VPC + ECS Fargate Service

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ecsPatterns from 'aws-cdk-lib/aws-ecs-patterns';

export class AppStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // L2 construct — VPC with public + private subnets across 3 AZs
    const vpc = new ec2.Vpc(this, 'AppVpc', {
      maxAzs: 3,
      natGateways: 1,
    });

    // L2 — ECS cluster
    const cluster = new ecs.Cluster(this, 'AppCluster', { vpc });

    // L3 — complete ALB + Fargate service pattern
    const service = new ecsPatterns.ApplicationLoadBalancedFargateService(this, 'WebService', {
      cluster,
      cpu: 512,
      memoryLimitMiB: 1024,
      desiredCount: 3,
      taskImageOptions: {
        image: ecs.ContainerImage.fromRegistry('nginx:latest'),
        containerPort: 80,
      },
      publicLoadBalancer: true,
    });

    // Output the load balancer URL
    new cdk.CfnOutput(this, 'LoadBalancerUrl', {
      value: service.loadBalancer.loadBalancerDnsName,
    });
  }
}
```

### Core CDK Commands

```bash
# Synthesise to CloudFormation (preview generated template)
cdk synth

# Show diff between deployed stack and local code
cdk diff

# Deploy a stack
cdk deploy AppStack

# Deploy all stacks
cdk deploy --all

# Destroy a stack
cdk destroy AppStack

# Bootstrap — deploys CDK toolkit stack into account/region (one-time setup)
cdk bootstrap aws://123456789012/eu-west-1

# List all stacks in the app
cdk list
```

> 📸 **SCREENSHOT:** CDK CLI output of `cdk diff`.
> Show the diff table listing resources to add (green +), modify (yellow ~), and remove (red -) with resource types and IDs.

---

## Quick Reference

```bash
# CloudFormation — stacks
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE
aws cloudformation describe-stacks --stack-name prod-web
aws cloudformation describe-stack-events --stack-name prod-web  # shows progress

# Deploy (create or update)
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name prod-web \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --parameter-overrides Env=prod

# Validate a template
aws cloudformation validate-template --template-body file://template.yaml

# Delete a stack
aws cloudformation delete-stack --stack-name dev-web

# Get stack outputs
aws cloudformation describe-stacks \
  --stack-name prod-web \
  --query 'Stacks[0].Outputs'
```
