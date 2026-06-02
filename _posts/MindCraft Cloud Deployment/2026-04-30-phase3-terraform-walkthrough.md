---
layout: post
title: "Phase 3: Provisioning a 3-Tier AWS Network with Terraform —  Walkthrough"
date: 2026-04-30T10:00:00
categories:
  - MindCraft Cloud Deployment
tags:
  - mindcraft
  - aws
  - cloud
  - terraform
  - i-ac
  - cicd
  - devsecops
  - docker
author: muhammed
description: terraform walkthrough
toc: true
pin: false
math: false
mermaid: false
image: /assets/mindcraft/phase3.png
Link: "[[2026-03-24-Introduction-about-Application]]"
Link1:
---
Phase 2 gave us a working MERN stack running in Docker. Three containers, one command,

fully isolated networks. Phase 3 takes that same architecture and provisions the real

AWS infrastructure to run it in production — using Terraform so every resource is

version-controlled, repeatable, and destroyable.

  

```bash

terraform apply   # provision everything

terraform destroy # tear it all down

```

  

That's the goal. Here's what it took to get there.

  

---

  

## What Terraform Is (and Why It Matters)

  

Terraform is Infrastructure as Code — you describe the resources you want in `.tf`

files, and Terraform figures out what to create, update, or delete to match that description.

  

The alternative is clicking through the AWS console. That approach has three problems:

  

1. **Not repeatable** — you can't reproduce the exact environment reliably

2. **Not reviewable** — there's no diff, no history, no code review

3. **Not destroyable** — deleting 35 resources by hand in the right order is slow and

   error-prone

  

With Terraform, the entire infrastructure is a text file. You `git diff` it, `git blame`

it, spin it up, tear it down, spin it up again identically. For a portfolio project, that

means you can run it for a few hours to test and screenshot, then destroy it — paying

cents instead of running a $150/month bill indefinitely.

  

---

  

## Remote State — Why It Exists and What We Used

  

Terraform tracks what it has created in a **state file** (`terraform.tfstate`). By

default this lives on your local machine — fine for solo projects, a problem for teams

or CI/CD pipelines.

  

We store state remotely in S3 with DynamoDB for locking. These two resources were

created before the Terraform code, using the AWS CLI:

  

```bash

# S3 bucket — stores the state file, versioned and encrypted

aws s3api create-bucket \

  --bucket mindcraft-tfstate-327327821586 \

  --region ap-southeast-1 \

  --create-bucket-configuration LocationConstraint=ap-southeast-1

  

aws s3api put-bucket-versioning \

  --bucket mindcraft-tfstate-327327821586 \

  --versioning-configuration Status=Enabled

  

aws s3api put-public-access-block \

  --bucket mindcraft-tfstate-327327821586 \

  --public-access-block-configuration \

  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

  

# DynamoDB table — provides state locking (prevents concurrent applies)

aws dynamodb create-table \

  --table-name mindcraft-tfstate-lock \

  --attribute-definitions AttributeName=LockID,AttributeType=S \

  --key-schema AttributeName=LockID,KeyType=HASH \

  --billing-mode PAY_PER_REQUEST \

  --region ap-southeast-1

```

  

Why the account ID in the bucket name (`327327821586`)? S3 bucket names are globally

unique across all AWS accounts. Appending your account ID is a standard pattern to

guarantee uniqueness without needing to guess.

  

The backend is configured in `terraform/versions.tf`:

  

```hcl

backend "s3" {

  bucket         = "mindcraft-tfstate-327327821586"

  key            = "mindcraft/terraform.tfstate"

  region         = "ap-southeast-1"

  dynamodb_table = "mindcraft-tfstate-lock"

  encrypt        = true

}

```

  

`encrypt = true` means the state file is encrypted at rest using S3-managed keys.

The state file contains resource IDs, outputs, and potentially sensitive values —

keeping it encrypted is standard practice.

  

---

  

## File Structure

  

```

terraform/

├── versions.tf          # provider pin + S3 backend config

├── variables.tf         # all input variables with defaults

├── main.tf              # root module — calls the 4 child modules

├── outputs.tf           # ALB DNS, instance IDs, VPC ID

└── modules/

    ├── vpc/             # VPC, 6 subnets, IGW, NAT GW, route tables

    ├── security-groups/ # sg-alb, sg-web, sg-api, sg-db

    ├── ec2/             # IAM role, 3 EC2 instances, EBS volume

    └── alb/             # ALB, target group, HTTP listener

```

  

Each module is self-contained — it takes inputs via `variables.tf`, creates resources in

`main.tf`, and exposes outputs via `outputs.tf`. The root `main.tf` wires the modules

together by passing one module's outputs as another module's inputs:

  

