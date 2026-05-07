# VPC and Networking Basics — Complete Guide

## Table of Contents
1. [What is a VPC?](#1-what-is-a-vpc)
2. [VPC Components Overview](#2-vpc-components-overview)
3. [CIDR and IP Addressing](#3-cidr-and-ip-addressing)
4. [Subnets (Public and Private)](#4-subnets-public-and-private)
5. [Internet Gateway](#5-internet-gateway)
6. [Route Tables](#6-route-tables)
7. [How Traffic Flows in a VPC](#7-how-traffic-flows-in-a-vpc)
8. [Network ACLs (NACLs)](#8-network-acls-nacls)
9. [VPC Flow Logs](#9-vpc-flow-logs)
10. [Custom VPC: Full Build](#10-custom-vpc-full-build)
11. [Default VPC](#11-default-vpc)
12. [Cheat Sheet](#12-cheat-sheet)
13. [Common Interview Questions](#13-common-interview-questions)

---

## 1. What is a VPC?

A **VPC (Virtual Private Cloud)** is your own **private, isolated network** within AWS. It is logically isolated from other AWS customers' networks.

```
AWS Global
│
└── Region: us-east-1
      │
      └── Your VPC: 10.0.0.0/16
            │
            ├── Public Subnet: 10.0.1.0/24 (AZ-a)
            │     ├── EC2 Web Server (has public IP)
            │     └── NAT Gateway
            │
            ├── Private Subnet: 10.0.2.0/24 (AZ-a)
            │     ├── EC2 App Server (no public IP)
            │     └── RDS Database
            │
            └── Private Subnet: 10.0.3.0/24 (AZ-b)
                  └── RDS Standby (Multi-AZ)
```

**Key VPC facts:**
- Each AWS account gets a **Default VPC** in every region (pre-configured, ready to use)
- You can create **up to 5 VPCs per region** per account (soft limit, can increase)
- VPCs span a **single region** but can span multiple AZs
- VPCs are **isolated** by default — no traffic between VPCs unless explicitly configured (peering, transit gateway)
- You define the IP range using **CIDR notation**

---

## 2. VPC Components Overview

```
VPC
│
├── CIDR Block (IP range definition)
│
├── Subnets (slices of the VPC IP range, locked to an AZ)
│     ├── Public Subnet  ──► Has route to Internet Gateway
│     └── Private Subnet ──► No direct route to internet
│
├── Internet Gateway (IGW)
│     └── Allows internet in/out for public subnets
│
├── Route Tables
│     ├── Main Route Table (default)
│     └── Custom Route Tables (one per subnet typically)
│
├── Network ACLs (NACLs)
│     └── Stateless firewall at the subnet boundary
│
├── Security Groups
│     └── Stateful firewall at the instance level
│
├── NAT Gateway / NAT Instance
│     └── Allows private subnet instances to reach internet (outbound only)
│
└── VPC Peering / Transit Gateway
      └── Connect to other VPCs
```

---

## 3. CIDR and IP Addressing

**CIDR (Classless Inter-Domain Routing)** notation defines a range of IP addresses.

```
Format: <IP address>/<prefix length>
Example: 10.0.0.0/16

/16 means: first 16 bits are the network portion
           last 16 bits are the host portion
           = 2^16 = 65,536 IP addresses
```

### Common CIDR Sizes for VPC

| CIDR | Number of IPs | Typical Use |
|------|---------------|-------------|
| /16 | 65,536 | Large VPC, many subnets |
| /24 | 256 | Medium subnet |
| /28 | 16 | Very small subnet |

> AWS **reserves 5 IP addresses** per subnet:
> - .0 — Network address
> - .1 — VPC router
> - .2 — DNS server
> - .3 — Future use
> - .255 — Broadcast (not used, but reserved)
>
> So a /24 (256 total) gives you **251 usable IPs**.

### Subnet Planning Example

```
VPC CIDR: 10.0.0.0/16 (65,536 IPs total)

Planning:
┌──────────────────────┬──────────────┬─────────────┐
│ Subnet               │ CIDR         │ Usable IPs  │
├──────────────────────┼──────────────┼─────────────┤
│ Public Subnet AZ-a   │ 10.0.1.0/24  │ 251         │
│ Public Subnet AZ-b   │ 10.0.2.0/24  │ 251         │
│ Private App AZ-a     │ 10.0.10.0/24 │ 251         │
│ Private App AZ-b     │ 10.0.11.0/24 │ 251         │
│ Private DB AZ-a      │ 10.0.20.0/24 │ 251         │
│ Private DB AZ-b      │ 10.0.21.0/24 │ 251         │
└──────────────────────┴──────────────┴─────────────┘
```

---

## 4. Subnets (Public and Private)

### What Makes a Subnet "Public"?

A subnet is **public** if it has:
1. A route to an **Internet Gateway** in its route table
2. Resources in it have **public IP addresses** (or Elastic IPs)

A subnet is **private** if it has:
1. **No direct route** to the Internet Gateway
2. Resources only have **private IP addresses**

```
Public Subnet (10.0.1.0/24):
  Route Table:
    10.0.0.0/16 → local     (VPC-internal traffic)
    0.0.0.0/0   → igw-xxxx  (internet traffic via IGW)
  
  EC2 instance: 10.0.1.10 (private) + 54.x.x.x (public)
  ↑ Reachable from internet

Private Subnet (10.0.10.0/24):
  Route Table:
    10.0.0.0/16 → local     (VPC-internal traffic)
    0.0.0.0/0   → nat-xxxx  (outbound only via NAT)
  
  EC2 instance: 10.0.10.20 (private only)
  ↑ NOT reachable from internet inbound
```

### Creating Subnets

```bash
# Create public subnet in AZ-a
PUBLIC_SUBNET=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-az-a}]' \
  --query 'Subnet.SubnetId' --output text)

# Enable auto-assign public IP for public subnet
aws ec2 modify-subnet-attribute \
  --subnet-id $PUBLIC_SUBNET \
  --map-public-ip-on-launch

# Create private subnet in AZ-a
PRIVATE_SUBNET=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.10.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-az-a}]' \
  --query 'Subnet.SubnetId' --output text)
```

---

## 5. Internet Gateway

An **Internet Gateway (IGW)** is a horizontally-scaled, redundant, highly-available VPC component that enables communication between instances in your VPC and the internet.

```
Internet
   │
   ▼
┌──────────────────┐
│ Internet Gateway │  ← The door between your VPC and internet
│ (IGW)            │    - Scales automatically (no bandwidth limits)
│                  │    - Highly available (AWS-managed)
└────────┬─────────┘
         │
         ▼
      VPC
```

```bash
# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=my-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)

# Attach to VPC (one IGW per VPC)
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID

# Verify
aws ec2 describe-internet-gateways --internet-gateway-ids $IGW_ID
```

---

## 6. Route Tables

A **route table** contains rules (routes) that determine where network traffic is directed.

```
Route Table: public-rt
───────────────────────
Destination      Target
10.0.0.0/16      local       ← All VPC-internal traffic stays inside VPC
0.0.0.0/0        igw-xxxx    ← Everything else goes to Internet Gateway
```

### Working with Route Tables

```bash
# Create a route table for public subnets
PUBLIC_RT=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)

# Add route: all internet traffic → IGW
aws ec2 create-route \
  --route-table-id $PUBLIC_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

# Associate public subnet with this route table
aws ec2 associate-route-table \
  --route-table-id $PUBLIC_RT \
  --subnet-id $PUBLIC_SUBNET

# Create private route table (no internet route)
PRIVATE_RT=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)

# Associate private subnet with private route table
aws ec2 associate-route-table \
  --route-table-id $PRIVATE_RT \
  --subnet-id $PRIVATE_SUBNET

# View route tables
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --output table
```

---

## 7. How Traffic Flows in a VPC

### Scenario: User Accesses a Web Server in Public Subnet

```
1. User types http://54.1.2.3 in browser
         │
         ▼
2. Request reaches Internet Gateway (IGW)
         │
         ▼
3. IGW checks route table for public subnet
   Route: 0.0.0.0/0 → igw (so traffic is admitted)
         │
         ▼
4. NACL for public subnet checks inbound rules
   Allows TCP port 80 → PASS
         │
         ▼
5. Security Group on EC2 checks inbound rules
   Allows TCP port 80 from 0.0.0.0/0 → PASS
         │
         ▼
6. Request reaches EC2 instance on 10.0.1.10
         │
         ▼
7. Response travels back:
   EC2 → Security Group (stateful, auto-allows) → NACL (check outbound) → IGW → Internet
```

---

## 8. Network ACLs (NACLs)

**NACLs** are stateless firewalls at the subnet boundary. Already compared with Security Groups in the Security Groups notes — here's the operational side.

```
NACL Rules are evaluated in NUMBERED ORDER (lowest first):

Rule 100: ALLOW TCP 80 from 0.0.0.0/0
Rule 200: ALLOW TCP 443 from 0.0.0.0/0
Rule 300: ALLOW TCP 22 from 203.0.113.0/32
Rule 400: DENY TCP 22 from 0.0.0.0/0    ← Would block, but rule 300 already allowed your IP
Rule *:   DENY ALL                        ← Default deny everything else
```

```bash
# Create a NACL
NACL_ID=$(aws ec2 create-network-acl \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=public-nacl}]' \
  --query 'NetworkAcl.NetworkAclId' --output text)

# Add inbound rule: Allow HTTP
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow \
  --ingress

# Add inbound rule: Allow ephemeral return ports (IMPORTANT for stateless!)
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 200 \
  --protocol tcp \
  --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow \
  --ingress

# Associate with subnet
aws ec2 associate-network-acl \
  --subnet-id $PUBLIC_SUBNET \
  --network-acl-id $NACL_ID
```

> **Stateless gotcha**: Because NACLs are stateless, you must allow **both** the inbound request (port 80) AND the outbound response (ephemeral ports 1024-65535). Forgetting to allow ephemeral ports is a very common mistake.

---

## 9. VPC Flow Logs

**VPC Flow Logs** capture information about IP traffic going to and from network interfaces in your VPC. Useful for:
- Security analysis
- Troubleshooting connectivity issues
- Compliance logging

```bash
# Enable flow logs to CloudWatch Logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::xxxx:role/VPCFlowLogsRole

# Flow log format:
# version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
# 2       123456789  eni-abc123   10.0.1.10  54.2.3.4  49152  80     6       10    840  1620000000 1620000060 ACCEPT OK
```

---

## 10. Custom VPC: Full Build

Complete script to build a production-ready VPC from scratch:

```bash
#!/bin/bash
# Build a custom VPC with public and private subnets

# 1. Create VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=my-custom-vpc}]' \
  --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames

# 2. Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# 3. Create subnets
PUB_SUB=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
PRV_SUB=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.10.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)

# Enable public IP for public subnet
aws ec2 modify-subnet-attribute --subnet-id $PUB_SUB --map-public-ip-on-launch

# 4. Create route tables
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
PRV_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)

# Add internet route to public RT
aws ec2 create-route --route-table-id $PUB_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

# Associate subnets with route tables
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_SUB
aws ec2 associate-route-table --route-table-id $PRV_RT --subnet-id $PRV_SUB

echo "VPC:            $VPC_ID"
echo "IGW:            $IGW_ID"
echo "Public Subnet:  $PUB_SUB"
echo "Private Subnet: $PRV_SUB"
echo "Public RT:      $PUB_RT"
echo "Private RT:     $PRV_RT"
```

---

## 11. Default VPC

Every AWS region has a **Default VPC** created automatically:
- CIDR: `172.31.0.0/16`
- Default subnets in every AZ
- Internet Gateway already attached
- Default route table with IGW route
- All instances get public IPs by default

> Good for: getting started, quick tests
> Not for: production (use a custom VPC with proper isolation)

---

## 12. Cheat Sheet

```bash
# VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16
aws ec2 describe-vpcs --filters "Name=is-default,Values=false"
aws ec2 delete-vpc --vpc-id <vpc-id>

# Subnet
aws ec2 create-subnet --vpc-id <vpc> --cidr-block 10.0.1.0/24 --availability-zone us-east-1a
aws ec2 modify-subnet-attribute --subnet-id <subnet> --map-public-ip-on-launch

# IGW
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --internet-gateway-id <igw> --vpc-id <vpc>
aws ec2 detach-internet-gateway --internet-gateway-id <igw> --vpc-id <vpc>

# Route Table
aws ec2 create-route-table --vpc-id <vpc>
aws ec2 create-route --route-table-id <rt> --destination-cidr-block 0.0.0.0/0 --gateway-id <igw>
aws ec2 associate-route-table --route-table-id <rt> --subnet-id <subnet>
```

---

## 13. Common Interview Questions

**Q: What is the difference between a public and private subnet?**
> A public subnet has a route to an Internet Gateway in its route table, so resources in it can send/receive internet traffic (if they have a public IP). A private subnet has no such route — resources can only be reached from within the VPC or via VPN/Direct Connect. Private subnet instances can access the internet via a NAT Gateway.

**Q: What is an Internet Gateway?**
> An Internet Gateway is a VPC component that enables bidirectional communication between instances in your VPC and the internet. It is horizontally scalable, highly available, and managed by AWS. It performs NAT for instances that have public IP addresses.

**Q: Why can't you SSH to an EC2 instance in a private subnet directly from the internet?**
> Because private subnets have no route to the Internet Gateway. Traffic from the internet cannot reach instances without a public IP and an IGW route. To SSH to private instances, you go through a bastion host (jump server) in a public subnet, or use AWS Systems Manager Session Manager (no bastion needed).

**Q: What is the main route table?**
> Every VPC has a main route table that is automatically assigned to new subnets unless you explicitly associate a different route table. The main route table initially only has a local route for VPC-internal traffic. Best practice: don't modify the main route table. Create custom route tables for public and private subnets.

**Q: What are AWS-reserved IP addresses in a subnet?**
> AWS reserves 5 IP addresses per subnet: .0 (network address), .1 (VPC router), .2 (AWS DNS), .3 (future use), and .255 (broadcast, not used). A /24 subnet gives 256 total IPs minus 5 reserved = 251 usable IPs.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
