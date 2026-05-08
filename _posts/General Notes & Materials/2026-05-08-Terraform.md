---
layout: post
title: Terraform Full Walkthrough
date: 2026-05-08T12:00:00
categories:
  - General Notes & Materials
tags:
  - terraform
  - iac
  - devops
  - aws
  - infrastructure
author: muhammed
description: Full Terraform walkthrough — providers, resources, state, variables, modules, workspaces, and real-world AWS patterns from the ground up
toc: true
pin: false
math: false
mermaid: false
image: /assets/notes-img/terraform.png
Link: "[[]]"
Link1:
img:
---

# Terraform Full Walkthrough

---

## What is Terraform?

Terraform is an **Infrastructure as Code (IaC)** tool by HashiCorp. You describe your infrastructure in `.tf` files and Terraform figures out what to create, update, or destroy to match that description.

**Why Terraform over clicking the AWS console:**
- Infrastructure is version-controlled (Git)
- Reproducible across environments (dev/staging/prod)
- Team collaboration with code review
- Destroy everything cleanly with one command
- Works across AWS, GCP, Azure, Kubernetes, and hundreds of others

**The workflow:**

```
Write .tf files → terraform init → terraform plan → terraform apply → terraform destroy
```

---

## Installation

```bash
# Linux — via tfenv (version manager, recommended)
git clone https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
tfenv install latest
tfenv use latest

# Or direct binary
wget https://releases.hashicorp.com/terraform/1.8.0/terraform_1.8.0_linux_amd64.zip
unzip terraform_1.8.0_linux_amd64.zip && sudo mv terraform /usr/local/bin/

terraform -version
```

---

## Core Concepts

| Concept | What it is |
|---|---|
| **Provider** | Plugin that talks to an API (AWS, GCP, Kubernetes) |
| **Resource** | A single infrastructure object (EC2 instance, S3 bucket) |
| **Data source** | Read existing infrastructure you didn't create |
| **Variable** | Input parameter for your config |
| **Output** | Values exported after apply (like return values) |
| **State** | Terraform's record of what it has created |
| **Module** | Reusable group of resources |
| **Workspace** | Isolated state environments (dev/staging/prod) |

---

## Project Structure

```
project/
├── main.tf          # resources
├── variables.tf     # input variable declarations
├── outputs.tf       # output declarations
├── providers.tf     # provider config
├── terraform.tfvars # variable values (don't commit secrets)
└── versions.tf      # required Terraform and provider versions
```

For larger projects:

```
project/
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── compute/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       └── terraform.tfvars
```

---

## Providers

A provider is how Terraform talks to an API.

```hcl
# versions.tf
terraform {
  required_version = ">= 1.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"      # allow 5.x but not 6.x
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.0"
    }
  }
}
```

```hcl
# providers.tf
provider "aws" {
  region  = "eu-west-1"
  profile = "myprofile"    # from ~/.aws/credentials
}

# Multiple regions — use alias
provider "aws" {
  alias  = "us"
  region = "us-east-1"
}
```

```bash
terraform init          # downloads providers into .terraform/
terraform init -upgrade # upgrade providers to latest matching version
```

---

## Resources

A resource declares a piece of infrastructure.

```hcl
resource "<provider>_<type>" "<local_name>" {
  argument = value
}
```

```hcl
# EC2 instance
resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id    # reference another resource

  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}

# S3 bucket
resource "aws_s3_bucket" "assets" {
  bucket = "mycompany-assets-bucket"
}

resource "aws_s3_bucket_versioning" "assets" {
  bucket = aws_s3_bucket.assets.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

### Resource references

Reference another resource with `<type>.<name>.<attribute>`:

```hcl
resource "aws_instance" "app" {
  subnet_id         = aws_subnet.private.id
  security_groups   = [aws_security_group.app.id]
  iam_instance_profile = aws_iam_instance_profile.app.name
}
```

Terraform automatically infers the dependency order — it creates `aws_subnet.private` before `aws_instance.app`.

---

## Variables

### Declaring variables (`variables.tf`)

```hcl
variable "region" {
  type        = string
  description = "AWS region to deploy into"
  default     = "eu-west-1"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "allowed_cidr_blocks" {
  type    = list(string)
  default = ["10.0.0.0/8"]
}

variable "tags" {
  type = map(string)
  default = {
    Project = "myapp"
    Owner   = "platform-team"
  }
}

variable "db_password" {
  type      = string
  sensitive = true    # won't be shown in plan output or logs
}
```

### Variable types

| Type | Example |
|---|---|
| `string` | `"eu-west-1"` |
| `number` | `3` |
| `bool` | `true` |
| `list(string)` | `["a", "b", "c"]` |
| `map(string)` | `{ key = "value" }` |
| `object({...})` | Structured type |
| `any` | No type constraint |

### Setting variable values

```bash
# 1. terraform.tfvars (auto-loaded)
region        = "eu-west-1"
instance_type = "t3.small"
db_password   = "changeme"

# 2. Named .tfvars file
terraform apply -var-file="prod.tfvars"

# 3. CLI flag
terraform apply -var="instance_type=t3.large"

# 4. Environment variables (TF_VAR_ prefix)
export TF_VAR_db_password="supersecret"
terraform apply
```

Priority (highest to lowest): CLI `-var` > `*.auto.tfvars` > `terraform.tfvars` > environment variables > default

### Using variables

```hcl
resource "aws_instance" "web" {
  instance_type = var.instance_type
  tags          = var.tags
}
```

---

## Outputs

Outputs expose values after `apply` — useful for passing data between modules or displaying connection info.

```hcl
# outputs.tf
output "instance_ip" {
  description = "Public IP of the web server"
  value       = aws_instance.web.public_ip
}

output "db_endpoint" {
  description = "RDS endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = true    # hide from terminal output
}

output "vpc_id" {
  value = aws_vpc.main.id
}
```

```bash
terraform output                    # show all outputs
terraform output instance_ip        # specific output
terraform output -json              # JSON format (for scripting)
```

---

## Locals

Locals are computed values within a module — like variables but derived from other values.

```hcl
locals {
  environment = "production"
  name_prefix = "${var.project}-${local.environment}"

  common_tags = merge(var.tags, {
    Environment = local.environment
    ManagedBy   = "terraform"
  })
}

resource "aws_instance" "web" {
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web"
  })
}
```

---

## Data Sources

Data sources read existing infrastructure that Terraform didn't create.

```hcl
# Get latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Get existing VPC by tag
data "aws_vpc" "main" {
  tags = {
    Name = "main-vpc"
  }
}