```hcl

module "security_groups" {

  source = "./modules/security-groups"

  vpc_id = module.vpc.vpc_id          # ← output from vpc module

}

  

module "ec2" {

  source        = "./modules/ec2"

  sg_web_id     = module.security_groups.sg_web_id   # ← output from sg module

  web_subnet_id = module.vpc.public_subnet_ids[0]    # ← output from vpc module

}

```

  

Terraform builds a dependency graph from these references and creates resources in the

correct order automatically.

  

---

  

## The VPC Module

  

The VPC is the foundation — everything else lives inside it.

  

```

10.0.0.0/16  (the full VPC)

├── 10.0.1.0/24  Public  AZ-a  — Web tier + ALB

├── 10.0.2.0/24  Public  AZ-b  — Web tier + ALB (HA)

├── 10.0.3.0/24  Private AZ-a  — Express API

├── 10.0.4.0/24  Private AZ-b  — Express API (HA)

├── 10.0.5.0/24  Private AZ-a  — MongoDB

└── 10.0.6.0/24  Private AZ-b  — MongoDB (HA)

```

  

Two Availability Zones means if one AWS data centre fails, the other keeps serving

traffic. Six subnets gives each tier its own isolated network segment.

  

**Public vs Private subnets** — the difference is the route table:

  

```hcl

# Public subnet route table — has a route to the Internet Gateway

resource "aws_route_table" "public" {

  route {

    cidr_block = "0.0.0.0/0"

    gateway_id = aws_internet_gateway.main.id

  }

}

  

# Private subnet route table — outbound only via NAT Gateway

resource "aws_route_table" "private" {

  route {

    cidr_block     = "0.0.0.0/0"

    nat_gateway_id = aws_nat_gateway.main.id

  }

}

```

  

Public subnets have a route to the Internet Gateway — instances there can receive

inbound connections from the internet (controlled by Security Groups). Private subnets

route through the NAT Gateway instead — instances there can make outbound connections

(to download packages, pull Docker images) but the internet cannot initiate an inbound

connection to them. The MongoDB tier has no NAT either — it has no internet connectivity

at all.

  

**One NAT Gateway, not two** — a second NAT in AZ-b would survive an AZ failure, but

costs an extra ~$33/month. For this project, one NAT is the right cost trade-off.

The production recommendation would note this as a known limitation.

  

---

  

## The Security Groups Module

  

Security Groups are where the 3-tier isolation becomes enforceable. The key design:

each group only accepts traffic from the security group directly above it.

  

```hcl

# sg-alb: accepts HTTP/HTTPS from anywhere (the internet)

ingress 80  from 0.0.0.0/0

ingress 443 from 0.0.0.0/0

  

# sg-web: accepts port 3000 from sg-alb only

ingress 3000 from sg-alb

  

# sg-api: accepts port 3001 from sg-web only

ingress 3001 from sg-web

  

# sg-db: accepts port 27017 from sg-api only

ingress 27017 from sg-api

```

  

The source of `sg-web`'s rule is not an IP range — it's a security group reference

(`security_groups = [aws_security_group.alb.id]`). This means: only traffic that

originated from a resource inside `sg-alb` is allowed. If I deploy a new EC2 instance

tomorrow, it cannot reach the web tier unless it's explicitly added to `sg-alb`.

  

**What this means in practice:** The MongoDB port (27017) is unreachable from the

internet even if every other control fails. To reach it, you would need to:

  

1. Bypass the ALB and reach the web EC2 directly (blocked — web EC2 has no public IP

   and only accepts from `sg-alb`)

2. OR pivot from the web tier to the API tier (blocked — API only accepts from `sg-web`)

3. AND then pivot from the API tier to the database (blocked — DB only accepts from

   `sg-api`)

  

Three independent security boundaries. This is the same isolation we implemented in

Docker Compose with `backend-net: internal: true` — now enforced at the AWS network

layer.

  

---

  

## The EC2 Module

  

Three EC2 instances — one per tier.

  

### No SSH Keys

  

```hcl

resource "aws_instance" "web" {

  ami                    = data.aws_ami.al2023.id

  instance_type          = "t3.micro"

  subnet_id              = var.web_subnet_id

  vpc_security_group_ids = [var.sg_web_id]

  iam_instance_profile   = aws_iam_instance_profile.ec2.name

  # no key_name — SSH access via SSM Session Manager instead

}

```

  

No SSH key means port 22 is never open. Instead, the instance has an IAM Instance

Profile with the `AmazonSSMManagedInstanceCore` policy, which allows AWS Systems

Manager to establish a session without any open port. If you need a shell on any

instance: `aws ssm start-session --target <instance-id>`. No key files, no port 22.

  

### AMI Data Source

  

```hcl

data "aws_ami" "al2023" {

  most_recent = true

  owners      = ["amazon"]

  filter {

    name   = "name"

    values = ["al2023-ami-*-x86_64"]

  }

}

```

  

