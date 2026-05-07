# AMI — Practical Assignments

## Table of Contents
1. [Assignment 1 — Create a Golden AMI](#1-assignment-1--create-a-golden-ami)
2. [Assignment 2 — Launch Multiple Instances from AMI](#2-assignment-2--launch-multiple-instances-from-ami)
3. [Assignment 3 — Cross-Region AMI Copy](#3-assignment-3--cross-region-ami-copy)
4. [Assignment 4 — AMI Cleanup Script](#4-assignment-4--ami-cleanup-script)
5. [Interview Practice Scenarios](#5-interview-practice-scenarios)

---

## 1. Assignment 1 — Create a Golden AMI

### Objective
Configure an EC2 instance with a web server and custom settings, then capture it as a reusable AMI.

### Steps

```bash
# 1. Launch a base instance
AMI_BASE=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' --output text)

BASE_ID=$(aws ec2 run-instances \
  --image-id $AMI_BASE \
  --instance-type t2.micro \
  --key-name my-learning-key \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=golden-image-builder}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $BASE_ID
BASE_IP=$(aws ec2 describe-instances --instance-ids $BASE_ID --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

# 2. Configure the instance
ssh -i ~/.ssh/my-learning-key.pem ec2-user@$BASE_IP << 'REMOTE'
  # Update system
  sudo dnf update -y

  # Install applications
  sudo dnf install -y nginx python3 git htop

  # Start and enable nginx
  sudo systemctl enable nginx
  sudo systemctl start nginx

  # Create custom welcome page
  sudo bash -c 'cat > /usr/share/nginx/html/index.html << EOF
<html><body>
<h1>Golden Image Instance</h1>
<p>Pre-configured with: nginx, python3, git, htop</p>
</body></html>
EOF'

  # Add a helper script
  sudo bash -c 'cat > /usr/local/bin/instance-info << EOF
#!/bin/bash
echo "Instance ID: \$(curl -s http://169.254.169.254/latest/meta-data/instance-id)"
echo "Instance Type: \$(curl -s http://169.254.169.254/latest/meta-data/instance-type)"
echo "AZ: \$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)"
EOF'
  sudo chmod +x /usr/local/bin/instance-info

  # Clean up before imaging
  sudo dnf clean all
  sudo rm -rf /tmp/* /var/tmp/*
  sudo find /var/log -type f -exec truncate -s 0 {} \;
  history -c && history -w

  echo "Configuration complete. Ready for AMI creation."
REMOTE

# 3. Stop instance for clean snapshot
aws ec2 stop-instances --instance-ids $BASE_ID
aws ec2 wait instance-stopped --instance-ids $BASE_ID

# 4. Create the AMI
MY_AMI=$(aws ec2 create-image \
  --instance-id $BASE_ID \
  --name "golden-webserver-$(date +%Y-%m-%d)" \
  --description "Golden image: nginx, python3, git, htop" \
  --no-reboot \
  --tag-specifications 'ResourceType=image,Tags=[
    {Key=Name,Value=golden-webserver},
    {Key=Version,Value=1.0},
    {Key=OS,Value=AmazonLinux2023}
  ]' \
  --query 'ImageId' --output text)

echo "AMI: $MY_AMI"
aws ec2 wait image-available --image-ids $MY_AMI
echo "AMI is available!"

# 5. Terminate builder instance
aws ec2 terminate-instances --instance-ids $BASE_ID
```

---

## 2. Assignment 2 — Launch Multiple Instances from AMI

```bash
# Launch 3 instances from your golden AMI - all pre-configured!
for i in 1 2 3; do
  ID=$(aws ec2 run-instances \
    --image-id $MY_AMI \
    --instance-type t2.micro \
    --key-name my-learning-key \
    --associate-public-ip-address \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=web-server-$i}]" \
    --query 'Instances[0].InstanceId' --output text)
  echo "Launched web-server-$i: $ID"
done

# Wait for all and verify each has nginx running immediately (no bootstrapping!)
sleep 60
for ID in $(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=web-server-*" "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[0].PublicIpAddress' --output text); do
  echo -n "Testing http://$ID ... "
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 http://$ID)
  echo "HTTP $STATUS"
done
```

---

## 3. Assignment 3 — Cross-Region AMI Copy

```bash
# Copy your AMI to ap-south-1 (Mumbai)
MUMBAI_AMI=$(aws ec2 copy-image \
  --source-region us-east-1 \
  --source-image-id $MY_AMI \
  --name "golden-webserver-mumbai-$(date +%Y-%m-%d)" \
  --description "Copy of golden image for Mumbai region" \
  --region ap-south-1 \
  --query 'ImageId' --output text)

echo "Mumbai AMI: $MUMBAI_AMI"

# Wait for it in the destination region
aws ec2 wait image-available --image-ids $MUMBAI_AMI --region ap-south-1
echo "AMI available in Mumbai!"

# Launch from Mumbai AMI
aws ec2 run-instances \
  --image-id $MUMBAI_AMI \
  --instance-type t2.micro \
  --key-name my-learning-key \
  --region ap-south-1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=mumbai-web-server}]' \
  --query 'Instances[0].InstanceId' --output text
```

---

## 4. Assignment 4 — AMI Cleanup Script

```bash
#!/bin/bash
# Find and delete AMIs older than 30 days
CUTOFF_DATE=$(date -d "30 days ago" +%Y-%m-%dT%H:%M:%S)

OLD_AMIS=$(aws ec2 describe-images \
  --owners self \
  --query "Images[?CreationDate<'$CUTOFF_DATE'].[ImageId,Name,CreationDate]" \
  --output text)

if [ -z "$OLD_AMIS" ]; then
  echo "No AMIs older than 30 days found."
  exit 0
fi

echo "Old AMIs to clean up:"
echo "$OLD_AMIS"

while IFS=$'\t' read -r AMI_ID AMI_NAME CREATION_DATE; do
  echo "Processing: $AMI_ID ($AMI_NAME)"

  # Get associated snapshots
  SNAPS=$(aws ec2 describe-images --image-ids $AMI_ID \
    --query 'Images[0].BlockDeviceMappings[*].Ebs.SnapshotId' --output text)

  # Deregister AMI
  aws ec2 deregister-image --image-id $AMI_ID
  echo "  Deregistered: $AMI_ID"

  # Delete associated snapshots
  for SNAP in $SNAPS; do
    aws ec2 delete-snapshot --snapshot-id $SNAP
    echo "  Deleted snapshot: $SNAP"
  done
done <<< "$OLD_AMIS"
```

---

## 5. Interview Practice Scenarios

### Scenario: Consistent Deployments
> "Your team regularly deploys new EC2 instances. Each time, they run a 10-minute bootstrap script to install software. How would you speed this up?"

**Answer**: Build a golden AMI. Configure one instance fully, create an AMI from it. All future deployments launch from this AMI — software is pre-installed, instances are production-ready in 2 minutes instead of 12. Automate AMI builds with EC2 Image Builder or a pipeline. Version your AMIs: when software changes, rebuild the AMI and roll out new instances.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
