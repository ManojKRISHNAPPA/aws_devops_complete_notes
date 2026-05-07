# VPC Peering — Complete Guide

## Table of Contents
1. [What is VPC Peering?](#1-what-is-vpc-peering)
2. [VPC Peering Rules and Limitations](#2-vpc-peering-rules-and-limitations)
3. [How VPC Peering Works](#3-how-vpc-peering-works)
4. [Setting Up VPC Peering](#4-setting-up-vpc-peering)
5. [Cross-Account VPC Peering](#5-cross-account-vpc-peering)
6. [Cross-Region VPC Peering](#6-cross-region-vpc-peering)
7. [VPC Peering vs Transit Gateway](#7-vpc-peering-vs-transit-gateway)
8. [Cheat Sheet](#8-cheat-sheet)
9. [Common Interview Questions](#9-common-interview-questions)

---

## 1. What is VPC Peering?

**VPC Peering** is a networking connection between two VPCs that allows you to route traffic between them using **private IPv4 or IPv6 addresses**.

It is like connecting two private networks together so they can communicate as if they were on the same network.

```
Without VPC Peering:
──────────────────────
VPC A (10.0.0.0/16)          VPC B (172.16.0.0/16)
   EC2: 10.0.1.5                 RDS: 172.16.1.10
        │                               │
   Cannot reach                         │
   each other directly                  │

With VPC Peering:
──────────────────────
VPC A (10.0.0.0/16) ◄──────────────────► VPC B (172.16.0.0/16)
   EC2: 10.0.1.5        VPC Peering         RDS: 172.16.1.10
        │                Connection               │
        └──── 172.16.1.10 is now reachable ───────┘
```

**Common use cases:**
- Connect production VPC to shared-services VPC (monitoring, logging)
- Connect application VPC to database VPC
- Connect VPCs across AWS accounts (multi-account architecture)
- Connect VPCs across regions (inter-region peering)

---

## 2. VPC Peering Rules and Limitations

### What VPC Peering CANNOT Do

```
❌ NO Transitive Routing:
────────────────────────
VPC A ◄──── Peered ────► VPC B ◄──── Peered ────► VPC C

VPC A CANNOT reach VPC C through VPC B!
Each connection must be direct.

To connect A↔B, B↔C, and A↔C:
You need 3 separate peering connections.
```

### Hard Limitations

| Limitation | Detail |
|------------|--------|
| **No overlapping CIDRs** | VPC A (10.0.0.0/16) and VPC B (10.0.0.0/16) CANNOT be peered |
| **No transitive routing** | Cannot route through a peered VPC to reach a third VPC |
| **No edge-to-edge routing** | Cannot use peering to access VPN, Direct Connect, or IGW of the peered VPC |
| **One peering per pair** | Can have only one peering connection between the same two VPCs |

### What You CAN Do

| Feature | Detail |
|---------|--------|
| Same region | Peering within the same region (intra-region) |
| Cross-region | Peering across AWS regions |
| Cross-account | Peering between different AWS accounts |
| Multiple peerings | One VPC can peer with up to 125 others |

---

## 3. How VPC Peering Works

VPC Peering is not a gateway or VPN. Traffic flows through **AWS's private network** — not the internet. It is low-latency and high-bandwidth.

After creating a peering connection, you must:
1. **Accept** the peering request (if cross-account or cross-region)
2. **Update route tables** in BOTH VPCs
3. **Update security groups** to allow traffic from the other VPC's CIDR

```
Setup steps visualization:
───────────────────────────
Step 1: Create peering connection (A requests, B accepts)
        VPC A ──[pcx-xxxx]──► VPC B

Step 2: Add routes in VPC A's route table
        Destination: 172.16.0.0/16 (VPC B CIDR)
        Target: pcx-xxxx (peering connection)

Step 3: Add routes in VPC B's route table
        Destination: 10.0.0.0/16 (VPC A CIDR)
        Target: pcx-xxxx (peering connection)

Step 4: Update security groups to allow traffic
        VPC B security group:
          Inbound: TCP 3306 from 10.0.0.0/16 (VPC A CIDR)
```

---

## 4. Setting Up VPC Peering

### Same-Account, Same-Region Peering

```bash
# === SETUP ===
# VPC A (Requester)
VPC_A_CIDR="10.0.0.0/16"
VPC_B_CIDR="172.16.0.0/16"

# Create VPC A
VPC_A=$(aws ec2 create-vpc --cidr-block $VPC_A_CIDR --query 'Vpc.VpcId' --output text)
aws ec2 create-tags --resources $VPC_A --tags Key=Name,Value=vpc-a

# Create subnet in VPC A
SUBNET_A=$(aws ec2 create-subnet --vpc-id $VPC_A --cidr-block 10.0.1.0/24 --query 'Subnet.SubnetId' --output text)

# Create VPC B
VPC_B=$(aws ec2 create-vpc --cidr-block $VPC_B_CIDR --query 'Vpc.VpcId' --output text)
aws ec2 create-tags --resources $VPC_B --tags Key=Name,Value=vpc-b

# Create subnet in VPC B
SUBNET_B=$(aws ec2 create-subnet --vpc-id $VPC_B --cidr-block 172.16.1.0/24 --query 'Subnet.SubnetId' --output text)

echo "VPC A: $VPC_A"
echo "VPC B: $VPC_B"

# === PEERING ===

# Step 1: Create the peering connection (VPC A requests)
PEER_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $VPC_A \
  --peer-vpc-id $VPC_B \
  --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=vpc-a-to-vpc-b}]' \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
  --output text)

echo "Peering Connection: $PEER_ID"

# Step 2: Accept the peering connection (same account, auto-accept or manual)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PEER_ID

# Wait for it to be active
aws ec2 wait vpc-peering-connection-exists --vpc-peering-connection-ids $PEER_ID
sleep 5

# Verify state
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids $PEER_ID \
  --query 'VpcPeeringConnections[0].Status.Code' \
  --output text
# Expected: active

# === ROUTE TABLES ===

# Get route tables
RT_A=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_A" "Name=association.main,Values=true" --query 'RouteTables[0].RouteTableId' --output text)
RT_B=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_B" "Name=association.main,Values=true" --query 'RouteTables[0].RouteTableId' --output text)

# Step 3: Add route in VPC A to reach VPC B
aws ec2 create-route \
  --route-table-id $RT_A \
  --destination-cidr-block $VPC_B_CIDR \
  --vpc-peering-connection-id $PEER_ID

# Step 4: Add route in VPC B to reach VPC A
aws ec2 create-route \
  --route-table-id $RT_B \
  --destination-cidr-block $VPC_A_CIDR \
  --vpc-peering-connection-id $PEER_ID

echo "Routes added to both VPCs!"

# === SECURITY GROUPS ===

# Create SG in VPC B that allows traffic from VPC A's CIDR
SG_B=$(aws ec2 create-security-group \
  --group-name "allow-from-vpc-a" \
  --description "Allow all traffic from VPC A" \
  --vpc-id $VPC_B \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_B \
  --protocol -1 \
  --cidr $VPC_A_CIDR

echo "Security group created in VPC B: $SG_B"
echo "Peering setup complete!"
```

### Verify Peering Works

```bash
# Launch a test instance in each VPC
# Launch in VPC A
INST_A=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --subnet-id $SUBNET_A \
  --key-name my-learning-key \
  --associate-public-ip-address \
  --query 'Instances[0].InstanceId' --output text)

# Launch in VPC B
INST_B=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --subnet-id $SUBNET_B \
  --security-group-ids $SG_B \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $INST_A $INST_B

IP_A=$(aws ec2 describe-instances --instance-ids $INST_A --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
PRIV_IP_B=$(aws ec2 describe-instances --instance-ids $INST_B --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

# Test: from Instance A, ping Instance B using its PRIVATE IP
ssh -i ~/.ssh/my-learning-key.pem ec2-user@$IP_A "ping -c 3 $PRIV_IP_B"
# If successful: peering is working!
```

---

## 5. Cross-Account VPC Peering

```bash
# Account A (Requester):
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-aaaaaaa \
  --peer-vpc-id vpc-bbbbbbb \
  --peer-owner-id 987654321098    # ← Other account's ID

# Account B (Accepter — must run in Account B):
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-xxxxxxxxxxxxxxxxx

# Then update route tables in BOTH accounts/VPCs (same as same-account)
```

---

## 6. Cross-Region VPC Peering

```bash
# Create peering from us-east-1 to ap-south-1
PEER_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-xxxx \
  --peer-vpc-id vpc-yyyy \
  --peer-region ap-south-1 \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
  --output text)

# Accept in the destination region
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEER_ID \
  --region ap-south-1

# Update route tables in BOTH regions (using their respective region flags)
```

---

## 7. VPC Peering vs Transit Gateway

When peering gets complex, **Transit Gateway** is the better solution.

```
VPC Peering (N-to-N mesh):
───────────────────────────
With 5 VPCs, full mesh = 10 peering connections
Each VPC needs routes to all others
Complex to manage

VPC A ──── VPC B
 │ ╲      ╱ │
 │  ╲    ╱  │
 │   ╲  ╱   │
VPC E ── VPC C
 │     ╲╱   │
 │     ╱╲   │
VPC D ────── (etc)

Transit Gateway (hub-and-spoke):
──────────────────────────────────
Each VPC connects once to TGW
TGW routes between all VPCs
Much simpler to manage

VPC A ──┐
VPC B ──┤
VPC C ──┼──► Transit Gateway ──► Routes all traffic
VPC D ──┤
VPC E ──┘
```

| Use Case | Recommendation |
|----------|----------------|
| 2-3 VPCs, simple topology | VPC Peering |
| Many VPCs, complex mesh | Transit Gateway |
| Cross-account 1-to-1 | VPC Peering |
| Hub-and-spoke, shared services | Transit Gateway |

---

## 8. Cheat Sheet

```bash
# Create peering
aws ec2 create-vpc-peering-connection --vpc-id <vpc-a> --peer-vpc-id <vpc-b>

# Cross-account peering
aws ec2 create-vpc-peering-connection --vpc-id <vpc-a> --peer-vpc-id <vpc-b> --peer-owner-id <account-id>

# Cross-region peering
aws ec2 create-vpc-peering-connection --vpc-id <vpc-a> --peer-vpc-id <vpc-b> --peer-region <region>

# Accept peering
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id <pcx-id>

# Add route (do in BOTH VPCs)
aws ec2 create-route --route-table-id <rt-id> --destination-cidr-block <other-vpc-cidr> --vpc-peering-connection-id <pcx-id>

# List peering connections
aws ec2 describe-vpc-peering-connections --output table

# Delete peering
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id <pcx-id>
```

---

## 9. Common Interview Questions

**Q: What is VPC peering?**
> VPC peering is a networking connection between two VPCs that enables routing traffic between them using private IP addresses. Traffic travels over AWS's private network, not the internet. It can be within an account, cross-account, and cross-region.

**Q: Does VPC peering support transitive routing?**
> No. VPC peering is non-transitive. If VPC A is peered with VPC B, and VPC B is peered with VPC C, VPC A cannot reach VPC C through VPC B — they need a direct peering connection. For hub-and-spoke architectures with many VPCs, use Transit Gateway instead.

**Q: What are the prerequisites for VPC peering?**
> 1. The VPCs must not have overlapping CIDR blocks
> 2. Both parties must accept the peering connection (in cross-account scenarios)
> 3. Route tables in BOTH VPCs must be updated to route traffic to the other VPC via the peering connection
> 4. Security groups must be updated to allow traffic from the other VPC's CIDR

**Q: When would you use Transit Gateway instead of VPC peering?**
> When you have many VPCs (5+) that need to communicate. VPC peering requires N*(N-1)/2 connections for a full mesh — 10 connections for 5 VPCs, 45 for 10 VPCs. Transit Gateway uses a hub-and-spoke model where each VPC connects once, and the TGW routes between them. It also supports on-premises connectivity via VPN or Direct Connect.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