Instead of hardcoding an AMI ID (which is region-specific and changes as new versions

release), a data source queries AWS at plan time for the latest Amazon Linux 2023 AMI.

On the first plan this resolved to `ami-064ac0bc94e195394` in ap-southeast-1.

  

### User Data — Docker Install

  

All three instances run the same bootstrap script:

  

```bash

#!/bin/bash

dnf update -y

dnf install -y docker

systemctl start docker

systemctl enable docker

usermod -a -G docker ec2-user

mkdir -p /usr/local/lib/docker/cli-plugins

curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64" \

  -o /usr/local/lib/docker/cli-plugins/docker-compose

chmod +x /usr/local/lib/docker/cli-plugins/docker-compose

```

  

This runs automatically on first boot. After the script completes, the instance has

Docker and Docker Compose ready — waiting for the CI/CD pipeline (Phase 4) to pull

and run the containers.

  

### EBS Volume for MongoDB

  

```hcl

resource "aws_ebs_volume" "mongodb_data" {

  availability_zone = aws_instance.db.availability_zone

  size              = 20

  type              = "gp3"

}

  

resource "aws_volume_attachment" "mongodb_data" {

  device_name = "/dev/sdf"

  volume_id   = aws_ebs_volume.mongodb_data.id

  instance_id = aws_instance.db.id

}

```

  

MongoDB data lives on a separate EBS volume, not the root disk. This means if the EC2

instance is terminated and a new one is launched, the volume persists and can be

reattached. The root disk is ephemeral; the data disk is permanent.

  

---

  

## The ALB Module

  

The Application Load Balancer is the only internet-facing entry point.

  

```hcl

resource "aws_lb" "main" {

  internal           = false   # internet-facing

  load_balancer_type = "application"

  security_groups    = [var.sg_alb_id]

  subnets            = var.public_subnets   # spans both AZs

}

  

resource "aws_lb_target_group" "web" {

  port        = 3000    # forwards to port 3000 on web EC2

  protocol    = "HTTP"

  target_type = "instance"

  

  health_check {

    path    = "/"

    matcher = "200-399"

  }

}

  

resource "aws_lb_listener" "http" {

  port     = 80

  protocol = "HTTP"

  default_action {

    type             = "forward"

    target_group_arn = aws_lb_target_group.web.arn

  }

}

```

  

The ALB listens on port 80 (HTTP for now — HTTPS requires a domain and ACM certificate,

which is a Phase 4 item). It forwards to the target group, which routes to the web EC2

on port 3000. The health check hits `/` and expects a 2xx or 3xx response — if the

instance fails this check, the ALB stops sending traffic to it.

  

---

  

## How to Run It

  

### First time setup

  

```bash

# 1. Install Terraform (Windows — needs new terminal after)

winget install HashiCorp.Terraform

  

# 2. Verify AWS credentials are configured

aws sts get-caller-identity

  

# 3. Initialize — downloads AWS provider, connects to S3 backend

cd terraform/

terraform init

```

  

### Plan (no cost, no changes)

  

```bash

terraform plan

```

  

Shows every resource that will be created. Review it. If anything looks wrong, fix the

code and re-run plan. Nothing is created until you explicitly apply.

  

### Apply (creates real AWS resources — costs money while running)

  

```bash

terraform apply

# Review the plan summary one more time

# Type: yes

```

  

Takes 3–5 minutes. At the end, Terraform prints the outputs:

  

```

Outputs:

alb_dns_name    = "mindcraft-alb-123456789.ap-southeast-1.elb.amazonaws.com"

web_instance_id = "i-0abc123..."

api_instance_id = "i-0def456..."

db_instance_id  = "i-0ghi789..."

vpc_id          = "vpc-0xyz..."

```

  

The ALB DNS name is your live URL. Open it in a browser.

  

### Connect to any instance (no SSH required)

  

```bash

# Web tier

aws ssm start-session --target <web_instance_id>

  

# App tier (private subnet — no public access at all)

aws ssm start-session --target <api_instance_id>

  

# DB tier

aws ssm start-session --target <db_instance_id>

```

  

### Destroy everything (stops all charges)

  

```bash

terraform destroy

# Type: yes

```

  

Deletes all 35 resources in the correct dependency order. The S3 bucket and DynamoDB

table (created manually) are not managed by Terraform and are not destroyed — they

persist for the next apply.

  

---

  

## What the Plan Created

  

