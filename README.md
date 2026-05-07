# AWS Certification Notes — Complete Study Guide

Welcome to the AWS learning notes, structured similarly to the CKA certification series.
Each topic has a **theory guide** and a **practical assignments file**.

---

## Learning Path (Recommended Order)

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

---

## Common Interview Questions (Quick Reference)

See the "Common Interview Questions" section at the end of each theory file.

### Top Questions by Category

**Cloud Basics**
- Explain the Shared Responsibility Model
- What is the difference between a Region and an AZ?

**EC2**
- Stop vs terminate? What happens to data?
- What is user data? How do you bootstrap an EC2?
- On-Demand vs Reserved vs Spot?

**Storage**
- EBS vs EFS — when to use which?
- Are EBS snapshots incremental?
- What is an AMI?

**Networking**
- What makes a subnet public vs private?
- How does a NAT Gateway work?
- What is VPC peering and its limitations?

**Security**
- Security Groups vs NACLs
- IAM user vs IAM role
- How does EC2 access S3 without hardcoded keys?

**S3**
- How to make S3 public?
- What is S3 versioning?
- CRR vs SRR?

---

## Prerequisites

- AWS account (free tier is sufficient for all labs)
- AWS CLI v2 installed and configured
- Basic Linux command line knowledge
- SSH client

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
