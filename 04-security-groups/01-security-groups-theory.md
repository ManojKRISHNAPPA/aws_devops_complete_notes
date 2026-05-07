# Security Groups — Complete Guide

## Table of Contents
1. [What is a Security Group?](#1-what-is-a-security-group)
2. [Security Group Rules](#2-security-group-rules)
3. [Stateful Behavior](#3-stateful-behavior)
4. [Security Groups vs NACLs](#4-security-groups-vs-nacls)
5. [Creating and Managing Security Groups](#5-creating-and-managing-security-groups)
6. [Security Group Referencing](#6-security-group-referencing)
7. [Default Security Group](#7-default-security-group)
8. [Common Security Group Patterns](#8-common-security-group-patterns)
9. [Security Group Best Practices](#9-security-group-best-practices)
10. [Cheat Sheet](#10-cheat-sheet)
11. [Common Interview Questions](#11-common-interview-questions)

---

## 1. What is a Security Group?

A **Security Group** is a **virtual firewall** that controls inbound and outbound traffic to AWS resources (EC2 instances, RDS, Lambda, etc.).

```
Internet
   │
   ▼
┌─────────────────────────────────────────┐
│          Security Group                 │  ← Acts like a firewall
│   ┌─────────────────────────────┐       │
│   │ Inbound Rules               │       │
│   │  Port 22  ← SSH from 1.2.3.4│       │
│   │  Port 80  ← HTTP from any   │       │
│   │  Port 443 ← HTTPS from any  │       │
│   └─────────────────────────────┘       │
│                                         │
│   ┌─────────────────────────────┐       │
│   │ Outbound Rules              │       │
│   │  All traffic → Anywhere     │       │
│   └─────────────────────────────┘       │
│                                         │
│   ┌─────────────────┐                   │
│   │  EC2 Instance   │                   │
│   └─────────────────┘                   │
└─────────────────────────────────────────┘
```

**Key characteristics:**
- Applied at the **instance level** (not subnet level)
- An instance can have **multiple security groups** (rules are aggregated — most permissive wins)
- A security group can be applied to **multiple instances**
- **Allow rules only** — you cannot explicitly deny traffic in a security group
- **Stateful** — if you allow inbound traffic, the response is automatically allowed
- Changes take effect **immediately**
- Security groups are **VPC-scoped** — cannot be used across VPCs

---

## 2. Security Group Rules

### Inbound Rules
Control traffic coming **into** your instance.

| Field | Description |
|-------|-------------|
| **Type** | Protocol shortcut (SSH, HTTP, HTTPS, etc.) |
| **Protocol** | TCP, UDP, ICMP, or All |
| **Port range** | Single port or range (e.g., 3000-3100) |
| **Source** | IP address, CIDR block, or another security group |

### Outbound Rules
Control traffic going **out** of your instance. Default: all outbound traffic is allowed.

### Rule Examples

```
Inbound Rules for a Web Server:
┌────────┬──────────┬───────┬─────────────────────────┐
│ Type   │ Protocol │ Port  │ Source                  │
├────────┼──────────┼───────┼─────────────────────────┤
│ SSH    │ TCP      │ 22    │ 203.0.113.0/32 (my IP)  │
│ HTTP   │ TCP      │ 80    │ 0.0.0.0/0 (all IPv4)    │
│ HTTPS  │ TCP      │ 443   │ 0.0.0.0/0 (all IPv4)    │
│ HTTP   │ TCP      │ 80    │ ::/0 (all IPv6)          │
└────────┴──────────┴───────┴─────────────────────────┘

Outbound Rules (default):
┌──────────┬──────────┬───────┬──────────────────────┐
│ Type     │ Protocol │ Port  │ Destination          │
├──────────┼──────────┼───────┼──────────────────────┤
│ All      │ All      │ All   │ 0.0.0.0/0            │
└──────────┴──────────┴───────┴──────────────────────┘
```

### Common Port Numbers to Know

| Port | Protocol | Service |
|------|----------|---------|
| 22 | TCP | SSH |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 27017 | TCP | MongoDB |
| 6379 | TCP | Redis |
| 8080 | TCP | HTTP Alternate |
| 3389 | TCP | RDP (Windows) |
| 2049 | TCP/UDP | NFS (EFS) |
| ICMP | - | Ping |

---

## 3. Stateful Behavior

Security groups are **stateful** — this is a critical concept.

**Stateful means**: If you allow an inbound connection, the response traffic is automatically allowed OUT, even if there is no matching outbound rule. And vice versa.

```
Stateful Example:
─────────────────
User browser (1.2.3.4)          EC2 Instance
       │                              │
       │── HTTP GET port 80 ─────────►│  ← Inbound rule: port 80 allowed
       │                              │    Processes request...
       │◄── HTTP 200 response ────────│  ← Response automatically allowed
       │                              │    (no outbound rule needed for this)
       │
Even if outbound rules only allow port 443,
the response on port 80 is still permitted!
```

**Compare with NACLs (stateless)**: NACLs require explicit rules for both the request AND the response. Security groups handle this automatically.

---

## 4. Security Groups vs NACLs

This is a **very common interview question**. Know the differences thoroughly.

| Feature | Security Group | NACL |
|---------|---------------|------|
| **Level** | Instance level | Subnet level |
| **State** | **Stateful** | **Stateless** |
| **Rules** | Allow rules only | Allow AND Deny rules |
| **Rule evaluation** | All rules evaluated | Rules evaluated in order (lowest number first) |
| **Default inbound** | All denied (implicit) | All allowed |
| **Default outbound** | All allowed | All allowed |
| **Association** | Multiple SGs per instance | One NACL per subnet |
| **Use case** | Primary security layer | Additional defense-in-depth |

```
Defense in Depth:
─────────────────
Internet
   │
   ▼
┌──────────────┐
│   NACL       │ ← First line of defense (subnet level)
│   (Stateless)│   Blocks specific IPs/ports before they enter subnet
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Security    │ ← Second line of defense (instance level)
│  Group       │   Fine-grained control per instance
│  (Stateful)  │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  EC2 Instance│
└──────────────┘
```

> **Best practice**: Use Security Groups as your primary layer. Add NACLs only when you need to explicitly deny known bad IPs or for compliance requirements.

---

## 5. Creating and Managing Security Groups

### Via Console
1. Go to **EC2 → Security Groups → Create security group**
2. Enter name, description, VPC
3. Add inbound rules
4. Add outbound rules (usually leave default: all outbound)
5. Click Create

### Via CLI

```bash
# Create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name web-server-sg \
  --description "Security group for web servers" \
  --vpc-id vpc-xxxxxxxxxxxxxxxxx \
  --query 'GroupId' \
  --output text)

echo "Created security group: $SG_ID"

# Add inbound rules
# Allow SSH from a specific IP
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.10/32

# Allow HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow HTTPS from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Allow ICMP (ping) for troubleshooting
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol icmp \
  --port -1 \
  --cidr 0.0.0.0/0

# View the security group
aws ec2 describe-security-groups \
  --group-ids $SG_ID \
  --output json

# Remove a rule
aws ec2 revoke-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.10/32

# Delete security group (must not be attached to any instance)
aws ec2 delete-security-group --group-id $SG_ID
```

### Attach Security Groups to EC2

```bash
# At launch time
aws ec2 run-instances \
  --security-group-ids sg-xxxx sg-yyyy \   # Multiple SGs
  ...

# Modify after launch
aws ec2 modify-instance-attribute \
  --instance-id i-xxxx \
  --groups sg-xxxx sg-yyyy    # Replaces ALL existing SGs
```

---

## 6. Security Group Referencing

One of the most powerful features — a security group rule can reference **another security group** as the source, instead of an IP range.

### Why This Matters

```
Three-tier architecture:
───────────────────────
Web Tier SG ──► App Tier SG ──► DB Tier SG

DB security group rule:
  Inbound: Port 3306 from [App Tier SG]
  
This means ONLY EC2 instances in App Tier SG can reach the database.
Any IP address trying to connect directly will be blocked.
This is much better than trying to maintain a list of IP addresses!
```

### Example: Three-Tier Setup

```bash
# Create 3 security groups
WEB_SG=$(aws ec2 create-security-group --group-name web-sg --description "Web Tier" --vpc-id $VPC_ID --query 'GroupId' --output text)
APP_SG=$(aws ec2 create-security-group --group-name app-sg --description "App Tier" --vpc-id $VPC_ID --query 'GroupId' --output text)
DB_SG=$(aws ec2 create-security-group --group-name db-sg --description "DB Tier" --vpc-id $VPC_ID --query 'GroupId' --output text)

# Web tier: public HTTP/HTTPS
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0

# App tier: only from web tier SG
aws ec2 authorize-security-group-ingress \
  --group-id $APP_SG \
  --protocol tcp \
  --port 8080 \
  --source-group $WEB_SG

# DB tier: only from app tier SG
aws ec2 authorize-security-group-ingress \
  --group-id $DB_SG \
  --protocol tcp \
  --port 3306 \
  --source-group $APP_SG
```

---

## 7. Default Security Group

Every VPC comes with a **default security group**. Key behaviors:

- Inbound: Allow from same security group (instances in the same SG can communicate)
- Outbound: Allow all traffic
- Cannot be deleted, but can be modified

> **Best practice**: Do not use the default security group for anything. Create purpose-specific security groups with minimal permissions.

---

## 8. Common Security Group Patterns

### Web Server Pattern

```
web-server-sg:
  Inbound:
    TCP 80   ← 0.0.0.0/0    (HTTP public)
    TCP 443  ← 0.0.0.0/0    (HTTPS public)
    TCP 22   ← <your-ip>/32  (SSH admin only)
  Outbound:
    All → 0.0.0.0/0         (default - allow all)
```

### Application Server (Private) Pattern

```
app-server-sg:
  Inbound:
    TCP 8080 ← web-server-sg (only from web tier)
    TCP 22   ← bastion-sg    (SSH from bastion host only)
  Outbound:
    TCP 3306 → db-sg         (MySQL to database)
    TCP 443  → 0.0.0.0/0    (HTTPS for external APIs)
```

### Database Pattern

```
database-sg:
  Inbound:
    TCP 3306 ← app-server-sg (MySQL from app only)
  Outbound:
    All → 0.0.0.0/0         (allow updates/patches)
```

### Bastion Host Pattern

```
bastion-sg:
  Inbound:
    TCP 22 ← <your-office-ip>/32 (SSH from office only)
  Outbound:
    TCP 22 → private-instances-sg  (SSH to private instances)
```

---

## 9. Security Group Best Practices

| Practice | Why |
|----------|-----|
| **Least privilege** | Only allow ports that are actually needed |
| **Use SG references** | Prefer SG source over CIDR for internal traffic |
| **Named security groups** | Use descriptive names: `web-tier-sg`, `db-sg`, not `sg-1` |
| **Never 0.0.0.0/0 for SSH** | Always restrict SSH to specific IPs or bastion host |
| **No unused rules** | Audit and remove rules no longer needed |
| **Separate SGs per tier** | Web, app, and DB should have separate SGs |
| **Use tags** | Tag SGs with environment, owner, project |
| **Regular audits** | Use AWS Config or Security Hub to detect overly permissive rules |

---

## 10. Cheat Sheet

```bash
# Create
aws ec2 create-security-group --group-name <name> --description <desc> --vpc-id <vpc>

# Add inbound rule (CIDR)
aws ec2 authorize-security-group-ingress --group-id <sg> --protocol tcp --port <port> --cidr <cidr>

# Add inbound rule (another SG as source)
aws ec2 authorize-security-group-ingress --group-id <sg> --protocol tcp --port <port> --source-group <src-sg>

# Remove rule
aws ec2 revoke-security-group-ingress --group-id <sg> --protocol tcp --port <port> --cidr <cidr>

# Describe
aws ec2 describe-security-groups --group-ids <sg>
aws ec2 describe-security-groups --filters "Name=group-name,Values=<name>"

# Attach to instance
aws ec2 modify-instance-attribute --instance-id <i-id> --groups <sg1> <sg2>

# Delete
aws ec2 delete-security-group --group-id <sg>
```

---

## 11. Common Interview Questions

**Q: What is the difference between a Security Group and a NACL?**
> Security Groups are at the instance level, stateful (response traffic is automatically allowed), and only support allow rules. NACLs are at the subnet level, stateless (both request and response need explicit rules), and support both allow and deny rules. Security Groups are the primary security layer; NACLs add extra defense at the subnet boundary.

**Q: What does "stateful" mean in the context of a security group?**
> Stateful means the security group tracks the state of connections. If you allow inbound traffic on port 80, the response traffic is automatically permitted outbound without needing a specific outbound rule. The connection state is remembered, so the return traffic is always allowed.

**Q: Can you deny traffic with a security group?**
> No. Security groups only support allow rules. All traffic not explicitly allowed is implicitly denied. If you need to explicitly deny specific IPs or ranges (e.g., blocking a malicious IP), you need to use a NACL, which supports both allow and deny rules with numbered priorities.

**Q: What happens when you add multiple security groups to an EC2 instance?**
> The rules from all security groups are aggregated, and the most permissive rule wins. If SG1 blocks port 8080 and SG2 allows port 8080, port 8080 will be accessible. You cannot use security groups to override/restrict what another security group allows on the same instance.

**Q: How do security group references (source group) work?**
> Instead of specifying an IP range as the source, you can specify another security group. This means only resources associated with that source security group can access the target. This is more maintainable than IP lists — when instances are replaced, their IPs change, but their security group membership stays the same.

**Q: What is the default outbound rule for a new security group?**
> By default, a new security group allows all outbound traffic (0.0.0.0/0 on all protocols). Many teams restrict outbound traffic to only necessary ports for defense-in-depth (e.g., only allow 443 for HTTPS, 3306 for MySQL), but the permissive default is fine for most use cases.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
