# AWS & DevOps Complete Study Guide

Welcome to the complete AWS + DevOps learning notes. Each topic has a **theory guide** and a **practical assignments file** with interview questions and real-world workflows.

---

## Learning Path (Recommended Order)

### Part 1 — AWS Core Services

| Module | Topic | Theory | Assignments |
|--------|-------|--------|-------------|
| 01 | Cloud Computing Basics | [Theory](01-cloud-computing-basics/01-cloud-computing-theory.md) | [Assignments](01-cloud-computing-basics/02-cloud-computing-assignments.md) |
| 02 | EC2 | [Theory](02-ec2/01-ec2-theory.md) | [Assignments](02-ec2/02-ec2-assignments.md) |
| 03 | EBS and EFS | [Theory](03-ebs-and-efs/01-ebs-efs-theory.md) | [Assignments](03-ebs-and-efs/02-ebs-efs-assignments.md) |
| 04 | Security Groups | [Theory](04-security-groups/01-security-groups-theory.md) | [Assignments](04-security-groups/02-security-groups-assignments.md) |
| 05 | AMI | [Theory](05-ami/01-ami-theory.md) | [Assignments](05-ami/02-ami-assignments.md) |
| 06 | Snapshots | [Theory](06-snapshots/01-snapshots-theory.md) | [Assignments](06-snapshots/02-snapshots-assignments.md) |
| 07 | VPC and Networking | [Theory](07-vpc-networking/01-vpc-networking-theory.md) | [Assignments](07-vpc-networking/02-vpc-networking-assignments.md) |
| 08 | NAT Gateway | [Theory](08-nat/01-nat-theory.md) | [Assignments](08-nat/02-nat-assignments.md) |
| 09 | VPC Peering | [Theory](09-vpc-peering/01-vpc-peering-theory.md) | [Assignments](09-vpc-peering/02-vpc-peering-assignments.md) |
| 10 | S3 | [Theory](10-s3/01-s3-theory.md) | [Assignments](10-s3/02-s3-assignments.md) |
| 11 | IAM | [Theory](11-iam/01-iam-theory.md) | [Assignments](11-iam/02-iam-assignments.md) |
| 12 | Static Website Hosting | [Theory](12-static-website-hosting/01-theory.md) | [Assignments](12-static-website-hosting/02-assignments.md) |
| 13 | AWS Lambda (Serverless) | [Theory](13-lambda/01-lambda-theory.md) | [Assignments](13-lambda/02-lambda-assignments.md) |

### Part 2 — Linux & Shell Scripting

| Module | Topic | Theory | Assignments |
|--------|-------|--------|-------------|
| 14 | Linux for DevOps Engineers | [Theory](14-linux-for-devops/01-linux-theory.md) | [Assignments](14-linux-for-devops/02-linux-assignments.md) |
| 15 | Shell Scripting | [Theory](15-shell-scripting/01-shell-scripting-theory.md) | [Assignments](15-shell-scripting/02-shell-scripting-assignments.md) |

### Part 3 — DevOps Tools & CI/CD

| Module | Topic | Theory | Assignments |
|--------|-------|--------|-------------|
| 16 | Git | [Theory](16-git/01-git-theory.md) | [Assignments](16-git/02-git-assignments.md) |
| 17 | Jenkins | [Theory](17-jenkins/01-jenkins-theory.md) | [Assignments](17-jenkins/02-jenkins-assignments.md) |
| 18 | Docker | [Theory](18-docker/01-docker-theory.md) | [Assignments](18-docker/02-docker-assignments.md) |
| 19 | Kubernetes | [Theory](19-kubernetes/01-kubernetes-theory.md) | [Assignments](19-kubernetes/02-kubernetes-assignments.md) |
| 20 | Terraform | [Theory](20-terraform/01-terraform-theory.md) | [Assignments](20-terraform/02-terraform-assignments.md) |
| 21 | Ansible | [Theory](21-ansible/01-ansible-theory.md) | [Assignments](21-ansible/02-ansible-assignments.md) |

---

## What Each Guide Contains

Every theory file includes:
- Core concepts with ASCII architecture diagrams
- Real-world use cases and workflows
- Commands, code examples, and configuration
- 10–15 detailed interview questions with answers
- Cheat sheets

Every assignments file includes:
- 6+ hands-on lab assignments
- Interview-style scenario assignments
- End-to-end workflow exercises
- Quick reference cheat sheet

---

## Key Topics Per Module

### 01 — Cloud Computing Basics
- IaaS vs PaaS vs SaaS
- Public, Private, Hybrid cloud
- AWS Regions, AZs, Edge Locations
- Shared Responsibility Model
- Pricing models overview

### 02 — EC2
- Instance types and families (t, m, c, r, i)
- Launch, stop, start, terminate lifecycle
- Key pairs and SSH access
- User Data bootstrapping
- Pricing: On-Demand, Reserved, Spot, Savings Plans
- Elastic IPs

### 03 — EBS and EFS
- EBS volume types: gp3, io2, st1, sc1
- Attach, format, mount, resize volumes
- EFS: shared NFS filesystem across AZs
- When to use EBS vs EFS

### 04 — Security Groups
- Inbound and outbound rules
- Stateful vs stateless (NACLs comparison)
- Security group referencing
- Three-tier security patterns

### 05 — AMI
- AMI components and types
- Creating golden images
- Cross-region copy
- AMI lifecycle and cleanup

### 06 — Snapshots
- Incremental backup behavior
- Data Lifecycle Manager (DLM) automation
- Cross-region copy for DR
- Restore from snapshot

