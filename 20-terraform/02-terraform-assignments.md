# Terraform — Assignments

## Assignment 1: First Terraform Configuration

```hcl
# Create your first infrastructure:
# 1. Create a new directory: mkdir terraform-lab && cd terraform-lab
# 2. Write main.tf that creates:
#    - AWS VPC with CIDR 10.0.0.0/16
#    - 2 public subnets in different AZs
#    - Internet Gateway
#    - Route table routing 0.0.0.0/0 to IGW
#    - Route table associations

# provider.tf
terraform {
  required_version = ">= 1.7.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

**Tasks:**
```bash
# Initialize
terraform init

# Plan (review carefully!)
terraform plan

# Apply
terraform apply

# Show what was created
terraform show
terraform state list

# Modify: add a 3rd subnet
# Then: terraform plan (see what changes)
# Then: terraform apply

# View outputs
terraform output vpc_id

# Destroy
terraform destroy
```

---

## Assignment 2: Variables and Outputs

**Refactor the VPC configuration to use variables:**

```hcl
# variables.tf — define these variables:
# - aws_region (string, default: us-east-1)
# - project_name (string, no default)
# - environment (string, validation: dev/staging/production)
# - vpc_cidr (string, default: 10.0.0.0/16)
# - public_subnet_cidrs (list(string), default: 2 subnets)
# - tags (map(string), default: {})

# locals.tf
locals {
  common_tags = merge(var.tags, {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    CreatedDate = formatdate("YYYY-MM-DD", timestamp())
  })
  
  name_prefix = "${var.project_name}-${var.environment}"
}

# outputs.tf — output:
# - vpc_id
# - vpc_cidr_block
# - public_subnet_ids (list)
# - public_subnet_arns (list)
```

**Create environment-specific tfvars:**
```hcl
# dev.tfvars
project_name = "myapp"
environment  = "dev"
vpc_cidr     = "10.0.0.0/16"
public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]

# production.tfvars
project_name = "myapp"
environment  = "production"
vpc_cidr     = "10.10.0.0/16"
public_subnet_cidrs = ["10.10.1.0/24", "10.10.2.0/24", "10.10.3.0/24"]
```

```bash
# Deploy dev
terraform apply -var-file=dev.tfvars

# Deploy production (in real life: different backend config)
terraform apply -var-file=production.tfvars
```

---

## Assignment 3: Remote State Backend

**Set up remote state in S3 with locking:**

```hcl
# Step 1: Manually create S3 bucket and DynamoDB table
# (can't use Terraform for bootstrap — chicken-and-egg problem)

aws s3 mb s3://mycompany-terraform-state-$(whoami)
aws s3api put-bucket-versioning \
    --bucket mycompany-terraform-state-$(whoami) \
    --versioning-configuration Status=Enabled

aws dynamodb create-table \
    --table-name terraform-state-lock \
    --attribute-definitions AttributeName=LockID,AttributeType=S \
    --key-schema AttributeName=LockID,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST

# Step 2: Update provider.tf to use S3 backend
# backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state-YOUR_USERNAME"
    key            = "dev/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

# Step 3: Migrate local state to remote
terraform init -migrate-state

# Verify
aws s3 ls s3://mycompany-terraform-state-YOUR_USERNAME/dev/
```

---

## Assignment 4: Create a Reusable Module

**Build a VPC module:**

```
# Directory structure to create:
modules/
└── vpc/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── README.md

environments/
├── dev/
│   ├── main.tf
│   ├── variables.tf
│   └── terraform.tfvars
└── production/
    ├── main.tf
    ├── variables.tf
    └── terraform.tfvars
```

**Module requirements:**
```hcl
# modules/vpc/variables.tf — module accepts:
# - name (string)
# - vpc_cidr
# - public_subnet_cidrs (list)
# - private_subnet_cidrs (list)
# - azs (list)
# - enable_nat_gateway (bool, default true)
# - single_nat_gateway (bool, default false — one per AZ if false)
# - tags

# modules/vpc/main.tf — creates:
# - VPC
# - Public subnets (with map_public_ip_on_launch)
# - Private subnets
# - Internet Gateway (for public subnets)
# - Elastic IPs for NAT
# - NAT Gateways (if enable_nat_gateway=true)
# - Public route table + associations
# - Private route tables + associations (one per AZ if single_nat_gateway=false)
```

**Use the module:**
```hcl
# environments/dev/main.tf
module "vpc" {
  source = "../../modules/vpc"
  