# Get availability zones for current region
data "aws_availability_zones" "available" {
  state = "available"
}

# Get current AWS account ID
data "aws_caller_identity" "current" {}
```

```hcl
# Use data sources
resource "aws_instance" "web" {
  ami  = data.aws_ami.amazon_linux.id
  vpc_security_group_ids = [data.aws_security_group.web.id]
}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}
```

---

## State

Terraform tracks what it has created in a **state file** (`terraform.tfstate`). This is the source of truth.

### Local state (default — not for teams)

State is stored in `terraform.tfstate` in your project directory. **Never commit this to Git** — it contains secrets.

### Remote state (required for teams)

```hcl
# versions.tf — store state in S3 with DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/networking/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

Create the S3 bucket and DynamoDB table first (chicken-and-egg — do it manually or with a bootstrap module).

### State commands

```bash
terraform state list                          # list all resources in state
terraform state show aws_instance.web         # show state for one resource
terraform state mv aws_instance.old aws_instance.new   # rename in state
terraform state rm aws_instance.web           # remove from state (doesn't destroy)
terraform import aws_instance.web i-0abc123   # import existing resource into state
terraform refresh                             # sync state with real infrastructure
```

### Sharing state between configs (remote state data source)

```hcl
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "prod/networking/terraform.tfstate"
    region = "eu-west-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_id
}
```

---

## The Core Workflow

```bash
terraform init          # download providers, set up backend
terraform fmt           # auto-format all .tf files
terraform validate      # check for syntax and config errors
terraform plan          # show what will change (dry run)
terraform plan -out=tfplan   # save plan to file
terraform apply              # apply changes (prompts for confirmation)
terraform apply tfplan        # apply saved plan (no prompt)
terraform apply -auto-approve # skip prompt (CI/CD)
terraform destroy             # destroy all resources
terraform destroy -target=aws_instance.web   # destroy one resource
```

### Reading plan output

```
  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami           = "ami-0abc123"
      + instance_type = "t3.micro"
    }

  # aws_security_group.old will be destroyed
  - resource "aws_security_group" "old" { ... }

  # aws_instance.app will be updated in-place
  ~ resource "aws_instance" "app" {
      ~ tags = {
          ~ "Name" = "old-name" -> "new-name"
        }
    }

  # aws_db_instance.main must be replaced (destroy + create)
  -/+ resource "aws_db_instance" "main" {
        ~ identifier = "old-id" -> "new-id"   # forces replacement
      }
```

Symbols: `+` create, `-` destroy, `~` update, `-/+` replace

---

## Modules

A module is a directory of `.tf` files. Every Terraform project is technically the **root module**. You can call **child modules** to reuse infrastructure patterns.

### Calling a module

```hcl
# main.tf
module "networking" {
  source = "./modules/networking"    # local path

  vpc_cidr   = "10.0.0.0/16"
  region     = var.region
  project    = var.project
}

module "database" {
  source  = "terraform-aws-modules/rds/aws"   # Terraform Registry
  version = "~> 6.0"

  identifier = "myapp-db"
  engine     = "postgres"
  instance_class = "db.t3.medium"
}

# Use outputs from a module
resource "aws_instance" "app" {
  subnet_id = module.networking.private_subnet_id
}
```

### Writing a module

```hcl
# modules/networking/variables.tf
variable "vpc_cidr" {
  type = string
}

variable "project" {
  type = string
}

# modules/networking/main.tf
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = { Name = "${var.project}-vpc" }
}

resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.main.id
  cidr_block = cidrsubnet(var.vpc_cidr, 4, 1)
}

# modules/networking/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_id" {
  value = aws_subnet.private.id
}
```

