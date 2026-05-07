# NAT Gateway — Practical Assignments

## Assignment 1 — Full NAT Gateway Setup

### Objective
Set up a NAT Gateway and prove that private instances can reach the internet but are not reachable from it.

```bash
# Assuming you have: $VPC, $PUB_A, $PRV_A, $PRV_RT from VPC assignment

# 1. Allocate Elastic IP
EIP=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
echo "EIP: $EIP"

# 2. Create NAT Gateway in PUBLIC subnet
NAT=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_A \
  --allocation-id $EIP \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=my-nat-gw}]' \
  --query 'NatGateway.NatGatewayId' --output text)

echo "NAT Gateway: $NAT"
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT
echo "NAT Gateway is ready!"

# 3. Add route in private subnet's route table
aws ec2 create-route \
  --route-table-id $PRV_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT

# 4. Launch private instance
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query 'Parameter.Value' --output text)

PRV_SG=$(aws ec2 create-security-group --group-name nat-test-sg --description "NAT test" --vpc-id $VPC --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $PRV_SG --protocol tcp --port 22 --cidr 10.0.0.0/16

PRIVATE=$(aws ec2 run-instances \
  --image-id $AMI --instance-type t2.micro --key-name my-learning-key \
  --subnet-id $PRV_A --security-group-ids $PRV_SG \
  --no-associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=nat-test-private}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $PRIVATE
PRV_IP=$(aws ec2 describe-instances --instance-ids $PRIVATE --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
echo "Private instance: $PRV_IP"
```

### Test NAT Works

```bash
# SSH through bastion, then to private instance
ssh -A -i ~/.ssh/my-learning-key.pem ec2-user@$BASTION_IP

# From bastion, SSH to private instance
ssh ec2-user@$PRV_IP

# Test: internet access from private instance
curl https://api.ipify.org       # Shows NAT GW's Elastic IP
ping 8.8.8.8 -c 3               # Internet reachable
sudo dnf check-update            # Package updates work

# Test: private instance has no public IP
aws ec2 describe-instances --instance-ids $PRIVATE \
  --query 'Reservations[0].Instances[0].PublicIpAddress'
# Result: None (not reachable from internet inbound)
```

### Clean Up

```bash
aws ec2 terminate-instances --instance-ids $PRIVATE
aws ec2 wait instance-terminated --instance-ids $PRIVATE
aws ec2 delete-nat-gateway --nat-gateway-id $NAT

# Wait for NAT GW to be deleted before releasing EIP
sleep 60
aws ec2 release-address --allocation-id $EIP
```

---

## Interview Practice

### Scenario: Debugging NAT
> "A developer says their private subnet EC2 instance cannot download packages from the internet after you set up NAT Gateway. How do you troubleshoot?"

**Answer checklist**:
1. Verify NAT Gateway state is `available` (not `pending` or `failed`)
2. Check the private subnet's route table — does it have `0.0.0.0/0 → nat-xxxx`?
3. Verify the NAT Gateway is in a **public** subnet (subnet with IGW route)
4. Check the private instance's security group — does it allow outbound traffic?
5. Verify NAT Gateway has an Elastic IP attached
6. Test from the instance: `curl http://169.254.169.254/latest/meta-data/` (metadata should work) then `ping 8.8.8.8` (internet test)

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
