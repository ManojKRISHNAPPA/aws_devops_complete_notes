# NAT Gateway — Complete Guide

## Table of Contents
1. [What is NAT and Why Do We Need It?](#1-what-is-nat-and-why-do-we-need-it)
2. [NAT Gateway vs NAT Instance](#2-nat-gateway-vs-nat-instance)
3. [How NAT Gateway Works](#3-how-nat-gateway-works)
4. [Setting Up a NAT Gateway](#4-setting-up-a-nat-gateway)
5. [NAT Gateway High Availability](#5-nat-gateway-high-availability)
6. [NAT Gateway Pricing](#6-nat-gateway-pricing)
7. [Cheat Sheet](#7-cheat-sheet)
8. [Common Interview Questions](#8-common-interview-questions)

---

## 1. What is NAT and Why Do We Need It?

**NAT (Network Address Translation)** allows resources in a **private subnet** to initiate **outbound** connections to the internet while remaining unreachable from the internet inbound.

### The Problem

```
Private Subnet Instance (10.0.10.5)
  ↓ Needs to do: dnf update -y (download packages from internet)
  
Without NAT:
  10.0.10.5 sends request to 8.8.8.8
  8.8.8.8 receives request... but HOW does it respond?
  10.0.10.5 is a PRIVATE IP — not routable on the internet!
  Response: 8.8.8.8 cannot route back to 10.0.10.5

With NAT Gateway:
  10.0.10.5 → NAT GW (54.x.x.x) → 8.8.8.8
                ↑ public IP
  8.8.8.8 responds to 54.x.x.x
  NAT GW translates and forwards back to 10.0.10.5
```

### NAT in Practice

Private subnet instances need internet access for:
- **Package updates**: `dnf update`, `apt-get upgrade`
- **Downloading software**: Docker images, GitHub code
- **External API calls**: Payment APIs, SMS, OAuth
- **AWS service access** (when not using VPC Endpoints)

---

## 2. NAT Gateway vs NAT Instance

| Feature | NAT Gateway | NAT Instance |
|---------|-------------|--------------|
| **Managed by** | AWS (fully managed) | You (self-managed EC2) |
| **Availability** | Highly available within AZ | Depends on your instance |
| **Bandwidth** | Up to 100 Gbps | Limited by instance type |
| **Maintenance** | None | You patch, update, monitor |
| **Cost** | Hourly charge + data | EC2 instance hourly |
| **Security Groups** | Not applicable | Can apply SGs |
| **Port forwarding** | Not supported | Supported |
| **Use today** | ✅ Recommended | ❌ Legacy option |

> **Always use NAT Gateway** for new deployments. NAT Instances are legacy and require extra management overhead.

---

## 3. How NAT Gateway Works

```
                          INTERNET
                             │
                             ▼
                   ┌─────────────────┐
                   │ Internet Gateway│
                   └────────┬────────┘
                            │
         ┌──────────────────┤
         │                  │
         ▼                  ▼
┌─────────────────┐  ┌─────────────────┐
│  Public Subnet  │  │  Public Subnet  │
│  (AZ-a)         │  │  (AZ-b)         │
│                 │  │                 │
│  NAT Gateway    │  │  NAT Gateway    │
│  EIP: 54.x.x.x │  │  EIP: 54.y.y.y │
└────────┬────────┘  └────────┬────────┘
         │                    │
         ▼                    ▼
┌─────────────────┐  ┌─────────────────┐
│  Private Subnet │  │  Private Subnet │
│  (AZ-a)         │  │  (AZ-b)         │
│                 │  │                 │
│  10.0.10.5      │  │  10.0.11.5      │
│  (App server)   │  │  (App server)   │
└─────────────────┘  └─────────────────┘

Traffic flow (private instance → internet):
1. Private instance sends packet: src=10.0.10.5, dst=8.8.8.8
2. Route table: 0.0.0.0/0 → NAT Gateway (in public subnet)
3. NAT Gateway replaces src IP: src=54.x.x.x, dst=8.8.8.8
4. IGW forwards to internet
5. Response: src=8.8.8.8, dst=54.x.x.x
6. NAT Gateway translates back: dst=10.0.10.5
7. Response reaches private instance
```

---

## 4. Setting Up a NAT Gateway

### Prerequisites
- A VPC with at least one public subnet
- An Elastic IP address to assign to the NAT Gateway
- Private subnet route table pointing to NAT Gateway

### Step-by-Step

```bash
# 1. Allocate an Elastic IP for the NAT Gateway
EIP_ALLOC=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' --output text)
echo "EIP Allocation: $EIP_ALLOC"

# 2. Create NAT Gateway in PUBLIC subnet
NAT_GW=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUB \
  --allocation-id $EIP_ALLOC \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=my-nat-gw}]' \
  --query 'NatGateway.NatGatewayId' --output text)

echo "NAT Gateway: $NAT_GW"

# 3. Wait for NAT Gateway to be available (takes ~1 minute)
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW
echo "NAT Gateway is ready!"

# 4. Update private subnet route table: internet traffic → NAT Gateway
aws ec2 create-route \
  --route-table-id $PRV_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW

echo "Route added: 0.0.0.0/0 → $NAT_GW"

# 5. Verify private route table
aws ec2 describe-route-tables \
  --route-table-ids $PRV_RT \
  --query 'RouteTables[0].Routes' \
  --output table
```

### Verify NAT Works from Private Subnet

```bash
# SSH into a public instance (bastion), then SSH to private instance
# From private instance, test internet access:

# Test DNS resolution
nslookup google.com

# Test HTTP download
curl -s https://api.ipify.org   # Shows your public IP (NAT GW's EIP)

# Test package download
sudo dnf check-update

echo "If these work, NAT Gateway is configured correctly!"
```

---

## 5. NAT Gateway High Availability

A NAT Gateway is **highly available within a single AZ**, but if the AZ fails, private instances in other AZs lose internet access.

### Production HA Setup: One NAT Gateway Per AZ

```bash
# Best practice for production: NAT GW in each AZ
# AZ-a setup:
EIP_A=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
NAT_A=$(aws ec2 create-nat-gateway --subnet-id $PUB_SUB_A --allocation-id $EIP_A --query 'NatGateway.NatGatewayId' --output text)

# AZ-b setup:
EIP_B=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
NAT_B=$(aws ec2 create-nat-gateway --subnet-id $PUB_SUB_B --allocation-id $EIP_B --query 'NatGateway.NatGatewayId' --output text)

aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_A $NAT_B

# Route private-subnet-az-a → NAT GW in AZ-a
aws ec2 create-route --route-table-id $PRV_RT_A --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_A

# Route private-subnet-az-b → NAT GW in AZ-b
aws ec2 create-route --route-table-id $PRV_RT_B --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_B
```

```
HA Architecture:
────────────────
           AZ-a                        AZ-b
  ┌─────────────────┐         ┌─────────────────┐
  │  Public Subnet  │         │  Public Subnet  │
  │  NAT GW (AZ-a)  │         │  NAT GW (AZ-b)  │
  └────────┬────────┘         └────────┬────────┘
           │                           │
  ┌────────┴────────┐         ┌────────┴────────┐
  │ Private Subnet  │         │ Private Subnet  │
  │ Route: → NAT-a  │         │ Route: → NAT-b  │
  └─────────────────┘         └─────────────────┘
  
If AZ-a fails: AZ-b private subnet still has internet via NAT-b
```

---

## 6. NAT Gateway Pricing

- **Hourly charge**: ~$0.045/hour per NAT Gateway
- **Data processing**: $0.045 per GB processed

**Monthly cost example:**
```
1 NAT Gateway (24/7) = 0.045 × 24 × 30 = ~$32.40/month
Data transfer (100 GB) = 100 × 0.045 = $4.50
Total: ~$37/month per NAT Gateway

HA setup (2 NAT GWs): ~$74/month
```

**Cost optimization**: Use VPC Endpoints for AWS services (S3, DynamoDB) to avoid routing traffic through NAT Gateway.

---

## 7. Cheat Sheet

```bash
# Create Elastic IP for NAT GW
aws ec2 allocate-address --domain vpc

# Create NAT Gateway (in PUBLIC subnet)
aws ec2 create-nat-gateway --subnet-id <pub-subnet> --allocation-id <eip-alloc>

# Wait for availability
aws ec2 wait nat-gateway-available --nat-gateway-ids <nat-id>

# Add route in private subnet's route table
aws ec2 create-route --route-table-id <private-rt> --destination-cidr-block 0.0.0.0/0 --nat-gateway-id <nat-id>

# List NAT Gateways
aws ec2 describe-nat-gateways --output table

# Delete NAT Gateway
aws ec2 delete-nat-gateway --nat-gateway-id <nat-id>

# Release EIP (after NAT GW is deleted)
aws ec2 release-address --allocation-id <eip-alloc>
```

---

## 8. Common Interview Questions

**Q: What is a NAT Gateway and why is it needed?**
> A NAT Gateway allows EC2 instances in private subnets to initiate outbound connections to the internet (e.g., for software updates, API calls) while remaining unreachable from inbound internet traffic. It translates the private IP to a public Elastic IP for outbound traffic and reverses the translation for responses.

**Q: What is the difference between a NAT Gateway and an Internet Gateway?**
> An Internet Gateway provides two-way internet access for public subnets — instances can initiate and receive connections. A NAT Gateway provides one-way outbound internet access for private subnets — instances can initiate connections but cannot receive inbound connections from the internet. NAT Gateway is always placed in a public subnet (it needs an IGW to reach the internet).

**Q: Why would you put a NAT Gateway in a public subnet?**
> The NAT Gateway needs internet access itself to forward traffic from private instances. It gets internet access through the Internet Gateway, which requires it to be in a public subnet (a subnet with a route to the IGW) and have a public Elastic IP.

**Q: What happens if the AZ containing your NAT Gateway fails?**
> Private instances in other AZs that route through that NAT Gateway lose internet connectivity. For production high availability, deploy one NAT Gateway per AZ and configure each AZ's private route table to use the NAT Gateway in its own AZ.

**Q: NAT Gateway vs NAT Instance — which would you choose?**
> NAT Gateway for all new deployments. It is fully managed by AWS, scales automatically up to 100 Gbps, requires no patching or monitoring, and is highly available within an AZ. NAT Instances are a legacy option requiring you to manage the EC2 instance, disable source/destination checks, and handle scaling yourself.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