```bash
terraform init      # re-run after adding modules to download them
```

---

## Meta-Arguments

### count — create N copies

```hcl
resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = { Name = "private-subnet-${count.index}" }
}

# Reference: aws_subnet.private[0], aws_subnet.private[1], etc.
```

### for_each — create one per map/set item

```hcl
variable "buckets" {
  default = {
    logs    = "myapp-logs-bucket"
    assets  = "myapp-assets-bucket"
    backups = "myapp-backups-bucket"
  }
}

resource "aws_s3_bucket" "buckets" {
  for_each = var.buckets
  bucket   = each.value

  tags = { Name = each.key }
}

# Reference: aws_s3_bucket.buckets["logs"], etc.
```

`for_each` is preferred over `count` for anything meaningful — items are addressed by key, not index, so removing one doesn't shift everything.

### depends_on — explicit dependency

```hcl
resource "aws_instance" "app" {
  depends_on = [aws_db_instance.main]    # wait for DB before starting app
}
```

### lifecycle — control create/destroy behavior

```hcl
resource "aws_db_instance" "main" {
  lifecycle {
    prevent_destroy       = true    # terraform destroy will error (safety net)
    create_before_destroy = true    # create new before destroying old (zero downtime)
    ignore_changes        = [password]  # don't track changes to this attribute
  }
}
```

---

## Expressions and Functions

### String interpolation

```hcl
name = "${var.project}-${var.environment}-web"
```

### Conditionals

```hcl
instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"
```

### Common built-in functions

```hcl
# String
lower("HELLO")                         # "hello"
upper("hello")                         # "HELLO"
format("%-10s %s", "hello", "world")   # formatted string
replace("hello world", "world", "k8s") # "hello k8s"
split(",", "a,b,c")                    # ["a", "b", "c"]
join("-", ["a", "b", "c"])             # "a-b-c"
trimspace("  hello  ")                 # "hello"

# Collections
length(["a", "b", "c"])                # 3
merge({a=1}, {b=2})                    # {a=1, b=2}
flatten([[1,2],[3,4]])                 # [1,2,3,4]
distinct(["a","b","a"])                # ["a","b"]
toset(["a","b","a"])                   # set (deduped)
keys({a=1, b=2})                       # ["a","b"]
values({a=1, b=2})                     # [1,2]
lookup(var.tags, "env", "default")     # get map key with fallback

# Networking
cidrsubnet("10.0.0.0/16", 8, 2)       # "10.0.2.0/24"
cidrhost("10.0.0.0/24", 5)            # "10.0.0.5"

# Encoding
base64encode("hello")
jsonencode({key = "value"})
yamlencode({key = "value"})

# Filesystem (at plan time)
file("./scripts/init.sh")
templatefile("./config.tpl", { port = 8080 })
```

### for expressions

```hcl
# List comprehension
output "subnet_ids" {
  value = [for s in aws_subnet.private : s.id]
}

# Map comprehension
output "instance_ips" {
  value = { for k, v in aws_instance.servers : k => v.private_ip }
}

# Filter with if
output "prod_instances" {
  value = [for i in aws_instance.all : i.id if i.tags["env"] == "prod"]
}
```

---

## Workspaces

Workspaces let you manage multiple state files with the same config — one per environment.

```bash
terraform workspace list               # list workspaces (default exists)
terraform workspace new staging        # create staging workspace
terraform workspace select staging     # switch to staging
terraform workspace show               # current workspace
terraform workspace delete staging     # delete workspace
```

```hcl
# Use workspace name in config
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"

  tags = {
    Environment = terraform.workspace
  }
}
```

**Workspaces vs directories:** Workspaces share config but have separate state. Separate directories (e.g., `environments/prod/`) give full isolation and are usually better for production.

---

## Real-World AWS Example

```hcl
# VPC with public and private subnets
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "${var.project}-vpc" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.project}-igw" }
}

resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "${var.project}-public-${count.index}" }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = { Name = "${var.project}-private-${count.index}" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "web" {
  name   = "${var.project}-web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## .gitignore for Terraform

```gitignore
# State files — contain secrets
*.tfstate
*.tfstate.backup
.terraform.tfstate.lock.info

# Downloaded providers
.terraform/

# Variable files with secrets
*.tfvars
!example.tfvars    # keep example files

# Saved plans
*.tfplan

# Crash logs
crash.log
```

---

## Quick Reference

| Command | What it does |
|---|---|
| `terraform init` | Download providers, set up backend |
| `terraform fmt` | Auto-format .tf files |
| `terraform validate` | Check syntax and logic |
| `terraform plan` | Show what will change |
| `terraform apply` | Create/update infrastructure |
| `terraform destroy` | Destroy all managed infrastructure |
| `terraform state list` | List resources in state |
| `terraform state show` | Show state for a resource |
| `terraform import` | Import existing resource into state |
| `terraform output` | Show output values |
| `terraform workspace new` | Create a new workspace |
| `terraform workspace select` | Switch workspace |
| `terraform taint` | Mark resource for recreation on next apply |
| `terraform graph` | Generate dependency graph (dot format) |