```

Plan: 35 to add, 0 to change, 0 to destroy

  

module.vpc.aws_vpc.main

module.vpc.aws_subnet.public[0,1]

module.vpc.aws_subnet.app_private[0,1]

module.vpc.aws_subnet.db_private[0,1]

module.vpc.aws_internet_gateway.main

module.vpc.aws_eip.nat

module.vpc.aws_nat_gateway.main

module.vpc.aws_route_table.public

module.vpc.aws_route_table.private

module.vpc.aws_route_table_association.public[0,1]

module.vpc.aws_route_table_association.app_private[0,1]

module.vpc.aws_route_table_association.db_private[0,1]

module.security_groups.aws_security_group.alb

module.security_groups.aws_security_group.web

module.security_groups.aws_security_group.api

module.security_groups.aws_security_group.db

module.ec2.aws_iam_role.ec2

module.ec2.aws_iam_role_policy_attachment.ssm

module.ec2.aws_iam_role_policy_attachment.cloudwatch

module.ec2.aws_iam_instance_profile.ec2

module.ec2.aws_instance.web

module.ec2.aws_instance.api

module.ec2.aws_instance.db

module.ec2.aws_ebs_volume.mongodb_data

module.ec2.aws_volume_attachment.mongodb_data

module.alb.aws_lb.main

module.alb.aws_lb_target_group.web

module.alb.aws_lb_listener.http

module.alb.aws_lb_target_group_attachment.web

```

  

---

  

## What Actually Happened — The First `terraform apply`

  

The plan was clean. The apply was not — two bugs surfaced immediately, both fixed in

under five minutes.

  

### Bug 1: Em Dash in Security Group Description

  

```

Error: creating Security Group (mindcraft-sg-alb): api error InvalidParameterValue:

Value (ALB — inbound HTTP and HTTPS from internet) for parameter GroupDescription is

invalid. Character sets beyond ASCII are not supported.

```

  

The Security Group `description` field in the Terraform code used em dashes (`—`) for

readability. AWS only accepts ASCII characters in that field. Fix: replace all em dashes

with plain hyphens (`-`) in all four Security Group descriptions.

  

The VPC, NAT Gateway, IAM role, and target group had already been created before the

error. Terraform's state tracked all of that — re-running apply only created the

remaining resources.

  

### Bug 2: Root Volume Smaller Than AMI Snapshot

  

```

Error: creating EC2 Instance: api error InvalidBlockDeviceMapping:

Volume of size 20GB is smaller than snapshot 'snap-00478581527fd8ea0',

expect size >= 30GB

```

  

The Amazon Linux 2023 AMI in ap-southeast-1 ships with a 30GB root snapshot. The EC2

module specified 20GB root volumes. Fix: bump `volume_size` from 20 to 30 in the

`root_block_device` block of all three instances. (The separate MongoDB data EBS volume

stays at 20GB — no snapshot constraint there.)

  

### The Actual Output

  

After the two fixes, the full apply completed cleanly:

  

```

Apply complete! Resources: 35 added, 0 changed, 0 destroyed.

  

Outputs:

  

alb_dns_name    = "mindcraft-alb-1837161131.ap-southeast-1.elb.amazonaws.com"

api_instance_id = "i-0296fb9f7bb00a1c8"

db_instance_id  = "i-02d3d72ebfd5765bf"

vpc_id          = "vpc-0ee5a275c1d560c5f"

web_instance_id = "i-0bfa5e840a6be1214"

```

  

Total provisioning time: approximately 4 minutes. The NAT Gateway is the bottleneck —

it alone takes around 90 seconds to become available, and everything in the private

subnets waits on it.

  

The ALB URL returns 502 at this point — expected. The EC2 instances have Docker

installed and running, but no containers have been pulled yet. The ALB health check

hits port 3000 on the web instance, gets no response, and marks it unhealthy. The

infrastructure is correct; the application layer comes in Phase 4.

  

### Destroy

  

```

Destroy complete! Resources: 35 destroyed.

```

  

Clean. All 35 resources removed in the correct dependency order. The S3 bucket and

DynamoDB table (created manually before Terraform) are not managed by Terraform and

remain — they'll be there for the next apply.

  

Both bugs are now fixed in the committed code. The corrected files:

  

- `terraform/modules/security-groups/main.tf` — all four descriptions use hyphens

- `terraform/modules/ec2/main.tf` — all three instances use 30GB root volumes

  

---

  

## What's Next

  

Phase 3 provisions empty infrastructure. The EC2 instances have Docker installed and are

waiting, but no containers are running yet. Phase 4 (GitHub Actions CI/CD) will:

  

1. Build Docker images and push them to Amazon ECR

2. Connect to each EC2 via SSM and run `docker pull` + restart

3. Automate this on every push to `main` — so deploying is just `git push`

  

The ALB is already pointing at the web instance. The moment the Next.js container starts

on that instance, the URL in the Terraform output becomes a live application.

  

Source: [github.com/Mhdomer/mindcraft-aws-migration](https://github.com/Mhdomer/mindcraft-aws-migration)