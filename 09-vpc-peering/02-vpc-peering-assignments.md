# VPC Peering — Practical Assignments

## Assignment 1 — Set Up VPC Peering Between Two VPCs

```bash
#!/bin/bash
# Full VPC peering assignment

# Create VPC A (10.0.0.0/16)
VPC_A=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)
aws ec2 create-tags --resources $VPC_A --tags Key=Name,Value=vpc-a
SUBNET_A=$(aws ec2 create-subnet --vpc-id $VPC_A --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
IGW_A=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_A --vpc-id $VPC_A
RT_A=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_A" --query 'RouteTables[0].RouteTableId' --output text)
aws ec2 create-route --route-table-id $RT_A --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_A
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_A --map-public-ip-on-launch

# Create VPC B (172.16.0.0/16)
VPC_B=$(aws ec2 create-vpc --cidr-block 172.16.0.0/16 --query 'Vpc.VpcId' --output text)
aws ec2 create-tags --resources $VPC_B --tags Key=Name,Value=vpc-b
SUBNET_B=$(aws ec2 create-subnet --vpc-id $VPC_B --cidr-block 172.16.1.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
RT_B=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_B" --query 'RouteTables[0].RouteTableId' --output text)

echo "VPC A: $VPC_A (10.0.0.0/16)"
echo "VPC B: $VPC_B (172.16.0.0/16)"

# Create peering connection
PEER=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $VPC_A \
  --peer-vpc-id $VPC_B \
  --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=a-to-b-peering}]' \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

echo "Peering: $PEER"

# Accept (same account)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PEER
sleep 5

# Add routes in BOTH VPCs
aws ec2 create-route --route-table-id $RT_A --destination-cidr-block 172.16.0.0/16 --vpc-peering-connection-id $PEER
aws ec2 create-route --route-table-id $RT_B --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id $PEER
echo "Routes added!"

# Create security groups
SG_A=$(aws ec2 create-security-group --group-name sg-a --description "VPC A SG" --vpc-id $VPC_A --query 'GroupId' --output text)
SG_B=$(aws ec2 create-security-group --group-name sg-b --description "VPC B SG" --vpc-id $VPC_B --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_A --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_A --protocol icmp --port -1 --cidr 172.16.0.0/16
aws ec2 authorize-security-group-ingress --group-id $SG_B --protocol icmp --port -1 --cidr 10.0.0.0/16
aws ec2 authorize-security-group-ingress --group-id $SG_B --protocol tcp --port 22 --cidr 10.0.0.0/16

# Launch instances
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query 'Parameter.Value' --output text)

INST_A=$(aws ec2 run-instances --image-id $AMI --instance-type t2.micro --key-name my-learning-key \
  --subnet-id $SUBNET_A --security-group-ids $SG_A --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=instance-vpc-a}]' \
  --query 'Instances[0].InstanceId' --output text)

INST_B=$(aws ec2 run-instances --image-id $AMI --instance-type t2.micro --key-name my-learning-key \
  --subnet-id $SUBNET_B --security-group-ids $SG_B --no-associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=instance-vpc-b}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $INST_A $INST_B

PUB_A=$(aws ec2 describe-instances --instance-ids $INST_A --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
PRV_B=$(aws ec2 describe-instances --instance-ids $INST_B --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

echo ""
echo "=== TEST PEERING ==="
echo "SSH to instance A: ssh -i ~/.ssh/my-learning-key.pem ec2-user@$PUB_A"
echo "From instance A, ping VPC B instance: ping -c 3 $PRV_B"
```

### Verify Peering

```bash
# From Instance A (VPC A), test connectivity to Instance B (VPC B)
ssh -i ~/.ssh/my-learning-key.pem ec2-user@$PUB_A << EOF
ping -c 3 $PRV_B     # Should succeed across VPCs!
traceroute $PRV_B    # Shows private routing path
EOF
```

---

## Assignment 2 — Prove Transitive Routing Does NOT Work

```bash
# Create VPC C (192.168.0.0/16)
VPC_C=$(aws ec2 create-vpc --cidr-block 192.168.0.0/16 --query 'Vpc.VpcId' --output text)
SUBNET_C=$(aws ec2 create-subnet --vpc-id $VPC_C --cidr-block 192.168.1.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
RT_C=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_C" --query 'RouteTables[0].RouteTableId' --output text)

# Peer B ↔ C
PEER_BC=$(aws ec2 create-vpc-peering-connection --vpc-id $VPC_B --peer-vpc-id $VPC_C --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PEER_BC
aws ec2 create-route --route-table-id $RT_B --destination-cidr-block 192.168.0.0/16 --vpc-peering-connection-id $PEER_BC
aws ec2 create-route --route-table-id $RT_C --destination-cidr-block 172.16.0.0/16 --vpc-peering-connection-id $PEER_BC

# Now: A ↔ B ↔ C exist, but A ↔ C does NOT
# Prove A cannot reach C (transitive routing blocked):
# From Instance A, try: ping <Instance-C-private-IP>
# Result: TIMEOUT — proves non-transitive nature of VPC peering
```

---

## Interview Practice

### Scenario: Overlapping CIDRs
> "Your company acquired another company. You need to connect their AWS VPC (10.0.0.0/16) with your VPC (also 10.0.0.0/16). How do you solve the overlapping CIDR problem?"

**Answer**: VPC peering requires non-overlapping CIDRs — you cannot peer these two VPCs directly. Options:
1. **Re-CIDR one VPC** (expensive, requires migrating instances, changing configurations — major effort)
2. **Use AWS PrivateLink** — expose specific services (not the whole VPC) via endpoint services
3. **Use a proxy layer** — deploy proxy instances in both VPCs that translate requests
4. **Long-term**: establish a new CIDR range policy across the merged company and plan migrations

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
