# Terraform — Complete Guide

## Table of Contents
1. [What is Terraform?](#1-what-is-terraform)
2. [Terraform Architecture](#2-terraform-architecture)
3. [Installation and Setup](#3-installation-and-setup)
4. [HCL Syntax](#4-hcl-syntax)
5. [Providers](#5-providers)
6. [Resources and Data Sources](#6-resources-and-data-sources)
7. [Variables and Outputs](#7-variables-and-outputs)
8. [State Management](#8-state-management)
9. [Modules](#9-modules)
10. [Terraform Workspaces](#10-terraform-workspaces)
11. [Terraform Cloud and CI/CD](#11-terraform-cloud-and-cicd)
12. [Real-World AWS Infrastructure](#12-real-world-aws-infrastructure)
13. [Best Practices](#13-best-practices)
14. [Common Interview Questions](#14-common-interview-questions)

---

## 1. What is Terraform?

**Terraform** is an Infrastructure as Code (IaC) tool by HashiCorp that lets you define cloud infrastructure in declarative configuration files.

```
Manual Infrastructure:              Terraform (IaC):
────────────────────────────        ─────────────────────────────────────
Click around AWS console            Write code once, deploy anywhere
No history of what was done         Git-tracked — full change history
Hard to reproduce exactly           Reproducible infrastructure
Drift between environments          Dev/staging/prod identical
Collaboration is hard               Team reviews infra changes via PRs
Deletion is risky (what breaks?)    Dependency graph — safe deletion
```

**Terraform vs CloudFormation vs Pulumi:**
| Feature | Terraform | CloudFormation | Pulumi |
|---------|-----------|----------------|--------|
| Language | HCL | JSON/YAML | Python/JS/Go/Java |
| Multi-cloud | Yes (400+ providers) | AWS only | Yes |
| State | Terraform backend | AWS managed | Pulumi Cloud |
| Maturity | Very mature | AWS native | Growing |
| Community | Large | AWS community | Growing |

---

## 2. Terraform Architecture

```
Developer writes .tf files
         │
         ▼
terraform init     → Download providers & modules
         │
         ▼
terraform plan     → Show what will change (dry run)
         │
         ▼
terraform apply    → Apply changes to real infrastructure
         │
         ▼
terraform.tfstate  → Record of what was created

Terraform Core Flow:
┌─────────────────────────────────────────────────────────┐
│  .tf files (configuration)                               │
│  ┌──────────────┐   ┌─────────────┐  ┌───────────────┐  │
│  │  main.tf     │   │ variables.tf│  │  outputs.tf   │  │
│  │  resources   │   │  input vars │  │  return values│  │
│  └──────────────┘   └─────────────┘  └───────────────┘  │
└──────────────────────────┬──────────────────────────────┘
                           │
                    ┌──────▼───────┐
                    │ Terraform CLI │
                    │  (plan/apply) │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              │                         │
    ┌─────────▼────────┐      ┌─────────▼────────┐
    │   State Backend  │      │    Providers      │
    │  (S3 + DynamoDB) │      │  (AWS, GCP, K8s) │
    └──────────────────┘      └─────────┬─────────┘
                                        │
                               ┌────────▼────────┐
                               │  Cloud APIs      │
                               │ (AWS, GCP, Azure)│
                               └─────────────────┘
```

---

## 3. Installation and Setup

```bash
# Install Terraform (Ubuntu)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Install via tfenv (recommended — manage multiple versions)
git clone --depth=1 https://github.com/tfutils/tfenv.git ~/.tfenv
export PATH="$HOME/.tfenv/bin:$PATH"
tfenv install 1.8.5
tfenv use 1.8.5
terraform version

# Configure AWS credentials
aws configure
# Or use environment variables:
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="us-east-1"
# Or IAM role (recommended for CI/CD and EC2 instances)
```

---

## 4. HCL Syntax

```hcl
# HashiCorp Configuration Language (HCL)

# Single-line comment
/* Multi-line
   comment */

# Blocks
resource "aws_instance" "web" {
  # Arguments
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  
  # Nested block
  tags = {
    Name        = "web-server"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# Expressions
ami    = var.ami_id                 # Variable reference
region = "us-${var.region_suffix}"  # String interpolation
count  = var.prod ? 3 : 1          # Conditional expression
tags   = merge(local.common_tags, { Role = "web" })  # Function call

# Loops
resource "aws_instance" "web" {
  count         = 3
  ami           = var.ami_id
  instance_type = "t3.micro"
  tags = {
    Name = "web-${count.index}"
  }
}

# for_each (recommended over count for maps/sets)
resource "aws_security_group_rule" "ingress" {
  for_each = {
    http  = 80
    https = 443
    ssh   = 22
  }
  type        = "ingress"
  from_port   = each.value
  to_port     = each.value
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
  security_group_id = aws_security_group.web.id
}

# Dynamic blocks
resource "aws_security_group" "web" {
  name = "web-sg"
  
  dynamic "ingress" {
    for_each = var.allowed_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}

# Data types
string_val   = "hello"
number_val   = 42
bool_val     = true
list_val     = ["a", "b", "c"]
map_val      = { key = "value", env = "prod" }
set_val      = toset(["a", "b", "c"])
tuple_val    = ["hello", 42, true]   # Mixed types
object_val   = { name = string, age = number }
```

---

## 5. Providers

```hcl
# providers.tf
terraform {
  required_version = ">= 1.7.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"    # >= 5.0.0, < 6.0.0
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
  }
  
  # Remote state backend
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/main.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

# Configure AWS provider
provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = var.environment
      Owner       = "devops-team"
    }
  }
}

# Multiple provider configurations (multi-region)
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

# Use in resource:
resource "aws_instance" "eu" {
  provider = aws.eu_west
  # ...
}
```

---

## 6. Resources and Data Sources

```hcl
# Resource: CREATE something
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "${var.project}-vpc"
  }
}

# Data source: READ existing resource
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

data "aws_vpc" "existing" {
  tags = {
    Name = "existing-vpc"
  }
}

# Reference
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id   # Data source reference
  instance_type = "t3.micro"
  vpc_security_group_ids = [aws_security_group.web.id]  # Resource reference
  subnet_id     = aws_subnet.public.id
}

# Lifecycle meta-arguments
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  
  lifecycle {
    create_before_destroy = true   # Create new before destroying old
    prevent_destroy       = true   # Error if someone tries to destroy
    ignore_changes        = [ami]  # Ignore drift in these attributes
    replace_triggered_by = [       # Recreate when this changes
      aws_launch_template.app.id
    ]
  }
}
```

---

## 7. Variables and Outputs

```hcl
# variables.tf
variable "aws_region" {
  type        = string
  description = "AWS region to deploy to"
  default     = "us-east-1"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
  
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production"
  }
}

variable "instance_type" {
  type = map(string)
  default = {
    dev        = "t3.micro"
    staging    = "t3.small"
    production = "t3.large"
  }
}

variable "db_password" {
  type      = string
  sensitive = true    # Masked in output and state
}

variable "allowed_cidrs" {
  type    = list(string)
  default = ["10.0.0.0/8"]
}

# Locals (computed values, not inputs)
locals {
  name_prefix = "${var.project}-${var.environment}"
  
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }
  
  instance_type = var.instance_type[var.environment]
  
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
}

# outputs.tf
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "load_balancer_dns" {
  description = "Load Balancer DNS name"
  value       = aws_lb.main.dns_name
}

output "rds_endpoint" {
  description = "RDS connection endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = true    # Not shown in console, still in state
}

output "instance_ips" {
  value = aws_instance.web[*].public_ip  # List of all IPs
}
```

### Variable Files

```hcl
# terraform.tfvars (auto-loaded)
aws_region  = "us-east-1"
environment = "production"
project     = "myapp"
allowed_cidrs = ["10.0.0.0/8", "172.16.0.0/12"]

# production.tfvars (explicit: -var-file=production.tfvars)
environment = "production"
min_size    = 3
max_size    = 20
```

```bash
# Variable precedence (highest to lowest):
# 1. -var="key=value" flag
# 2. -var-file flag  
# 3. *.auto.tfvars files
# 4. terraform.tfvars
# 5. Environment variables: TF_VAR_variable_name
# 6. Default values in variable blocks

# Example
export TF_VAR_db_password="supersecret"
terraform apply -var="environment=production" -var-file="prod.tfvars"
```

---

## 8. State Management

```bash
# Terraform state stores the mapping between your config and real infrastructure

# State commands
terraform state list                        # List all resources in state
terraform state show aws_instance.web       # Show specific resource
terraform state mv aws_instance.old aws_instance.new  # Rename resource in state
terraform state rm aws_instance.web         # Remove from state (keep real resource)
terraform import aws_instance.web i-1234567890abcdef0  # Import existing resource

# Remote state backend (ALWAYS use in teams!)
# S3 + DynamoDB setup:
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state"
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}

# Access remote state from another Terraform config
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "networking/vpc.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.vpc.outputs.private_subnet_ids[0]
}
```

---

## 9. Modules

```
Modules = reusable Terraform configurations

Module structure:
modules/
└── vpc/
    ├── main.tf          # Resources
    ├── variables.tf     # Input variables
    ├── outputs.tf       # Output values
    └── README.md
```

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  
  tags = merge(var.tags, { Name = var.name })
}

resource "aws_subnet" "public" {
  count             = length(var.public_cidrs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.public_cidrs[count.index]
  availability_zone = var.azs[count.index]
  
  map_public_ip_on_launch = true
  
  tags = merge(var.tags, {
    Name = "${var.name}-public-${count.index + 1}"
    Tier = "public"
  })
}

# modules/vpc/outputs.tf
output "vpc_id"            { value = aws_vpc.this.id }
output "public_subnet_ids" { value = aws_subnet.public[*].id }
```

```hcl
# root/main.tf — calling the module
module "vpc" {
  source = "./modules/vpc"
  # Or from Terraform Registry:
  # source = "terraform-aws-modules/vpc/aws"
  # version = "~> 5.0"
  
  name         = "production-vpc"
  vpc_cidr     = "10.0.0.0/16"
  azs          = ["us-east-1a", "us-east-1b", "us-east-1c"]
  public_cidrs = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  
  tags = local.common_tags
}

# Reference module output
resource "aws_instance" "app" {
  subnet_id = module.vpc.public_subnet_ids[0]
}
```

---

## 10. Terraform Workspaces

```bash
# Workspaces allow multiple state files in one config
terraform workspace list           # List workspaces
terraform workspace new staging    # Create workspace
terraform workspace select prod    # Switch workspace
terraform workspace show           # Current workspace
terraform workspace delete staging # Delete workspace

# Use workspace in config
resource "aws_instance" "app" {
  instance_type = terraform.workspace == "production" ? "t3.large" : "t3.micro"
  
  tags = {
    Environment = terraform.workspace
  }
}
```

---

## 11. Terraform Cloud and CI/CD

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  push:
    branches: [main]
    paths: ['terraform/**']
  pull_request:
    paths: ['terraform/**']

env:
  TF_VERSION: '1.8.5'
  AWS_REGION: 'us-east-1'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: ${{ env.TF_VERSION }}
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v3
      with:
        role-to-assume: arn:aws:iam::123456789:role/github-actions-terraform
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Terraform Format Check
      working-directory: terraform
      run: terraform fmt -check -recursive
    
    - name: Terraform Init
      working-directory: terraform
      run: terraform init -backend=false
    
    - name: Terraform Validate
      working-directory: terraform
      run: terraform validate
    
    - name: Run tfsec (security scan)
      uses: aquasecurity/tfsec-action@v1
      with:
        working_directory: terraform
  
  plan:
    needs: validate
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
    - uses: actions/checkout@v3
    - uses: hashicorp/setup-terraform@v2
    - uses: aws-actions/configure-aws-credentials@v3
      with:
        role-to-assume: arn:aws:iam::123456789:role/github-actions-terraform
        aws-region: us-east-1
    
    - name: Terraform Plan
      working-directory: terraform
      run: |
        terraform init
        terraform plan -out=tfplan
    
    - name: Post Plan to PR
      uses: borchero/terraform-plan-comment@v1
      with:
        working-directory: terraform
  
  apply:
    needs: [validate]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production    # Requires approval
    steps:
    - uses: actions/checkout@v3
    - uses: hashicorp/setup-terraform@v2
    - uses: aws-actions/configure-aws-credentials@v3
      with:
        role-to-assume: arn:aws:iam::123456789:role/github-actions-terraform
        aws-region: us-east-1
    
    - name: Terraform Apply
      working-directory: terraform
      run: |
        terraform init
        terraform apply -auto-approve
```

---

## 12. Real-World AWS Infrastructure

```hcl
# Complete production AWS infrastructure
# File structure:
# terraform/
# ├── main.tf
# ├── variables.tf
# ├── outputs.tf
# ├── provider.tf
# ├── vpc.tf
# ├── ecs.tf
# ├── rds.tf
# ├── alb.tf
# └── modules/

# vpc.tf
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  
  tags = { Name = "${local.name_prefix}-vpc" }
}

resource "aws_subnet" "public" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 4, count.index)
  availability_zone = local.azs[count.index]
  
  map_public_ip_on_launch = true
  tags = { Name = "${local.name_prefix}-public-${count.index + 1}" }
}

resource "aws_subnet" "private" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 4, count.index + 4)
  availability_zone = local.azs[count.index]
  
  tags = { Name = "${local.name_prefix}-private-${count.index + 1}" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${local.name_prefix}-igw" }
}

resource "aws_nat_gateway" "main" {
  count         = length(local.azs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  tags          = { Name = "${local.name_prefix}-nat-${count.index + 1}" }
}

# alb.tf
resource "aws_lb" "main" {
  name               = "${local.name_prefix}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
  
  enable_deletion_protection = var.environment == "production"
  
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    enabled = true
  }
}

resource "aws_lb_target_group" "app" {
  name        = "${local.name_prefix}-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"    # For ECS Fargate
  
  health_check {
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 10
    interval            = 30
    matcher             = "200"
  }
}

# ecs.tf — ECS Fargate cluster
resource "aws_ecs_cluster" "main" {
  name = "${local.name_prefix}-cluster"
  
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_task_definition" "app" {
  family                   = "${local.name_prefix}-task"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.task_cpu
  memory                   = var.task_memory
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn
  
  container_definitions = jsonencode([
    {
      name      = "app"
      image     = "${aws_ecr_repository.app.repository_url}:${var.image_tag}"
      cpu       = var.task_cpu
      memory    = var.task_memory
      essential = true
      
      portMappings = [
        {
          containerPort = 8080
          protocol      = "tcp"
        }
      ]
      
      environment = [
        { name = "APP_ENV", value = var.environment }
      ]
      
      secrets = [
        {
          name      = "DB_PASSWORD"
          valueFrom = aws_secretsmanager_secret.db_password.arn
        }
      ]
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "app"
        }
      }
    }
  ])
}

resource "aws_ecs_service" "app" {
  name            = "${local.name_prefix}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.app_count
  launch_type     = "FARGATE"
  
  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.app.id]
    assign_public_ip = false
  }
  
  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 8080
  }
  
  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }
  
  lifecycle {
    ignore_changes = [desired_count, task_definition]  # Managed by CI/CD
  }
}
```

---

## 13. Best Practices

```
Code Organization:
✅ Split into logical files: vpc.tf, ecs.tf, rds.tf, alb.tf
✅ Use modules for reusable patterns (vpc, ecs-service, rds)
✅ Separate environments: envs/dev/, envs/staging/, envs/prod/
✅ Use terraform-docs to auto-generate README

State Management:
✅ Remote state in S3 with DynamoDB locking
✅ Separate state per environment/component
✅ Never commit .tfstate files to git
✅ Enable S3 versioning for state rollback

Security:
✅ Never hardcode secrets — use Secrets Manager, SSM, Vault
✅ Mark sensitive variables/outputs as sensitive = true
✅ Use OIDC for CI/CD authentication (no long-lived keys)
✅ Enable state encryption (S3 SSE)
✅ Run tfsec/checkov for security scanning

Workflow:
✅ Always run terraform plan before apply
✅ Review plan output in PR before merging
✅ Use -target for surgical applies (carefully)
✅ Tag ALL resources with Environment, Project, ManagedBy
✅ Run terraform fmt before commit

Naming conventions:
✅ resource "aws_vpc" "main"  — descriptive names
✅ Use locals for computed names: local.name_prefix
✅ Variables in snake_case
```

---

## 14. Common Interview Questions

**Q1: What is the difference between Terraform and Ansible?**
> Terraform: Infrastructure provisioning (create VPCs, EC2, RDS). Declarative, idempotent, state-managed. Ansible: Configuration management (install software, configure servers). Procedural/declarative, agentless, push-based. Use Terraform to create cloud infrastructure, Ansible to configure what's on it.

**Q2: What is Terraform state and why is it important?**
> State is a JSON file mapping your Terraform config to real infrastructure. It tracks resource IDs, metadata, and dependencies. Without state, Terraform can't know what exists. State enables: change detection (plan), dependency graph, resource imports. Use remote state (S3) for teams — never local state files.

**Q3: What happens if you run `terraform apply` twice?**
> Second run is a no-op if nothing changed. Terraform compares desired state (config) to current state (tfstate). If they match, no changes. This is idempotency. If config changed, only the delta is applied.

**Q4: What is the difference between `terraform destroy` and removing a resource from config?**
> `terraform destroy` destroys ALL resources in the state. Removing a resource block and running `apply` destroys only that resource. Both update state. Use `terraform state rm` to stop managing a resource without destroying it.

**Q5: How do you import existing infrastructure into Terraform?**
> `terraform import aws_instance.web i-1234567890abcdef0` — writes resource to state. Then write the matching config block manually. `terraform plan` should show no changes. Note: `import` doesn't generate config automatically (use `terraform import` + `terraform state show` to get current values).

**Q6: What are Terraform modules and why use them?**
> Modules are self-contained packages of Terraform configs. Benefits: DRY (don't repeat yourself), consistent deployments, versioning. Use for: VPC setup, ECS service, RDS cluster. Source can be: local path, Git URL, Terraform Registry. Pass variables in, get outputs back.

**Q7: How do you handle sensitive data in Terraform?**
> Mark variables/outputs as `sensitive = true`. Use AWS Secrets Manager data source. Never hardcode in .tf files. Use environment variables: `TF_VAR_db_password`. State still contains sensitive values — encrypt S3 bucket, restrict access. Consider `terraform-vault-provider` for production.

**Q8: What is `prevent_destroy` and when do you use it?**
> A lifecycle argument: `lifecycle { prevent_destroy = true }`. Causes error if config would destroy the resource. Use for production databases, S3 buckets with critical data, stateful resources. Overridden by `terraform destroy -target` with explicit confirmation.

**Q9: How do you manage multiple environments with Terraform?**
> Options: (1) Workspaces — separate state files, same config. (2) Separate directories per environment — `envs/dev/`, `envs/prod/` — each with own `.tfvars`. (3) Terragrunt — DRY wrapper, manages multiple configs. Recommended: separate directories for strong isolation between prod and dev.

**Q10: What is `terraform refresh` and when is it needed?**
> `terraform refresh` updates state to match real infrastructure (deprecated in favor of `terraform apply -refresh-only`). Use when resources were changed outside Terraform (manual changes, other tools). Shows drift between desired and actual state. In plan, refresh happens automatically.

---

*Next: [Terraform Assignments](02-terraform-assignments.md)*
