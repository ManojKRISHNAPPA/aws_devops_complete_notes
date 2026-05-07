# Amazon EC2 — Complete Guide

## Table of Contents
1. [What is EC2?](#1-what-is-ec2)
2. [EC2 Instance Types](#2-ec2-instance-types)
3. [Launching an EC2 Instance](#3-launching-an-ec2-instance)
4. [Key Pairs and SSH Access](#4-key-pairs-and-ssh-access)
5. [EC2 User Data](#5-ec2-user-data)
6. [EC2 Instance Lifecycle](#6-ec2-instance-lifecycle)
7. [EC2 Pricing Models](#7-ec2-pricing-models)
8. [EC2 Storage Options](#8-ec2-storage-options)
9. [EC2 Networking](#9-ec2-networking)
10. [EC2 Metadata Service](#10-ec2-metadata-service)
11. [EC2 Auto Scaling Overview](#11-ec2-auto-scaling-overview)
12. [EC2 Placement Groups](#12-ec2-placement-groups)
13. [Common Troubleshooting](#13-common-troubleshooting)
14. [EC2 Quick Reference Cheat Sheet](#14-ec2-quick-reference-cheat-sheet)
15. [Common Interview Questions](#15-common-interview-questions)

---

## 1. What is EC2?

**Amazon EC2 (Elastic Compute Cloud)** is AWS's virtual machine service. It allows you to run virtual servers (instances) in the cloud within minutes.

```
Physical Server (Traditional)       EC2 Instance (Cloud)
──────────────────────────────      ────────────────────────
Buy hardware (weeks/months)         Launch in minutes
Fixed CPU, RAM, storage             Choose instance type
Physical location                   Any region globally
You maintain hardware               AWS maintains hardware
Pay even when idle                  Pay per second/hour
```

**Key EC2 capabilities:**
- Launch virtual machines with various OS (Linux, Windows, macOS)
- Choose CPU, memory, storage, and networking capacity
- Scale from 1 to thousands of instances
- Full control of the OS and applications
- Stop/start/terminate on demand

---

## 2. EC2 Instance Types

EC2 instance types follow a naming pattern:

```
Instance Name:  m5.xlarge
                │ │ └── Size
                │ └──── Generation (5th generation)
                └────── Family (m = general purpose)
```

### Instance Families

| Family | Type | Use Cases |
|--------|------|-----------|
| **t** | General Purpose (burstable) | Web servers, dev/test, small DBs |
| **m** | General Purpose (balanced) | App servers, gaming, enterprise apps |
| **c** | Compute Optimized | High-performance web, batch processing, ML |
| **r** | Memory Optimized | In-memory databases, real-time analytics |
| **i** | Storage Optimized | High I/O databases, data warehouses |
| **p/g** | GPU Instances | Machine learning, video rendering |
| **x** | Memory Optimized (extreme) | SAP HANA, large in-memory DBs |

### Common Sizes (smallest to largest)
```
nano → micro → small → medium → large → xlarge → 2xlarge → 4xlarge → ...
```

### Most Common for Learning
| Instance | vCPU | RAM | Use Case |
|----------|------|-----|---------|
| t2.micro | 1 | 1 GB | Free tier, learning |
| t3.micro | 2 | 1 GB | Free tier eligible |
| t3.small | 2 | 2 GB | Small web apps |
| t3.medium | 2 | 4 GB | Moderate workloads |
| m5.large | 2 | 8 GB | Production web servers |
| c5.large | 2 | 4 GB | CPU-heavy workloads |
| r5.large | 2 | 16 GB | Memory-heavy workloads |

> **For the exam/interviews**: T instances are "burstable" — they earn CPU credits when idle and spend them during spikes. M instances are always-available balanced. C instances are for compute-heavy work.

---

## 3. Launching an EC2 Instance

### Via AWS Console

**Step 1**: Go to **EC2 → Instances → Launch instances**

**Step 2**: Choose an **AMI (Amazon Machine Image)** — the OS template
- Amazon Linux 2023 (AWS-optimized, free)
- Ubuntu 22.04 LTS
- Windows Server 2022

**Step 3**: Choose **Instance type** (t2.micro for free tier)

**Step 4**: Configure **Key pair** — needed for SSH access

**Step 5**: Configure **Network settings**
- VPC (default VPC is fine for learning)
- Subnet (any)
- Auto-assign public IP: **Enable**
- Security group: Allow SSH (port 22) from your IP

**Step 6**: Configure **Storage** (8 GB default is fine)

**Step 7**: Review and **Launch**

### Via AWS CLI

```bash
# Launch a basic EC2 instance
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \     # Amazon Linux 2 AMI (us-east-1)
  --instance-type t2.micro \
  --key-name my-key-pair \
  --security-group-ids sg-xxxxxxxx \
  --subnet-id subnet-xxxxxxxx \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=my-first-ec2}]'

# List running instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,PublicIpAddress,Tags[?Key==`Name`].Value[]]' \
  --output table

# Stop an instance
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# Start an instance
aws ec2 start-instances --instance-ids i-1234567890abcdef0

# Terminate (delete) an instance
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0
```

### Finding the Right AMI ID

```bash
# Find latest Amazon Linux 2023 AMI for us-east-1
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*" "Name=architecture,Values=x86_64" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text
```

---

## 4. Key Pairs and SSH Access

### What is a Key Pair?
A key pair is an **asymmetric encryption pair** used for secure access to EC2:
- **Public key**: AWS stores this and puts it in the instance at launch
- **Private key**: You download once and keep safe — if lost, you cannot SSH in

### Creating a Key Pair

**Console**: EC2 → Key Pairs → Create key pair
- Format: `.pem` for Linux/macOS SSH, `.ppk` for Windows PuTTY

**CLI**:
```bash
# Create key pair and save private key
aws ec2 create-key-pair \
  --key-name my-key-pair \
  --query 'KeyMaterial' \
  --output text > my-key-pair.pem

# Set correct permissions (required!)
chmod 400 my-key-pair.pem
```

### SSH into Your Instance

```bash
# Connect to Amazon Linux / RHEL
ssh -i my-key-pair.pem ec2-user@<public-ip>

# Connect to Ubuntu
ssh -i my-key-pair.pem ubuntu@<public-ip>

# Connect to Debian
ssh -i my-key-pair.pem admin@<public-ip>

# Using EC2 Instance Connect (browser-based, no key needed)
# Console: EC2 → Instances → select instance → Connect → EC2 Instance Connect
```

> **Permission errors?** The most common SSH issue is wrong key file permissions. Run `chmod 400 my-key-pair.pem` and try again.

---

## 5. EC2 User Data

**User Data** is a script that runs **once** when an EC2 instance launches for the first time. Use it to bootstrap — install software, configure settings.

```
EC2 Launch
    │
    ▼
┌──────────────────────┐
│  User Data Script    │  ← Runs as root, one time only
│  (bash script)       │
└──────────────────────┘
    │
    ▼
Instance is ready with software pre-installed
```

### Example: Launch EC2 with Nginx Installed

In the console under "Advanced details" → "User data", or via CLI:

```bash
#!/bin/bash
# Update packages
yum update -y

# Install Nginx
amazon-linux-extras install nginx1 -y

# Start and enable Nginx
systemctl start nginx
systemctl enable nginx

# Create a simple page
echo "<h1>Hello from $(hostname)</h1>" > /usr/share/nginx/html/index.html
```

### Passing User Data via CLI

```bash
# Save the script to a file
cat > user-data.sh << 'EOF'
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello World from $(hostname)</h1>" > /var/www/html/index.html
EOF

# Launch instance with user data
aws ec2 run-instances \
  --image-id ami-xxxxxxxxx \
  --instance-type t2.micro \
  --key-name my-key-pair \
  --user-data file://user-data.sh
```

> **Debugging User Data**: Check logs at `/var/log/cloud-init-output.log` to see if the script ran successfully.

---

## 6. EC2 Instance Lifecycle

```
          ┌─────────┐
          │ Pending  │  ← Instance is starting, no charge
          └────┬─────┘
               │
               ▼
          ┌─────────┐
  ┌──────►│ Running  │◄──────┐  ← Billed per second
  │       └────┬─────┘       │
  │            │ stop        │ start
  │            ▼             │
  │       ┌─────────┐        │
  │       │ Stopping│        │
  │       └────┬─────┘       │
  │            │             │
  │            ▼             │
  │       ┌─────────┐        │
  └───────│ Stopped │────────┘  ← Not billed for compute,
          └────┬─────┘           EBS storage still billed
               │ terminate
               ▼
          ┌─────────────┐
          │ Shutting    │
          │ down        │
          └──────┬──────┘
                 │
                 ▼
          ┌──────────────┐
          │  Terminated  │  ← Final state, cannot be recovered
          └──────────────┘
```

### Key Behaviors

| State | Billed? | Can Start? | Public IP? |
|-------|---------|-----------|-----------|
| Pending | No | - | Assigned |
| Running | Yes | Already running | Yes |
| Stopping | No | - | - |
| Stopped | EBS only | Yes | **Released** |
| Terminated | No | No (gone forever) | Released |

> **Important**: When you **stop** and **start** an EC2 instance, the **public IP changes** unless you use an Elastic IP!

### Elastic IP Address
- A **static public IP** that stays with your account
- You can attach/detach it from instances
- **Free** while attached to a running instance
- **Charges apply** if allocated but not attached (to prevent hoarding)

```bash
# Allocate an Elastic IP
aws ec2 allocate-address --domain vpc

# Associate with an instance
aws ec2 associate-address \
  --instance-id i-1234567890abcdef0 \
  --allocation-id eipalloc-xxxxxxxx
```

---

## 7. EC2 Pricing Models

### On-Demand
```
Pay per second (Linux) or per hour (Windows)
No upfront commitment
Most flexible but most expensive
Best for: testing, unpredictable workloads
```

### Reserved Instances (RI)
```
1-year or 3-year commitment
Up to 72% savings vs On-Demand
Payment options:
  - All Upfront (biggest discount)
  - Partial Upfront
  - No Upfront (smallest discount)
Best for: steady-state production workloads
```

### Spot Instances
```
Bid on unused AWS capacity
Up to 90% cheaper
AWS can reclaim with 2-minute notice
Best for: batch jobs, data analysis, fault-tolerant apps
```

### Savings Plans
```
Commit to $X/hour for 1 or 3 years
Automatically applies to EC2, Lambda, Fargate
More flexible than RIs
Best for: mixed workloads
```

### Comparison Table

| Model | Cost | Flexibility | Interruption Risk |
|-------|------|-------------|-------------------|
| On-Demand | $$$$ | Highest | None |
| Reserved | $$ | Low | None |
| Savings Plans | $$ | Medium | None |
| Spot | $ | Medium | Yes (2-min warning) |
| Dedicated Host | $$$$ | Low | None |

---

## 8. EC2 Storage Options

| Storage Type | Description | Persistence | Use Case |
|--------------|-------------|-------------|----------|
| **EBS** | Network-attached block storage | Persists after stop | OS disk, databases |
| **Instance Store** | Physical disk on host | Lost on stop/terminate | Temp files, cache |
| **EFS** | Network file system (NFS) | Always persistent | Shared storage across instances |
| **S3** | Object storage | Always persistent | Static files, backups |

> **Key rule**: EBS root volumes are deleted when the instance terminates by default (can be changed). Instance store is always lost.

---

## 9. EC2 Networking

### Public vs Private IP

Every EC2 instance can have:
- **Private IP**: Always assigned, stays the same while instance exists
- **Public IP**: Assigned if in a public subnet with "Auto-assign" enabled, **changes on stop/start**
- **Elastic IP**: Static public IP you manage

### EC2 and ENI (Elastic Network Interface)
- Each EC2 gets a default **ENI (Elastic Network Interface)**
- You can attach additional ENIs for multiple IPs or network interfaces
- ENIs can be moved between instances (useful for failover)

```bash
# Get instance's public IP
aws ec2 describe-instances \
  --instance-ids i-xxxx \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text

# Get private IP
aws ec2 describe-instances \
  --instance-ids i-xxxx \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text
```

---

## 10. EC2 Metadata Service

**Instance Metadata Service (IMDS)** allows EC2 instances to query their own metadata **from within the instance**.

```bash
# SSH into your EC2 instance, then run:

# Get instance ID
curl http://169.254.169.254/latest/meta-data/instance-id

# Get instance type
curl http://169.254.169.254/latest/meta-data/instance-type

# Get public IP
curl http://169.254.169.254/latest/meta-data/public-ipv4

# Get IAM role credentials (if attached)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# IMDSv2 (more secure, required for new instances)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
```

> **IP 169.254.169.254 is a link-local address** — only accessible from within the EC2 instance itself.

---

## 11. EC2 Auto Scaling Overview

**Auto Scaling** automatically adjusts the number of EC2 instances based on demand.

```
                    High Traffic
                         │
                         ▼
              ┌─────────────────────┐
              │   Auto Scaling      │
              │   Group (ASG)       │
              │                     │
              │   Min: 2 instances  │
              │   Max: 10 instances │
              │   Desired: 4        │
              └────────┬────────────┘
                       │
              Scale-out when CPU > 70%
              Scale-in when CPU < 30%
```

**Components:**
- **Launch Template**: Defines instance configuration (AMI, type, key, SG)
- **Auto Scaling Group**: The group of instances, with min/max/desired counts
- **Scaling Policy**: Rules for when to scale (CPU-based, schedule-based, custom metric)

---

## 12. EC2 Placement Groups

Control how instances are placed across underlying hardware.

| Type | Description | Use Case |
|------|-------------|---------|
| **Cluster** | Pack instances close together in same AZ | Low latency, HPC, high throughput |
| **Spread** | Spread across distinct hardware (max 7/AZ) | Critical instances, HA |
| **Partition** | Groups of instances on different racks | Large distributed systems (Kafka, Cassandra) |

---

## 13. Common Troubleshooting

### Cannot SSH into instance
```
Checklist:
□ Security group allows port 22 from your IP
□ Instance is in Running state (not Stopped)
□ Key file has correct permissions (chmod 400)
□ Using correct username (ec2-user, ubuntu, admin)
□ Instance has public IP or you're using correct private IP
□ Subnet has internet gateway route (for public instances)
```

### Instance not starting (Pending for too long)
```
□ Check instance limits in the region (default: 32 vCPU limit)
□ Spot instance may have been interrupted
□ Check AWS Health Dashboard for AZ issues
```

### Application not accessible from browser
```
□ Security group allows the app port (e.g., 80 for HTTP, 443 for HTTPS)
□ Application is actually running on the instance
□ Correct public IP/DNS being used
□ Network ACL is not blocking traffic
```

---

## 14. EC2 Quick Reference Cheat Sheet

```bash
# Instance operations
aws ec2 run-instances --image-id <ami> --instance-type t2.micro --key-name <key>
aws ec2 start-instances --instance-ids <id>
aws ec2 stop-instances --instance-ids <id>
aws ec2 terminate-instances --instance-ids <id>
aws ec2 describe-instances --output table

# Elastic IPs
aws ec2 allocate-address --domain vpc
aws ec2 associate-address --instance-id <id> --allocation-id <eip-id>
aws ec2 release-address --allocation-id <eip-id>

# Key pairs
aws ec2 create-key-pair --key-name <name> --query 'KeyMaterial' --output text > key.pem
aws ec2 describe-key-pairs
aws ec2 delete-key-pair --key-name <name>

# AMIs
aws ec2 describe-images --owners self --output table
aws ec2 create-image --instance-id <id> --name "my-ami" --description "My backup AMI"
```

---

## 15. Common Interview Questions

**Q: What is the difference between stopping and terminating an EC2 instance?**
> Stopping an instance shuts it down but keeps the EBS root volume — you can start it again later (but the public IP changes). Terminating permanently deletes the instance and, by default, also deletes the root EBS volume. Termination is irreversible.

**Q: What is an Elastic IP and when would you use it?**
> An Elastic IP is a static public IPv4 address. You use it when you need a fixed public IP that does not change when you stop/start an instance. Common use: servers where clients need a stable IP address, or when running a fail-over setup where you re-attach the EIP to a healthy instance.

**Q: What is user data in EC2?**
> User data is a script (bash, cloud-init) that runs once when an EC2 instance first launches. It is used to bootstrap the instance — install packages, configure services, download code. It runs as root and executes only on the first boot.

**Q: What is the difference between On-Demand, Reserved, and Spot instances?**
> On-Demand: pay per second, no commitment, most flexible. Reserved: 1-3 year commitment, up to 72% discount, for steady workloads. Spot: bid on unused capacity, up to 90% off, but AWS can reclaim with 2-minute warning. Choose based on workload predictability and risk tolerance.

**Q: What happens to data on an Instance Store when an instance stops?**
> Instance Store data is lost permanently when an instance is stopped, terminated, or the underlying host fails. Instance Store is ephemeral — use it only for temporary data (cache, buffers) and not for data that must persist.

**Q: What is the EC2 metadata service?**
> It is an HTTP endpoint at `169.254.169.254` accessible only from within an EC2 instance. It provides instance metadata: instance ID, type, IP, IAM role credentials, and more. Used by applications running on EC2 to query their own environment without hardcoding values. IMDSv2 is the more secure version requiring a session token.

**Q: What are placement groups and when do you use them?**
> Placement groups control how instances are placed on underlying hardware. Cluster groups pack instances together for low-latency networking (HPC workloads). Spread groups put each instance on different hardware to reduce correlated failures (critical services). Partition groups divide instances into logical partitions on different racks, used for large distributed databases.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
