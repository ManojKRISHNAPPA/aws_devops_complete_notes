# Amazon EC2 — Practical Assignments

## Table of Contents
1. [Assignment 1 — Launch Your First EC2 Instance](#1-assignment-1--launch-your-first-ec2-instance)
2. [Assignment 2 — SSH Access and Exploration](#2-assignment-2--ssh-access-and-exploration)
3. [Assignment 3 — Bootstrap a Web Server with User Data](#3-assignment-3--bootstrap-a-web-server-with-user-data)
4. [Assignment 4 — Elastic IP and Stop/Start Behavior](#4-assignment-4--elastic-ip-and-stopstart-behavior)
5. [Assignment 5 — Launch via CLI](#5-assignment-5--launch-via-cli)
6. [Assignment 6 — Compare Instance Types](#6-assignment-6--compare-instance-types)
7. [Assignment 7 — Spot Instance Launch](#7-assignment-7--spot-instance-launch)
8. [Interview Practice Scenarios](#8-interview-practice-scenarios)

---

## 1. Assignment 1 — Launch Your First EC2 Instance

### Objective
Launch an EC2 instance through the console, understand every option, and terminate it cleanly.

### Prerequisites
- AWS account created and IAM user configured
- AWS CLI installed and configured

### Steps

**Step 1: Open EC2 Console**
1. Log in to AWS Console
2. Select region: **us-east-1** (N. Virginia)
3. Go to **EC2** → **Instances** → **Launch instances**

**Step 2: Configure the Instance**
```
Name: my-first-ec2

AMI: Amazon Linux 2023 AMI (Free tier eligible)
  → Note the AMI ID shown (looks like ami-xxxxxxxxx)

Instance type: t2.micro (Free tier eligible)
  → Compare it with t3.small in the dropdown — note the specs

Key pair:
  → Click "Create new key pair"
  → Name: my-learning-key
  → Type: RSA
  → Format: .pem (for SSH on Mac/Linux)
  → Download it and note where it's saved

Network settings:
  → VPC: default
  → Subnet: No preference
  → Auto-assign public IP: Enable
  → Security group: Create new
    → Name: my-first-sg
    → Inbound rule: SSH (port 22) → My IP

Storage:
  → 8 GiB gp3 (default is fine)

Advanced details:
  → Leave all default for now
```

**Step 3: Launch and Monitor**
```
→ Click "Launch instance"
→ Click the instance ID to view it
→ Watch the Status column: "Pending" → "Running"
→ Wait for "Status checks": 2/2 checks passed (takes 1-2 min)
```

**Step 4: Record Instance Details**
```
Note down:
□ Instance ID: i-xxxxxxxxxxxxxxxxx
□ Public IPv4 address: x.x.x.x
□ Private IPv4 address: 10.x.x.x
□ Availability Zone: us-east-1a (or b/c)
□ AMI ID used
□ Security group ID
```

**Step 5: Stop the Instance**
```
□ Select instance → Instance state → Stop instance
□ Wait for state: Stopped
□ Note: Public IP is now empty!
□ Private IP remains the same
```

**Step 6: Start the Instance**
```
□ Select instance → Instance state → Start instance
□ Note the NEW public IP (different from before!)
```

**Step 7: Clean Up**
```
□ Terminate the instance (Instance state → Terminate)
□ Verify state changes to "Terminated" → then disappears after ~1 hour
□ Delete the security group (EC2 → Security Groups → delete my-first-sg)
□ Note: Key pair stays — it's just stored metadata
```

### Expected Outcome
- You understand the full instance lifecycle
- You saw that public IP changes on stop/start
- You can launch and clean up instances

---

## 2. Assignment 2 — SSH Access and Exploration

### Objective
SSH into a running EC2 instance, explore the system, and understand the environment.

### Steps

**Step 1: Launch a Fresh Instance**
```
Same as Assignment 1, but add to User data (skip for now):
- Amazon Linux 2023
- t2.micro
- SSH security group allowing port 22 from My IP
- Download key pair
```

**Step 2: Set Key Permissions and Connect**
```bash
# Move key to safe location
mv ~/Downloads/my-learning-key.pem ~/.ssh/

# Set correct permissions (REQUIRED)
chmod 400 ~/.ssh/my-learning-key.pem

# SSH into instance (replace with your public IP)
ssh -i ~/.ssh/my-learning-key.pem ec2-user@<your-public-ip>

# You should see:
#   __|  __|_  )
#   _|  (     /   Amazon Linux 2023
#  ___|\___|___|
```

**Step 3: Explore the Instance**
```bash
# Check the hostname and OS
hostname
uname -a
cat /etc/os-release

# Check hardware specs
nproc                          # Number of CPUs
free -h                        # Memory
df -h                          # Disk space

# Check instance metadata from inside
curl http://169.254.169.254/latest/meta-data/instance-id
curl http://169.254.169.254/latest/meta-data/instance-type
curl http://169.254.169.254/latest/meta-data/local-ipv4
curl http://169.254.169.254/latest/meta-data/public-ipv4
curl http://169.254.169.254/latest/meta-data/placement/region

# Check network
ip addr show
ping 8.8.8.8         # Internet connectivity test
```

**Step 4: Install Some Software**
```bash
# Update packages
sudo dnf update -y

# Install useful tools
sudo dnf install -y htop wget curl net-tools

# Check running processes
htop    # Press q to quit

# Check open ports
ss -tlnp
```

**Step 5: Exit and Reconnect**
```bash
# Exit SSH session
exit

# Reconnect
ssh -i ~/.ssh/my-learning-key.pem ec2-user@<public-ip>

# Verify your files/changes are still there
ls -la
```

### Expected Outcome
- Confident with SSH key setup and connection
- Understand what commands to run after connecting
- Can query metadata service for instance details

---

## 3. Assignment 3 — Bootstrap a Web Server with User Data

### Objective
Launch an EC2 instance that automatically installs and starts a web server using User Data.

### Steps

**Step 1: Create the User Data Script**
```bash
#!/bin/bash
# Update the system
dnf update -y

# Install Apache web server
dnf install -y httpd

# Start Apache and enable on boot
systemctl start httpd
systemctl enable httpd

# Create a custom HTML page showing instance info
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
INSTANCE_TYPE=$(curl -s http://169.254.169.254/latest/meta-data/instance-type)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)

cat > /var/www/html/index.html << EOF
<!DOCTYPE html>
<html>
<body style="font-family:Arial; background:#f0f0f0; padding:40px;">
  <h1>Hello from AWS EC2!</h1>
  <table border="1" style="border-collapse:collapse; padding:8px;">
    <tr><td><b>Instance ID</b></td><td>$INSTANCE_ID</td></tr>
    <tr><td><b>Instance Type</b></td><td>$INSTANCE_TYPE</td></tr>
    <tr><td><b>Availability Zone</b></td><td>$AZ</td></tr>
  </table>
</body>
</html>
EOF
```

**Step 2: Launch Instance with This User Data**

Console method:
```
1. EC2 → Launch instances
2. Name: web-server-ec2
3. AMI: Amazon Linux 2023
4. Type: t2.micro
5. Security group: Allow HTTP (port 80) from Anywhere + SSH from My IP
6. Advanced details → User data → paste the script above
7. Launch
```

CLI method:
```bash
# Save script to file
cat > web-user-data.sh << 'SCRIPT'
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl start httpd
systemctl enable httpd
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
echo "<h1>Hello from $INSTANCE_ID</h1>" > /var/www/html/index.html
SCRIPT

# Get default VPC and subnet
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query 'Vpcs[0].VpcId' --output text)
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'Subnets[0].SubnetId' --output text)

# Create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name web-sg \
  --description "Web server SG" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0

# Get latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*" "Name=architecture,Values=x86_64" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)

# Launch instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name my-learning-key \
  --security-group-ids $SG_ID \
  --subnet-id $SUBNET_ID \
  --associate-public-ip-address \
  --user-data file://web-user-data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-server-ec2}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Launched: $INSTANCE_ID"

# Wait for it to be running
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# Get public IP
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "Instance running at: http://$PUBLIC_IP"
```

**Step 3: Verify the Web Server**
```bash
# Wait ~2 minutes after launch, then:
curl http://<public-ip>

# Or open in browser: http://<public-ip>

# Check user data logs if something went wrong
ssh -i ~/.ssh/my-learning-key.pem ec2-user@<public-ip>
sudo cat /var/log/cloud-init-output.log
```

### Expected Outcome
- Browser shows your custom HTML page with instance details
- Understand how user data bootstraps an instance

---

## 4. Assignment 4 — Elastic IP and Stop/Start Behavior

### Objective
Observe public IP change on stop/start, then fix it with an Elastic IP.

### Steps

**Step 1: Observe IP Change**
```bash
# Get current public IP
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=web-server-ec2" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text
# Save this: ORIGINAL_IP=x.x.x.x

# Stop the instance
aws ec2 stop-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-stopped --instance-ids $INSTANCE_ID

# Check IP after stopping
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text
# Result: None (IP is gone!)

# Start again
aws ec2 start-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# Check new IP
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text
# Different IP than before!
```

**Step 2: Allocate and Attach Elastic IP**
```bash
# Allocate an Elastic IP
EIP_ALLOC=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text)
echo "EIP Allocation ID: $EIP_ALLOC"

# Check the IP address assigned
aws ec2 describe-addresses \
  --allocation-ids $EIP_ALLOC \
  --query 'Addresses[0].PublicIp' \
  --output text

# Associate with your instance
aws ec2 associate-address \
  --instance-id $INSTANCE_ID \
  --allocation-id $EIP_ALLOC

# Verify
aws ec2 describe-addresses --allocation-ids $EIP_ALLOC --output table
```

**Step 3: Verify EIP Persists Through Stop/Start**
```bash
EIP=$(aws ec2 describe-addresses --allocation-ids $EIP_ALLOC --query 'Addresses[0].PublicIp' --output text)

# Stop instance
aws ec2 stop-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-stopped --instance-ids $INSTANCE_ID

# Elastic IP is still yours (just not associated to a running instance now)
aws ec2 describe-addresses --allocation-ids $EIP_ALLOC --output table

# Start again
aws ec2 start-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# Same IP!
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text
# Returns the SAME EIP!
curl http://$EIP   # Web server still accessible at same address
```

**Step 4: Clean Up**
```bash
# Disassociate EIP
ASSOC_ID=$(aws ec2 describe-addresses --allocation-ids $EIP_ALLOC --query 'Addresses[0].AssociationId' --output text)
aws ec2 disassociate-address --association-id $ASSOC_ID

# Release EIP (important — or you'll be charged!)
aws ec2 release-address --allocation-id $EIP_ALLOC

# Terminate instance
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
```

---

## 5. Assignment 5 — Launch via CLI

### Objective
Build a full CLI script that creates all needed resources and launches a properly configured EC2 instance.

### Full Script

```bash
#!/bin/bash
set -e    # Exit on any error

REGION="us-east-1"
KEY_NAME="cli-assignment-key"
INSTANCE_NAME="cli-ec2-assignment"

echo "=== EC2 CLI Deployment Script ==="

# 1. Create key pair
echo "[1/6] Creating key pair..."
aws ec2 create-key-pair \
  --key-name $KEY_NAME \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/${KEY_NAME}.pem
chmod 400 ~/.ssh/${KEY_NAME}.pem
echo "Key saved to: ~/.ssh/${KEY_NAME}.pem"

# 2. Get default VPC
echo "[2/6] Getting default VPC..."
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=is-default,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)
echo "VPC: $VPC_ID"

# 3. Create security group
echo "[3/6] Creating security group..."
SG_ID=$(aws ec2 create-security-group \
  --group-name "cli-assignment-sg" \
  --description "CLI Assignment Security Group" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

MY_IP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr ${MY_IP}/32
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
echo "Security Group: $SG_ID (SSH from $MY_IP, HTTP from all)"

# 4. Get latest AMI
echo "[4/6] Finding latest Amazon Linux 2023 AMI..."
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*" "Name=architecture,Values=x86_64" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)
echo "AMI: $AMI_ID"

# 5. Launch instance
echo "[5/6] Launching instance..."
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name $KEY_NAME \
  --security-group-ids $SG_ID \
  --associate-public-ip-address \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$INSTANCE_NAME}]" \
  --query 'Instances[0].InstanceId' \
  --output text)
echo "Instance ID: $INSTANCE_ID"

# 6. Wait and get IP
echo "[6/6] Waiting for instance to be running..."
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo ""
echo "=== DONE ==="
echo "Instance ID: $INSTANCE_ID"
echo "Public IP:   $PUBLIC_IP"
echo ""
echo "Connect with:"
echo "  ssh -i ~/.ssh/${KEY_NAME}.pem ec2-user@${PUBLIC_IP}"
echo ""
echo "Cleanup with:"
echo "  aws ec2 terminate-instances --instance-ids $INSTANCE_ID"
echo "  aws ec2 delete-security-group --group-id $SG_ID"
echo "  aws ec2 delete-key-pair --key-name $KEY_NAME"
```

---

## 6. Assignment 6 — Compare Instance Types

### Objective
Empirically compare CPU performance across different instance types using a compute benchmark.

### Steps

```bash
# Launch 3 instances of different types:
# 1. t2.micro  (1 vCPU, 1 GB RAM)
# 2. t3.medium (2 vCPU, 4 GB RAM)
# 3. c5.large  (2 vCPU, 4 GB RAM) ← CPU optimized

# On each instance, run:
# ── CPU Benchmark ──
time echo "scale=5000; a(1)*4" | bc -l   # Pi calculation

# ── Memory bandwidth ──
dd if=/dev/zero of=/dev/null bs=1M count=10000

# ── Write performance ──
dd if=/dev/zero of=/tmp/test bs=1M count=1000 oflag=dsync

# Record results in this table:
# Instance   | Pi Time  | Mem BW  | Disk Write
# t2.micro   |          |         |
# t3.medium  |          |         |
# c5.large   |          |         |
```

### Reflection Questions
- Which instance was fastest for CPU? Why?
- What is the difference between t3.medium and c5.large even though both have 2 vCPU?
- How does the t2 CPU credit system affect benchmark results if run multiple times?

---

## 7. Assignment 7 — Spot Instance Launch

### Objective
Launch a Spot Instance and observe the savings.

### Steps

```bash
# Check current spot prices
aws ec2 describe-spot-price-history \
  --instance-types t3.medium \
  --product-descriptions "Linux/UNIX" \
  --max-items 5 \
  --output table

# Launch a spot instance
SPOT_REQUEST=$(aws ec2 request-spot-instances \
  --spot-price "0.02" \
  --instance-count 1 \
  --type "one-time" \
  --launch-specification '{
    "ImageId": "ami-xxxxxxxxx",
    "InstanceType": "t3.medium",
    "KeyName": "my-learning-key",
    "SecurityGroupIds": ["sg-xxxxxxxxx"]
  }' \
  --query 'SpotInstanceRequests[0].SpotInstanceRequestId' \
  --output text)

echo "Spot Request: $SPOT_REQUEST"

# Check status
aws ec2 describe-spot-instance-requests \
  --spot-instance-request-ids $SPOT_REQUEST \
  --query 'SpotInstanceRequests[0].[Status.Code,InstanceId]' \
  --output table

# Cancel spot request
aws ec2 cancel-spot-instance-requests \
  --spot-instance-request-ids $SPOT_REQUEST
```

---

## 8. Interview Practice Scenarios

### Scenario 1: Public IP Problem
> "Your team deployed a web server on EC2. Everything works fine until Monday morning — the site is down. Investigation shows the server is running but no one can reach it. What happened and how do you fix it permanently?"

**Answer**: The instance was stopped over the weekend (to save costs) and restarted Monday. The public IP changed. Fix: assign an Elastic IP to ensure a stable address. Or better: put the server behind a load balancer with a DNS name pointing to it.

### Scenario 2: Cost Optimization
> "You have 50 EC2 instances running 24/7 for the past year on On-Demand pricing. How would you reduce costs without changing the architecture?"

**Answer**: Purchase 1-year or 3-year Reserved Instances or Savings Plans for these steady-state workloads. This gives 40-72% savings with no architectural changes. For any batch workloads or dev/test, convert to Spot.

### Scenario 3: Failed Bootstrapping
> "Your EC2 instance launched with user data to install your app, but the app is not running. How do you debug this?"

**Answer**:
1. SSH into the instance
2. Check `/var/log/cloud-init-output.log` for script errors
3. Check if the package repository was reachable during launch
4. Try running the user data script manually line by line
5. Add `set -x` and error logging to the script
6. Verify security group allows outbound internet for package downloads

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