  name                 = "dev"
  vpc_cidr             = "10.0.0.0/16"
  azs                  = ["us-east-1a", "us-east-1b"]
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.20.0/24"]
  single_nat_gateway   = true   # Save cost in dev
  
  tags = { Environment = "dev" }
}

output "vpc_id" { value = module.vpc.vpc_id }
```

---

## Assignment 5: Complete AWS Infrastructure

**Build a production-ready three-tier architecture:**

```hcl
# Create all these resources:
# 
# Networking:
# - VPC (use your module)
# - Public subnets × 2 AZs (for ALB)
# - Private subnets × 2 AZs (for app + RDS)
#
# Compute:
# - Launch Template with user data to install nginx
# - Auto Scaling Group (min: 2, max: 6, desired: 2)
# - Scale out when CPU > 70%
#
# Load Balancing:
# - Application Load Balancer in public subnets
# - Target Group with health check on /health
# - HTTP listener forwarding to target group
#
# Database:
# - RDS MySQL 8.0 in private subnets (Multi-AZ)
# - DB Subnet Group
# - Random password with random_password resource
# - Store password in Secrets Manager
#
# Security:
# - SG for ALB: inbound 80 from 0.0.0.0/0
# - SG for app: inbound 80 from ALB SG only
# - SG for RDS: inbound 3306 from app SG only
#
# Outputs:
# - ALB DNS name
# - RDS endpoint
# - Auto Scaling Group name
```

---

## Assignment 6: Terraform Import

**Import existing manually-created resources:**

```bash
# Scenario: Your colleague created an S3 bucket manually
# You need to bring it under Terraform management

# 1. Create a bucket manually (simulate the scenario)
BUCKET_NAME="terraform-import-test-$(whoami)-$(date +%s)"
aws s3 mb s3://$BUCKET_NAME

# 2. Write the Terraform resource block (empty, just to exist)
cat > import-test.tf << 'EOF'
resource "aws_s3_bucket" "imported" {
  # We'll fill this in after import
}
EOF

# 3. Import the resource
terraform import aws_s3_bucket.imported $BUCKET_NAME

# 4. View the imported state
terraform state show aws_s3_bucket.imported

# 5. Fill in the resource block based on state output
# The resource block must match current state

# 6. Run terraform plan — should show no changes
terraform plan

# 7. Document all imported resources in README
```

---

## Interview Assignment: Design IaC Strategy

**Scenario:** You've joined a company that provisions all infrastructure manually via AWS Console. They have 3 environments (dev, staging, production) with identical architecture but different sizes.

**Current state:**
- 3 VPCs (manually created)
- 6 EC2 instances per environment (18 total)
- 3 RDS instances
- Various security groups and IAM roles
- No consistency between environments

**Tasks:**
1. Design the Terraform directory structure for this company
2. Identify which existing resources to import vs recreate
3. Write the module structure (what should be a module?)
4. Create a migration plan: how do you move to Terraform without downtime?
5. Define the state backend strategy (where does state live per environment?)
6. Write the CI/CD pipeline (GitHub Actions) for Terraform changes
7. Define the policy: who can `terraform plan` vs `terraform apply`?

---

## Cheat Sheet

```bash
# Common Terraform commands
terraform init               # Initialize
terraform init -upgrade      # Upgrade providers
terraform validate           # Check syntax
terraform fmt                # Format code
terraform plan               # Preview changes
terraform plan -out=tfplan   # Save plan
terraform apply              # Apply changes
terraform apply tfplan       # Apply saved plan
terraform apply -auto-approve  # No confirmation (CI/CD)
terraform destroy             # Destroy everything
terraform destroy -target=aws_instance.web  # Destroy specific
terraform output              # Show outputs
terraform output vpc_id       # Specific output
terraform show                # Show current state
terraform state list          # List resources in state
terraform state show aws_vpc.main  # Show specific resource
terraform import aws_vpc.main vpc-abc123  # Import resource
terraform taint aws_instance.web  # Mark for recreation
terraform refresh              # Update state from reality
terraform graph | dot -Tpng > graph.png  # Dependency graph

# Debug
TF_LOG=DEBUG terraform plan
TF_LOG_PATH=/tmp/terraform.log terraform apply

# Useful tools
terraform-docs   # Generate documentation
tfsec            # Security scanning
checkov          # Compliance scanning
infracost        # Cost estimation
```