### 07 — VPC and Networking
- Custom VPC with public/private subnets
- Internet Gateway, Route Tables
- NACLs vs Security Groups
- VPC Flow Logs

### 08 — NAT Gateway
- Private subnet internet access
- NAT Gateway vs NAT Instance
- HA: one NAT GW per AZ
- Route table configuration

### 09 — VPC Peering
- Peering setup and route table updates
- No transitive routing limitation
- Cross-account and cross-region peering
- VPC Peering vs Transit Gateway

### 10 — S3
- Buckets, objects, keys
- Storage classes (Standard to Glacier)
- Versioning and lifecycle policies
- Bucket policies and security
- Static website hosting

### 11 — IAM
- Users, Groups, Roles, Policies
- Least privilege principle
- EC2 instance profiles (no access keys in code)
- Cross-account role assumption
- Policy simulation and auditing

### 13 — AWS Lambda
- Function anatomy (init phase, handler, cold/warm starts)
- Triggers: S3, SQS, API Gateway, EventBridge
- Concurrency: reserved vs provisioned
- VPC integration and IAM execution roles
- Error handling: DLQ, destinations, retry logic

### 14 — Linux for DevOps
- File system hierarchy (/etc, /var, /proc, /dev)
- Process management (ps, kill, systemd, journalctl)
- Networking (ss, ip, ufw, iptables)
- Users, permissions, SSH hardening
- Performance monitoring and troubleshooting

### 15 — Shell Scripting
- Script structure: shebang, `set -euo pipefail`
- Variables, arrays, conditionals, loops
- Functions, error handling, trap signals
- Text processing: awk, sed, grep
- Real-world automation scripts

### 16 — Git
- Three states: working dir, staging, repository
- Branching strategies: Git Flow, GitHub Flow, Trunk-based
- Rebasing vs merging, interactive rebase
- Git hooks and CI/CD integration
- Troubleshooting: reflog, bisect, recovery

### 17 — Jenkins
- Declarative vs Scripted pipelines
- Agents: SSH, Docker, Kubernetes
- Shared Libraries for reusable code
- Credentials management (never hardcode secrets)
- Blue/green deployments and rollbacks

### 18 — Docker
- Images, containers, layers, registry
- Dockerfile best practices (multi-stage, non-root)
- Docker Compose for multi-container apps
- Networking (bridge, host, overlay)
- Security: scanning, capabilities, read-only filesystems

### 19 — Kubernetes
- Control plane (API server, scheduler, etcd, controllers)
- Workloads: Pod, Deployment, StatefulSet, DaemonSet, Job
- Services: ClusterIP, NodePort, LoadBalancer, Ingress
- HPA, VPA, Cluster Autoscaler
- RBAC, PodSecurityContext, Network Policies

### 20 — Terraform
- HCL syntax, providers, resources, data sources
- Variables, locals, outputs, `tfvars` files
- Remote state (S3 + DynamoDB locking)
- Modules for reusable infrastructure
- CI/CD with plan/apply, tfsec scanning

### 21 — Ansible
- Agentless architecture (SSH-based)
- Inventory: static, dynamic (AWS EC2)
- Playbooks, roles, handlers, templates (Jinja2)
- Ansible Vault for secrets management
- Galaxy for community roles and collections

---

## Interview Questions Quick Reference

### AWS Lambda
- Cold start vs warm start — what causes it, how to reduce
- Synchronous vs asynchronous invocation — error handling differences
- Reserved vs provisioned concurrency
- Lambda + VPC: what's needed for private resource access
- How to handle secrets in Lambda

### Linux
- Find which process is using a port
- High load average but low CPU — what's the cause
- Difference between hard link and soft link
- What is inode? When do you run out without disk space?
- How to troubleshoot SSH connection refused

### Shell Scripting
- `set -euo pipefail` — what each flag does
- `$@` vs `$*` difference
- How to make a script idempotent
- Lock file pattern for preventing concurrent runs
- Trap signals for cleanup on exit

### Git
- Merge vs rebase — when to use each
- `git fetch` vs `git pull`
- How to revert a pushed commit safely
- Detached HEAD state — what is it and how to fix
- How to find which commit introduced a bug (bisect)

### Jenkins
- Declarative vs Scripted pipeline differences
- How to handle secrets in Jenkins pipelines
- What is a shared library and why use it
- How to implement zero-downtime deployment
- Parallel stages — syntax and when to use

### Docker
- Image vs container — the analogy
- How layer caching works and how to optimize it
- CMD vs ENTRYPOINT — the difference with examples
- How to reduce Docker image size
- Handling secrets (not in env vars, not in image)

### Kubernetes
- Pod vs ReplicaSet vs Deployment
- Liveness vs readiness probes — different purposes
- How zero-downtime rolling updates work
- ClusterIP vs NodePort vs LoadBalancer vs Ingress
- What is RBAC — ServiceAccount, Role, RoleBinding

### Terraform
- What is Terraform state and why is it critical
- `terraform apply` twice — what happens
- `terraform destroy` vs removing resource from config
- How to import existing resources
- Modules — what are they and how to use them

### Ansible
- Agentless — how does it work then?
- What is idempotency — which modules are NOT idempotent
- `include_tasks` vs `import_tasks`
- How to handle errors (ignore_errors, block/rescue/always)
- Ansible Vault — encrypt and use secrets

---

## Prerequisites

- AWS account (free tier sufficient for all labs)
- AWS CLI v2 installed and configured
- Docker Desktop (for Docker and Kubernetes labs)
- Terraform installed
- Python 3.8+ with pip
- Git configured with your identity

---

*Notes by ITkannadigaru | AWS + DevOps Complete Series 2026*
