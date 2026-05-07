# Security Groups — Practical Assignments

## Table of Contents
1. [Assignment 1 — Basic Security Group Setup](#1-assignment-1--basic-security-group-setup)
2. [Assignment 2 — Test Inbound Rule Blocking](#2-assignment-2--test-inbound-rule-blocking)
3. [Assignment 3 — Three-Tier Security Architecture](#3-assignment-3--three-tier-security-architecture)
4. [Assignment 4 — Security Group Auditing](#4-assignment-4--security-group-auditing)
5. [Interview Practice Scenarios](#5-interview-practice-scenarios)

---

## 1. Assignment 1 — Basic Security Group Setup

### Objective
Create security groups for different purposes and observe how they control access.

### Steps

**Step 1: Create a Restrictive Web Server SG**
```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query 'Vpcs[0].VpcId' --output text)

# Create SG
WEB_SG=$(aws ec2 create-security-group \
  --group-name assignment-web-sg \
  --description "Web server: HTTP/HTTPS public, SSH from my IP" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)
echo "Web SG: $WEB_SG"

# Get your current IP
MY_IP=$(curl -s https://checkip.amazonaws.com)
echo "My IP: $MY_IP"

# Add rules
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 22 --cidr ${MY_IP}/32

# Verify
aws ec2 describe-security-groups --group-ids $WEB_SG --query 'SecurityGroups[0].IpPermissions' --output table
```

**Step 2: Launch EC2 with This SG and Test**
```bash
AMI_ID=$(aws ec2 describe-images --owners amazon \
  --filters "Name=name,Values=al2023-ami-*" "Name=architecture,Values=x86_64" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' --output text)

SUBNET_ID=$(aws ec2 describe-subnets --query 'Subnets[0].SubnetId' --output text)

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name my-learning-key \
  --security-group-ids $WEB_SG \
  --subnet-id $SUBNET_ID \
  --associate-public-ip-address \
  --user-data '#!/bin/bash
dnf install -y httpd
systemctl start httpd
echo "<h1>SG Test Server</h1>" > /var/www/html/index.html' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=sg-test-server}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $INSTANCE_ID

PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

echo "Instance: $PUBLIC_IP"
echo "Test: curl http://$PUBLIC_IP"
echo "Test: ssh -i ~/.ssh/my-learning-key.pem ec2-user@$PUBLIC_IP"
```

---

## 2. Assignment 2 — Test Inbound Rule Blocking

### Objective
Observe security group blocking in action by modifying rules and testing connectivity.

### Steps

**Step 1: Confirm Access**
```bash
# Test HTTP access (should work)
curl -I http://$PUBLIC_IP
# Expected: HTTP/1.1 200 OK

# Test SSH (should work)
ssh -i ~/.ssh/my-learning-key.pem ec2-user@$PUBLIC_IP "echo 'SSH works!'"
```

**Step 2: Block HTTP (Remove Port 80 Rule)**
```bash
# Remove HTTP inbound rule
aws ec2 revoke-security-group-ingress \
  --group-id $WEB_SG \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

echo "HTTP rule removed. Testing..."

# Test HTTP - should FAIL now (will timeout)
curl --connect-timeout 5 http://$PUBLIC_IP
# Expected: curl: (28) Connection timed out after 5001 milliseconds

echo "HTTP is now blocked!"
```

**Step 3: Re-Add HTTP Rule**
```bash
aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Test again (changes apply immediately)
curl -I http://$PUBLIC_IP
# Expected: HTTP/1.1 200 OK
```

**Step 4: Test SSH from Different Source**
```bash
# Restrict SSH to a different IP (not yours)
aws ec2 revoke-security-group-ingress --group-id $WEB_SG --protocol tcp --port 22 --cidr ${MY_IP}/32
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 22 --cidr 192.0.2.0/24

# Try SSH from your real IP — should fail
ssh -o ConnectTimeout=5 -i ~/.ssh/my-learning-key.pem ec2-user@$PUBLIC_IP
# Expected: Connection timed out

# Restore correct SSH rule
aws ec2 revoke-security-group-ingress --group-id $WEB_SG --protocol tcp --port 22 --cidr 192.0.2.0/24
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 22 --cidr ${MY_IP}/32
```

**Step 5: Observe Stateful Behavior**
```bash
# SSH into the instance
ssh -i ~/.ssh/my-learning-key.pem ec2-user@$PUBLIC_IP

# Now remove the OUTBOUND rule while you're connected
# (Run this from another terminal)
aws ec2 revoke-security-group-egress \
  --group-id $WEB_SG \
  --protocol -1 \
  --cidr 0.0.0.0/0 \
  --port -1

# Your existing SSH session should STILL work!
# This proves stateful behavior — active connections remain
echo "I am still connected — stateful security group!"

# New SSH connections should fail now
# (try from another terminal)

# Restore outbound rule
aws ec2 authorize-security-group-egress \
  --group-id $WEB_SG \
  --protocol -1 \
  --cidr 0.0.0.0/0 \
  --port -1
```

---

## 3. Assignment 3 — Three-Tier Security Architecture

### Objective
Build a proper three-tier security model with Web, App, and Database security groups that only allow communication between adjacent tiers.

### Steps

**Step 1: Create All Three Security Groups**
```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query 'Vpcs[0].VpcId' --output text)
MY_IP=$(curl -s https://checkip.amazonaws.com)

# Web tier SG
WEB_SG=$(aws ec2 create-security-group \
  --group-name three-tier-web \
  --description "Web tier: public HTTP/HTTPS" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

# App tier SG
APP_SG=$(aws ec2 create-security-group \
  --group-name three-tier-app \
  --description "App tier: only from web tier" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

# DB tier SG
DB_SG=$(aws ec2 create-security-group \
  --group-name three-tier-db \
  --description "DB tier: only from app tier" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

echo "Web SG: $WEB_SG"
echo "App SG: $APP_SG"
echo "DB SG:  $DB_SG"
```

**Step 2: Configure Rules**
```bash
# Web tier: public access + SSH from admin
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 22 --cidr ${MY_IP}/32

# App tier: ONLY from web tier SG (port 8080), SSH from web tier
aws ec2 authorize-security-group-ingress \
  --group-id $APP_SG \
  --ip-permissions IpProtocol=tcp,FromPort=8080,ToPort=8080,UserIdGroupPairs=[{GroupId=$WEB_SG}]
aws ec2 authorize-security-group-ingress \
  --group-id $APP_SG \
  --ip-permissions IpProtocol=tcp,FromPort=22,ToPort=22,UserIdGroupPairs=[{GroupId=$WEB_SG}]

# DB tier: ONLY from app tier SG (MySQL 3306)
aws ec2 authorize-security-group-ingress \
  --group-id $DB_SG \
  --ip-permissions IpProtocol=tcp,FromPort=3306,ToPort=3306,UserIdGroupPairs=[{GroupId=$APP_SG}]
```

**Step 3: Launch Instances in Each Tier**
```bash
AMI_ID=$(aws ec2 describe-images --owners amazon \
  --filters "Name=name,Values=al2023-ami-*" "Name=architecture,Values=x86_64" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' --output text)
SUBNET_ID=$(aws ec2 describe-subnets --query 'Subnets[0].SubnetId' --output text)

# Launch web instance
WEB_INSTANCE=$(aws ec2 run-instances --image-id $AMI_ID --instance-type t2.micro \
  --key-name my-learning-key --security-group-ids $WEB_SG --subnet-id $SUBNET_ID \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-tier}]' \
  --query 'Instances[0].InstanceId' --output text)

# Launch app instance (private - no public IP)
APP_INSTANCE=$(aws ec2 run-instances --image-id $AMI_ID --instance-type t2.micro \
  --key-name my-learning-key --security-group-ids $APP_SG --subnet-id $SUBNET_ID \
  --no-associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=app-tier}]' \
  --query 'Instances[0].InstanceId' --output text)

# Launch db instance (private)
DB_INSTANCE=$(aws ec2 run-instances --image-id $AMI_ID --instance-type t2.micro \
  --key-name my-learning-key --security-group-ids $DB_SG --subnet-id $SUBNET_ID \
  --no-associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=db-tier}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $WEB_INSTANCE $APP_INSTANCE $DB_INSTANCE
echo "All instances running!"
```

**Step 4: Verify Security Groups Work Correctly**
```bash
# Get IPs
WEB_PUBLIC=$(aws ec2 describe-instances --instance-ids $WEB_INSTANCE --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
APP_PRIVATE=$(aws ec2 describe-instances --instance-ids $APP_INSTANCE --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
DB_PRIVATE=$(aws ec2 describe-instances --instance-ids $DB_INSTANCE --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

echo "Web Public: $WEB_PUBLIC"
echo "App Private: $APP_PRIVATE"
echo "DB Private: $DB_PRIVATE"

# Test: Web tier can reach app tier on port 8080
ssh -i ~/.ssh/my-learning-key.pem ec2-user@$WEB_PUBLIC \
  "nc -zv $APP_PRIVATE 8080 2>&1 || echo 'Port 8080 check done'"

# Test: Web tier CANNOT reach DB directly on 3306
ssh -i ~/.ssh/my-learning-key.pem ec2-user@$WEB_PUBLIC \
  "timeout 3 nc -zv $DB_PRIVATE 3306 2>&1 || echo 'DB port blocked from web tier (correct!)'"
```

**Step 5: Clean Up**
```bash
aws ec2 terminate-instances --instance-ids $WEB_INSTANCE $APP_INSTANCE $DB_INSTANCE
aws ec2 wait instance-terminated --instance-ids $WEB_INSTANCE $APP_INSTANCE $DB_INSTANCE
aws ec2 delete-security-group --group-id $DB_SG
aws ec2 delete-security-group --group-id $APP_SG
aws ec2 delete-security-group --group-id $WEB_SG
```

---

## 4. Assignment 4 — Security Group Auditing

### Objective
Audit your account for overly permissive security groups (SSH open to 0.0.0.0/0).

### Steps

```bash
# Find all security groups with SSH open to the world (common misconfiguration)
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?IpPermissions[?ToPort==`22` && contains(IpRanges[].CidrIp, `0.0.0.0/0`)]].[GroupId,GroupName,Description]' \
  --output table

# Find security groups with ALL traffic from anywhere
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?IpPermissions[?IpProtocol==`-1` && IpRanges[?CidrIp==`0.0.0.0/0`]]].[GroupId,GroupName]' \
  --output table

# List all security groups with their rules as a report
aws ec2 describe-security-groups \
  --query 'SecurityGroups[*].{ID:GroupId,Name:GroupName,Rules:IpPermissions[*].{Port:ToPort,Source:IpRanges[*].CidrIp}}' \
  --output json | python3 -c "
import sys, json
groups = json.load(sys.stdin)
for g in groups:
    print(f\"\\nSG: {g['ID']} ({g['Name']})\")
    for rule in g['Rules']:
        port = rule.get('Port', 'All')
        sources = rule.get('Source', [])
        if sources:
            for src in sources:
                if src == '0.0.0.0/0':
                    marker = ' ⚠️  PUBLIC' if port in [22, 3389, 3306, 5432] else ''
                    print(f'  Port {port} ← {src}{marker}')
"
```

---

## 5. Interview Practice Scenarios

### Scenario 1: Security Review
> "During a security audit, you find an EC2 instance with SSH (port 22) open to 0.0.0.0/0. How do you fix this and prevent it in the future?"

**Answer**:
1. **Immediate fix**: Find the security group rule and revoke the 0.0.0.0/0 SSH rule. Replace with your office IP range or a bastion host security group reference
2. **Prevention options**:
   - Use AWS Config rule `restricted-ssh` to alert on or auto-remediate this
   - Use AWS Security Hub for continuous compliance monitoring
   - Use Service Control Policies (SCP) to prevent creating public SSH rules
   - Use EC2 Instance Connect or Session Manager (SSM) for SSH — eliminates port 22 entirely

### Scenario 2: Debugging Connectivity
> "A developer says they cannot connect from the web server to the database on port 5432. Both are in the same VPC. How do you debug this?"

**Answer** (systematic approach):
1. Check the **database security group** inbound rules — does port 5432 allow the web server's SG or IP?
2. Check the **web server security group** outbound rules — is port 5432 allowed? (usually all outbound is allowed by default)
3. Check **NACLs** on both subnets — are they blocking port 5432 or the ephemeral response ports?
4. Check **routing** — are both subnets in the same VPC and can they route to each other?
5. Check if the **database is actually listening** on port 5432 with `ss -tlnp | grep 5432`

### Scenario 3: RDS Access
> "Your application runs on EC2 and needs to access an RDS database. How would you configure security groups?"

**Answer**: Create two security groups:
- `app-sg`: for the EC2 instances, with outbound port 3306 allowed (or keep default all-outbound)
- `rds-sg`: for the RDS instance, with inbound port 3306 sourced from `app-sg`

Never allow 0.0.0.0/0 on the database security group. The SG reference ensures only your application EC2 instances can reach the database, regardless of IP address changes.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
