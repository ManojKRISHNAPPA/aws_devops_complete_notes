# VPC and Networking — Practical Assignments

## Assignment 1 — Build a Custom VPC from Scratch

### Objective
Create a complete, production-style VPC with public/private subnets across 2 AZs.

```bash
#!/bin/bash
# Complete Custom VPC Build

# 1. Create VPC
VPC=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=custom-vpc}]' \
  --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id $VPC --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC --enable-dns-support
echo "VPC: $VPC"

# 2. Create Internet Gateway
IGW=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=custom-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC
echo "IGW: $IGW"

# 3. Create subnets in 2 AZs
PUB_A=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-az-a}]' \
  --query 'Subnet.SubnetId' --output text)
aws ec2 modify-subnet-attribute --subnet-id $PUB_A --map-public-ip-on-launch

PUB_B=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-az-b}]' \
  --query 'Subnet.SubnetId' --output text)
aws ec2 modify-subnet-attribute --subnet-id $PUB_B --map-public-ip-on-launch

PRV_A=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.10.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-az-a}]' \
  --query 'Subnet.SubnetId' --output text)

PRV_B=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.11.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-az-b}]' \
  --query 'Subnet.SubnetId' --output text)

# 4. Route tables
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PUB_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_A
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_B

PRV_RT=$(aws ec2 create-route-table --vpc-id $VPC \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 associate-route-table --route-table-id $PRV_RT --subnet-id $PRV_A
aws ec2 associate-route-table --route-table-id $PRV_RT --subnet-id $PRV_B

echo "========================================"
echo "Custom VPC Setup Complete!"
echo "VPC:            $VPC"
echo "IGW:            $IGW"
echo "Public AZ-a:    $PUB_A"
echo "Public AZ-b:    $PUB_B"
echo "Private AZ-a:   $PRV_A"
echo "Private AZ-b:   $PRV_B"
echo "Public RT:      $PUB_RT"
echo "Private RT:     $PRV_RT"
```

---

## Assignment 2 — Launch Instances and Test Connectivity

```bash
# Get AMI
AMI=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' --output text)

# Create security groups
PUB_SG=$(aws ec2 create-security-group --group-name pub-sg --description "Public SG" --vpc-id $VPC --query 'GroupId' --output text)
PRV_SG=$(aws ec2 create-security-group --group-name prv-sg --description "Private SG" --vpc-id $VPC --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $PUB_SG --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $PRV_SG --protocol tcp --port 22 --source-group $PUB_SG
aws ec2 authorize-security-group-ingress --group-id $PRV_SG --protocol icmp --port -1 --source-group $PUB_SG

# Launch in public subnet (bastion)
BASTION=$(aws ec2 run-instances \
  --image-id $AMI --instance-type t2.micro --key-name my-learning-key \
  --subnet-id $PUB_A --security-group-ids $PUB_SG \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=bastion}]' \
  --query 'Instances[0].InstanceId' --output text)

# Launch in private subnet
PRIVATE=$(aws ec2 run-instances \
  --image-id $AMI --instance-type t2.micro --key-name my-learning-key \
  --subnet-id $PRV_A --security-group-ids $PRV_SG \
  --no-associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=private-ec2}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $BASTION $PRIVATE

BASTION_IP=$(aws ec2 describe-instances --instance-ids $BASTION --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
PRV_IP=$(aws ec2 describe-instances --instance-ids $PRIVATE --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

echo "Bastion public IP: $BASTION_IP"
echo "Private instance IP: $PRV_IP"
echo ""
echo "Test: ssh -i ~/.ssh/my-learning-key.pem ec2-user@$BASTION_IP"
echo "From bastion: ping $PRV_IP"
echo "From bastion: ssh ec2-user@$PRV_IP"
```

---

## Interview Practice

### Scenario: VPC Design
> "Design a VPC for a web application that must be highly available and comply with PCI-DSS (cardholder data must not be on internet-accessible systems)."

**Answer**:
```
VPC: 10.0.0.0/16, us-east-1

Public subnets (AZ-a, AZ-b):  Application Load Balancer, NAT Gateways
Private app subnets (AZ-a, AZ-b): Web/App EC2 instances
Private data subnets (AZ-a, AZ-b): RDS Multi-AZ (cardholder data here)

Security:
- ALB only exposes port 443 (HTTPS)
- App SG: only allows from ALB SG
- DB SG: only allows from App SG, never from internet
- VPC Flow Logs enabled
- CloudTrail enabled
- No public IPs on app or DB instances
- WAF on ALB
```

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
