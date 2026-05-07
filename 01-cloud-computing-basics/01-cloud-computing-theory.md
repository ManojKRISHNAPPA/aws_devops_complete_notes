# Cloud Computing Basics — Complete Guide

## Table of Contents
1. [What is Cloud Computing?](#1-what-is-cloud-computing)
2. [Why Cloud Computing?](#2-why-cloud-computing)
3. [Cloud Service Models](#3-cloud-service-models)
4. [Cloud Deployment Models](#4-cloud-deployment-models)
5. [AWS Global Infrastructure](#5-aws-global-infrastructure)
6. [Regions and Availability Zones](#6-regions-and-availability-zones)
7. [Edge Locations and CloudFront](#7-edge-locations-and-cloudfront)
8. [AWS Pricing Models](#8-aws-pricing-models)
9. [AWS Free Tier](#9-aws-free-tier)
10. [AWS Management Console, CLI, and SDK](#10-aws-management-console-cli-and-sdk)
11. [Shared Responsibility Model](#11-shared-responsibility-model)
12. [Key Cloud Concepts Cheat Sheet](#12-key-cloud-concepts-cheat-sheet)
13. [Common Interview Questions](#13-common-interview-questions)

---

## 1. What is Cloud Computing?

Cloud computing means **delivering computing services — servers, storage, databases, networking, software — over the internet** (the cloud) on a **pay-as-you-go** basis.

Instead of buying and maintaining physical hardware, you **rent** computing resources from a cloud provider like AWS, Azure, or GCP.

```
Traditional IT                     Cloud Computing
────────────────                   ─────────────────
Buy hardware upfront               Pay per use
Weeks to provision                 Minutes to provision
You manage everything              Provider manages hardware
Fixed capacity                     Elastic (scale up/down)
High CapEx                         Low OpEx
```

**Key characteristics of cloud computing (NIST definition):**
- **On-demand self-service** — provision resources without human interaction from provider
- **Broad network access** — access via standard mechanisms (internet, phone, tablet)
- **Resource pooling** — shared infrastructure across multiple customers (multi-tenancy)
- **Rapid elasticity** — scale up and down quickly as demand changes
- **Measured service** — pay only for what you use

---

## 2. Why Cloud Computing?

### Problems with Traditional Data Centers

```
Traditional DC Pain Points:
┌────────────────────────────────────────┐
│ 1. Capital Expense (servers, racks)    │
│ 2. Space, power, cooling costs         │
│ 3. Long procurement time (weeks/months)│
│ 4. Over-provisioning (waste)           │
│ 5. Under-provisioning (outages)        │
│ 6. Maintenance burden on your team     │
│ 7. Disaster recovery is expensive      │
└────────────────────────────────────────┘
```

### Cloud Benefits

| Benefit | Description |
|---------|-------------|
| **Trade CapEx for OpEx** | No upfront hardware investment |
| **Massive economies of scale** | AWS buys at huge volume, passes savings to you |
| **Stop guessing capacity** | Scale exactly as needed |
| **Speed and agility** | Launch globally in minutes |
| **Go global in minutes** | Deploy to multiple regions instantly |
| **Focus on business** | AWS handles undifferentiated heavy lifting |

---

## 3. Cloud Service Models

These are the three fundamental service models. Think of them as **layers of responsibility**:

```
┌──────────────────────────────────────────────────────────┐
│                     SaaS                                 │
│         (Gmail, Salesforce, Office 365)                  │
│   Provider manages everything. You just USE the app.     │
├──────────────────────────────────────────────────────────┤
│                     PaaS                                 │
│       (Elastic Beanstalk, Heroku, App Engine)            │
│   You manage: App code, Data                             │
│   Provider manages: OS, runtime, servers, networking     │
├──────────────────────────────────────────────────────────┤
│                     IaaS                                 │
│          (AWS EC2, Azure VM, Google Compute)             │
│   You manage: OS, runtime, app, data                     │
│   Provider manages: Physical hardware, virtualization    │
└──────────────────────────────────────────────────────────┘
```

### Detailed Breakdown

#### IaaS — Infrastructure as a Service
- You get raw compute, storage, and networking
- You install your own OS, middleware, and apps
- **Maximum control**, maximum responsibility
- **Example**: Launching an EC2 instance and installing your own web server

#### PaaS — Platform as a Service
- You get a managed platform (OS + runtime pre-configured)
- You deploy your application code
- **Moderate control**, less management overhead
- **Example**: Deploying a Python app to AWS Elastic Beanstalk

#### SaaS — Software as a Service
- You get a fully managed application
- No infrastructure to manage at all
- **Minimum control**, zero management overhead
- **Example**: Using Gmail, Slack, or AWS WorkMail

---

## 4. Cloud Deployment Models

### Public Cloud
- Infrastructure is owned and operated by a third-party cloud provider (AWS, Azure, GCP)
- Shared infrastructure across many customers
- You access resources over the internet
- **Best for**: startups, variable workloads, cost-sensitive applications

### Private Cloud
- Infrastructure is dedicated to a single organization
- Can be on-premises or hosted by a provider
- Complete control and isolation
- **Best for**: highly regulated industries (banking, healthcare), compliance requirements

### Hybrid Cloud
- Mix of public and private cloud
- Data and apps can move between private and public cloud
- Common pattern: keep sensitive data on-prem, run workloads in public cloud

```
Hybrid Cloud Architecture:
┌──────────────────┐          ┌──────────────────┐
│   On-Premises    │          │   AWS (Public)   │
│   Private Cloud  │◄────────►│   Cloud          │
│                  │  VPN /   │                  │
│ ■ Sensitive DB   │ Direct   │ ■ Web servers    │
│ ■ Legacy apps    │ Connect  │ ■ Dev/test envs  │
│ ■ Compliance data│          │ ■ Analytics      │
└──────────────────┘          └──────────────────┘
```

### Multi-Cloud
- Using multiple cloud providers simultaneously
- Avoids vendor lock-in
- **Example**: Running workloads on both AWS and Azure

---

## 5. AWS Global Infrastructure

AWS has the largest cloud infrastructure in the world. It is organized into:

```
AWS Global Infrastructure:
─────────────────────────
Regions
  └── Availability Zones (AZs)
        └── Data Centers
              └── Servers, Storage, Networking
```

### By the Numbers (approximate, as of 2025)
- **33+ Regions** across the world
- **105+ Availability Zones**
- **600+ Edge Locations** (CloudFront PoPs)
- **200+ services** available

---

## 6. Regions and Availability Zones

### What is a Region?

A **Region** is a **geographic area** where AWS has a cluster of data centers. Each region is completely independent — a failure in one region does not affect another.

**Examples of AWS Regions:**
```
us-east-1        → N. Virginia (oldest, most services)
us-west-2        → Oregon
eu-west-1        → Ireland
ap-south-1       → Mumbai
ap-southeast-1   → Singapore
```

> **How to choose a region?**
> 1. **Latency** — pick the region closest to your users
> 2. **Compliance** — data residency laws (EU data stays in EU)
> 3. **Service availability** — not all services are in all regions
> 4. **Pricing** — varies by region (us-east-1 is usually cheapest)

### What is an Availability Zone (AZ)?

An **AZ is one or more discrete data centers** within a Region, each with:
- Redundant power
- Networking
- Connectivity

AZs within a region are **connected via low-latency links** but are physically separate enough to be isolated from failures.

```
Region: us-east-1 (N. Virginia)
┌─────────────────────────────────────────────────┐
│                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────┐  │
│  │  AZ: us-east │  │  AZ: us-east │  │  AZ  │  │
│  │    -1a       │  │    -1b       │  │  -1c │  │
│  │              │  │              │  │      │  │
│  │ Data Center 1│  │ Data Center 2│  │  DC3 │  │
│  └──────────────┘  └──────────────┘  └──────┘  │
│        ◄──── Low-latency links ────►            │
└─────────────────────────────────────────────────┘
```

> **Best Practice**: Always deploy across **at least 2 AZs** for high availability. If one AZ fails, your app keeps running in the other.

---

## 7. Edge Locations and CloudFront

### What is an Edge Location?
An **Edge Location** is a site where AWS **caches content** for fast delivery to end users. These are part of AWS **CloudFront** (CDN).

- Located in **major cities worldwide** (600+ locations)
- Much more numerous than regions
- Content is cached here, closer to users

```
User in Sydney                  AWS Origin (us-east-1)
       │                                │
       ▼                                │
┌─────────────────┐                     │
│  Edge Location  │◄────────────────────┘
│  Sydney, AUS    │   Content cached here
│  (CloudFront PoP)│
└─────────────────┘
       │
       ▼
  Fast response to user (ms latency vs seconds from US)
```

---

## 8. AWS Pricing Models

AWS pricing follows several models. Choose the right one to optimize costs.

### On-Demand
- Pay by the **second/hour** with no commitment
- Most expensive per unit
- **Best for**: unpredictable workloads, short-term experiments

### Reserved Instances (RI)
- Commit to 1 or 3 years in exchange for **up to 72% discount**
- Types:
  - **Standard RI** — fixed instance type, biggest discount
  - **Convertible RI** — can change instance type, smaller discount
- **Best for**: steady-state workloads, production databases

### Spot Instances
- Bid on unused AWS capacity — **up to 90% cheaper than On-Demand**
- AWS can reclaim with **2-minute warning**
- **Best for**: batch processing, CI/CD, stateless workloads

### Savings Plans
- Flexible pricing model — commit to a **dollar/hour spend for 1-3 years**
- Applies automatically to eligible usage
- **Best for**: flexible teams that mix instance types

### Dedicated Hosts
- Physical server dedicated to you
- Required for **regulatory compliance** or **BYOL (bring your own license)**
- Most expensive option

```
Cost (High → Low):    On-Demand > Reserved > Savings Plans > Spot
Control (High → Low): Dedicated > On-Demand > Reserved > Spot
```

---

## 9. AWS Free Tier

AWS offers free usage for new accounts for **12 months** on many services:

| Service | Free Tier |
|---------|-----------|
| EC2 | 750 hours/month of t2.micro or t3.micro |
| S3 | 5 GB storage, 20,000 GET requests |
| RDS | 750 hours/month of db.t2.micro |
| Lambda | 1 million requests/month (always free) |
| CloudFront | 1 TB data transfer out/month |

> **Warning**: Always set up **billing alerts** when using AWS — unexpected charges are common for beginners who forget to stop resources.

---

## 10. AWS Management Console, CLI, and SDK

### Three Ways to Interact with AWS

#### 1. AWS Management Console (Web UI)
- Browser-based GUI at `console.aws.amazon.com`
- Best for **exploration and learning**
- Visual representation of resources

#### 2. AWS CLI (Command Line Interface)
- Command-line tool for managing AWS services
- **Install**:
  ```bash
  # macOS
  brew install awscli

  # Linux
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  unzip awscliv2.zip && sudo ./aws/install
  ```
- **Configure**:
  ```bash
  aws configure
  # Prompts for: Access Key ID, Secret Key, Region, Output format
  ```
- **Basic commands**:
  ```bash
  aws s3 ls                          # List S3 buckets
  aws ec2 describe-instances         # List EC2 instances
  aws iam list-users                 # List IAM users
  ```

#### 3. AWS SDK
- Programmatic access from your application code
- Available for Python (boto3), JavaScript, Java, Go, Ruby, etc.
- **Python example**:
  ```python
  import boto3

  # Create S3 client
  s3 = boto3.client('s3')

  # List all buckets
  response = s3.list_buckets()
  for bucket in response['Buckets']:
      print(bucket['Name'])
  ```

---

## 11. Shared Responsibility Model

This is one of the **most important AWS concepts**. AWS and you share responsibility for security.

```
┌─────────────────────────────────────────────────────────────┐
│                 YOUR RESPONSIBILITY                         │
│   "Security IN the cloud"                                   │
│                                                             │
│   ■ Customer data                                           │
│   ■ Platform, applications, identity management            │
│   ■ Operating system, network, firewall configuration       │
│   ■ Client-side data encryption                             │
│   ■ Server-side encryption (file system/data)               │
│   ■ Networking traffic protection                           │
├─────────────────────────────────────────────────────────────┤
│                 AWS RESPONSIBILITY                           │
│   "Security OF the cloud"                                   │
│                                                             │
│   ■ Hardware / global infrastructure                        │
│   ■ Regions, AZs, Edge Locations                            │
│   ■ Compute, Storage, Database, Networking services         │
│   ■ Physical security of data centers                       │
└─────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: You rent an apartment. The building owner is responsible for the building's security (locks, walls, fire suppression). You are responsible for locking your own door and not leaving valuables visible.

---

## 12. Key Cloud Concepts Cheat Sheet

| Term | Meaning |
|------|---------|
| **Region** | Geographic area with multiple AZs |
| **AZ** | One or more data centers in a region |
| **Edge Location** | CDN cache point, used by CloudFront |
| **IaaS** | You manage OS and up; AWS manages hardware |
| **PaaS** | You manage app code and data only |
| **SaaS** | You just use the software |
| **On-Demand** | Pay by hour/second, no commitment |
| **Reserved** | 1 or 3 year commit, big discount |
| **Spot** | Bid on spare capacity, up to 90% off |
| **Shared Responsibility** | AWS secures the hardware; you secure your data |
| **High Availability** | Design to survive AZ failure |
| **Fault Tolerance** | System continues even with component failure |
| **Elasticity** | Auto scale up and down with demand |
| **Scalability** | Ability to handle increased load |

---

## 13. Common Interview Questions

**Q: What is the difference between a Region and an Availability Zone?**
> A Region is a geographic location (e.g., N. Virginia). An AZ is a physical data center or group of data centers within that region. A Region has 2–6 AZs. AZs are connected with low-latency networking but isolated from each other's failures.

**Q: What is the Shared Responsibility Model?**
> AWS is responsible for security "of" the cloud (physical infrastructure, hardware, managed services). You are responsible for security "in" the cloud (your data, OS configuration, network rules, IAM policies, encryption).

**Q: What are the differences between IaaS, PaaS, and SaaS?**
> IaaS gives you raw infrastructure (EC2) — you manage OS and above. PaaS gives you a runtime platform (Elastic Beanstalk) — you manage only your code. SaaS is a complete application (Gmail) — you just use it.

**Q: What is an Edge Location used for?**
> Edge Locations are used by Amazon CloudFront (CDN) to cache content close to end users, reducing latency. They are different from AZs — there are 600+ edge locations versus ~105 AZs.

**Q: Why would you deploy to multiple AZs?**
> For **high availability**. If one AZ experiences an outage (power failure, networking issue), your application continues to serve traffic from other AZs. Single-AZ deployments have a single point of failure.

**Q: What is the difference between CapEx and OpEx in cloud context?**
> CapEx (Capital Expenditure) is upfront investment in hardware — traditional data centers. OpEx (Operational Expenditure) is ongoing usage-based costs — the cloud model. Cloud converts CapEx to OpEx, reducing upfront risk.

**Q: What is elasticity vs scalability?**
> Scalability is the ability to handle more load by adding resources (scale up/out). Elasticity is automatic scaling — the system automatically adds resources under load and removes them when load drops, optimizing cost.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
