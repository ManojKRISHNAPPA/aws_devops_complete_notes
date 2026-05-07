# Amazon EBS and EFS — Practical Assignments

## Table of Contents
1. [Assignment 1 — Create, Attach, and Use an EBS Volume](#1-assignment-1--create-attach-and-use-an-ebs-volume)
2. [Assignment 2 — EBS Snapshots and Restore](#2-assignment-2--ebs-snapshots-and-restore)
3. [Assignment 3 — Move Data Between AZs Using Snapshots](#3-assignment-3--move-data-between-azs-using-snapshots)
4. [Assignment 4 — Set Up EFS and Mount on Multiple Instances](#4-assignment-4--set-up-efs-and-mount-on-multiple-instances)
5. [Assignment 5 — EFS Shared File Test](#5-assignment-5--efs-shared-file-test)
6. [Interview Practice Scenarios](#6-interview-practice-scenarios)

---

## 1. Assignment 1 — Create, Attach, and Use an EBS Volume

### Objective
Create an EBS volume, attach it to a running EC2 instance, format it, mount it, store data, and verify persistence.

### Prerequisites
- A running EC2 instance in us-east-1a

### Steps

**Step 1: Create the Volume**
```bash
# Note the AZ of your running instance first
INSTANCE_ID=i-xxxxxxxxxxxxxxxxx   # replace with yours
AZ=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' \
  --output text)
echo "Instance AZ: $AZ"

# Create a 10 GB gp3 volume in the SAME AZ
VOLUME_ID=$(aws ec2 create-volume \
  --volume-type gp3 \
  --size 10 \
  --availability-zone $AZ \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=ebs-assignment-vol}]' \
  --query 'VolumeId' \
  --output text)

echo "Volume ID: $VOLUME_ID"

# Wait until available
aws ec2 wait volume-available --volume-ids $VOLUME_ID
echo "Volume is available"
```

**Step 2: Attach the Volume**
```bash
aws ec2 attach-volume \
  --volume-id $VOLUME_ID \
  --instance-id $INSTANCE_ID \
  --device /dev/sdf

# Wait for attachment
sleep 10

# Verify attachment
aws ec2 describe-volumes \
  --volume-ids $VOLUME_ID \
  --query 'Volumes[0].Attachments[0].[State,InstanceId,Device]' \
  --output table
```

**Step 3: Format and Mount (SSH into EC2)**
```bash
# SSH into EC2 first
ssh -i ~/.ssh/my-key.pem ec2-user@<public-ip>

# Verify the new device appears
lsblk
# Should show: xvdf (or nvme1n1 on newer instances)

# Check for existing filesystem
sudo file -s /dev/xvdf
# "data" means no filesystem — good, we need to format it

# Create XFS filesystem
sudo mkfs -t xfs /dev/xvdf

# Create a directory
sudo mkdir /ebs-data

# Mount the volume
sudo mount /dev/xvdf /ebs-data

# Verify
df -h /ebs-data
# Should show 10 GB partition mounted at /ebs-data
```

**Step 4: Write Data and Verify Persistence**
```bash
# Write some test data
sudo bash -c 'echo "Hello EBS - $(date)" > /ebs-data/test.txt'
sudo bash -c 'for i in {1..5}; do echo "Line $i of data" >> /ebs-data/testfile.txt; done'
ls -la /ebs-data/
cat /ebs-data/test.txt

# Unmount and remount to verify persistence
sudo umount /ebs-data
sudo mount /dev/xvdf /ebs-data

# Data should still be there
cat /ebs-data/test.txt
ls -la /ebs-data/
```

**Step 5: Make Mount Persistent Across Reboots**
```bash
# Get UUID of the device (more reliable than device name)
sudo blkid /dev/xvdf
# Example output: /dev/xvdf: UUID="abc123..." TYPE="xfs"

# Add to /etc/fstab using UUID
UUID=<your-uuid-here>
echo "UUID=$UUID /ebs-data xfs defaults,nofail 0 2" | sudo tee -a /etc/fstab

# Test fstab is valid
sudo mount -a    # Should succeed with no errors
df -h /ebs-data
```

**Step 6: Clean Up**
```bash
# Unmount from EC2
sudo umount /ebs-data

# Detach from CLI
aws ec2 detach-volume --volume-id $VOLUME_ID

# Wait for detachment
aws ec2 wait volume-available --volume-ids $VOLUME_ID

# Delete the volume
aws ec2 delete-volume --volume-id $VOLUME_ID
```

### Expected Outcome
- You can create, attach, format, mount, and use an EBS volume
- Data persists through unmount/remount cycles
- Volume can be detached and deleted cleanly

---

## 2. Assignment 2 — EBS Snapshots and Restore

### Objective
Create data, take a snapshot, simulate data loss, and restore from the snapshot.

### Steps

**Step 1: Set Up Data**
```bash
# Create a volume and mount it (see Assignment 1 steps)
# Then write important data:
sudo bash -c 'cat > /ebs-data/important.txt << EOF
This is critical business data!
Created: $(date)
Server: $(hostname)
EOF'

sudo bash -c 'for i in {1..100}; do echo "Record $i: user-data-$(date +%s)" >> /ebs-data/records.csv; done'

echo "Files in /ebs-data:"
ls -la /ebs-data/
wc -l /ebs-data/records.csv
```

**Step 2: Take a Snapshot**
```bash
SNAP_ID=$(aws ec2 create-snapshot \
  --volume-id $VOLUME_ID \
  --description "Backup before maintenance - $(date +%Y-%m-%d)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=ebs-assignment-backup}]' \
  --query 'SnapshotId' \
  --output text)

echo "Snapshot: $SNAP_ID"

# Monitor progress
watch -n 5 "aws ec2 describe-snapshots --snapshot-ids $SNAP_ID --query 'Snapshots[0].[State,Progress]' --output text"

# Or just wait
aws ec2 wait snapshot-completed --snapshot-ids $SNAP_ID
echo "Snapshot complete!"
```

**Step 3: Simulate Data Loss**
```bash
# "Accidentally" delete the data
sudo rm /ebs-data/important.txt
sudo bash -c '> /ebs-data/records.csv'   # empty the file

echo "After accidental deletion:"
ls -la /ebs-data/
wc -l /ebs-data/records.csv   # Should be 0 lines
```

**Step 4: Restore from Snapshot**
```bash
# Create a new volume from the snapshot
RESTORED_ID=$(aws ec2 create-volume \
  --snapshot-id $SNAP_ID \
  --availability-zone $AZ \
  --volume-type gp3 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=restored-volume}]' \
  --query 'VolumeId' \
  --output text)

echo "Restored volume: $RESTORED_ID"
aws ec2 wait volume-available --volume-ids $RESTORED_ID

# Attach restored volume
aws ec2 attach-volume \
  --volume-id $RESTORED_ID \
  --instance-id $INSTANCE_ID \
  --device /dev/sdg

sleep 10

# Mount the restored volume
sudo mkdir /restored-data
sudo mount /dev/xvdg /restored-data

# Verify data is back!
ls -la /restored-data/
cat /restored-data/important.txt
wc -l /restored-data/records.csv    # Back to 100 lines!
```

**Step 5: Clean Up**
```bash
sudo umount /restored-data
aws ec2 detach-volume --volume-id $RESTORED_ID
aws ec2 wait volume-available --volume-ids $RESTORED_ID
aws ec2 delete-volume --volume-id $RESTORED_ID
aws ec2 delete-snapshot --snapshot-id $SNAP_ID
```

---

## 3. Assignment 3 — Move Data Between AZs Using Snapshots

### Objective
Move an EBS volume from us-east-1a to us-east-1b — this requires snapshots since volumes are AZ-locked.

### Steps

```bash
# 1. Verify original volume is in us-east-1a
aws ec2 describe-volumes \
  --volume-ids $VOLUME_ID \
  --query 'Volumes[0].AvailabilityZone' \
  --output text

# 2. Create snapshot
SNAP_ID=$(aws ec2 create-snapshot \
  --volume-id $VOLUME_ID \
  --description "AZ migration snapshot" \
  --query 'SnapshotId' --output text)
aws ec2 wait snapshot-completed --snapshot-ids $SNAP_ID

# 3. Create new volume in a DIFFERENT AZ from the snapshot
NEW_VOLUME=$(aws ec2 create-volume \
  --snapshot-id $SNAP_ID \
  --availability-zone us-east-1b \    # Different AZ!
  --volume-type gp3 \
  --query 'VolumeId' --output text)

aws ec2 wait volume-available --volume-ids $NEW_VOLUME

echo "Original volume (us-east-1a): $VOLUME_ID"
echo "New volume (us-east-1b): $NEW_VOLUME"

# 4. Verify the new volume details
aws ec2 describe-volumes --volume-ids $NEW_VOLUME --output table
```

---

## 4. Assignment 4 — Set Up EFS and Mount on Multiple Instances

### Objective
Create an EFS file system, create mount targets, and mount on two EC2 instances in different AZs.

### Steps

**Step 1: Launch Two EC2 Instances in Different AZs**
```bash
# Get two different subnets in different AZs
SUBNET_AZ_A=$(aws ec2 describe-subnets \
  --filters "Name=availabilityZone,Values=us-east-1a" \
  --query 'Subnets[0].SubnetId' --output text)

SUBNET_AZ_B=$(aws ec2 describe-subnets \
  --filters "Name=availabilityZone,Values=us-east-1b" \
  --query 'Subnets[0].SubnetId' --output text)

# Get AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*" "Name=architecture,Values=x86_64" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' --output text)

# Create security group that allows NFS (port 2049)
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query 'Vpcs[0].VpcId' --output text)

SG_ID=$(aws ec2 create-security-group \
  --group-name efs-demo-sg \
  --description "EFS Demo Security Group" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

# Allow SSH and NFS
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 2049 --source-group $SG_ID

# Launch Instance 1 in AZ-a
INSTANCE_1=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name my-learning-key \
  --security-group-ids $SG_ID \
  --subnet-id $SUBNET_AZ_A \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=efs-instance-1}]' \
  --query 'Instances[0].InstanceId' --output text)

# Launch Instance 2 in AZ-b
INSTANCE_2=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name my-learning-key \
  --security-group-ids $SG_ID \
  --subnet-id $SUBNET_AZ_B \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=efs-instance-2}]' \
  --query 'Instances[0].InstanceId' --output text)

echo "Instance 1: $INSTANCE_1 (AZ-a)"
echo "Instance 2: $INSTANCE_2 (AZ-b)"
aws ec2 wait instance-running --instance-ids $INSTANCE_1 $INSTANCE_2
```

**Step 2: Create EFS File System**
```bash
EFS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted \
  --tags Key=Name,Value=efs-demo \
  --query 'FileSystemId' --output text)

echo "EFS: $EFS_ID"

# Wait until available
while [[ $(aws efs describe-file-systems --file-system-id $EFS_ID --query 'FileSystems[0].LifeCycleState' --output text) != "available" ]]; do
  echo "Waiting for EFS..."
  sleep 5
done

# Create mount targets in both subnets
aws efs create-mount-target --file-system-id $EFS_ID --subnet-id $SUBNET_AZ_A --security-groups $SG_ID
aws efs create-mount-target --file-system-id $EFS_ID --subnet-id $SUBNET_AZ_B --security-groups $SG_ID

echo "Mount targets created. Wait 1 minute before mounting..."
sleep 60
```

**Step 3: Mount EFS on Both Instances**
```bash
# Get IPs
IP_1=$(aws ec2 describe-instances --instance-ids $INSTANCE_1 --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
IP_2=$(aws ec2 describe-instances --instance-ids $INSTANCE_2 --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

# On Instance 1:
ssh -i ~/.ssh/my-learning-key.pem ec2-user@$IP_1 << EOF
sudo dnf install -y amazon-efs-utils
sudo mkdir /mnt/efs
sudo mount -t efs $EFS_ID:/ /mnt/efs
df -h /mnt/efs
echo "Mounted EFS on Instance 1"
EOF

# On Instance 2:
ssh -i ~/.ssh/my-learning-key.pem ec2-user@$IP_2 << EOF
sudo dnf install -y amazon-efs-utils
sudo mkdir /mnt/efs
sudo mount -t efs $EFS_ID:/ /mnt/efs
df -h /mnt/efs
echo "Mounted EFS on Instance 2"
EOF
```

---

## 5. Assignment 5 — EFS Shared File Test

### Objective
Prove EFS shared access by writing on one instance and reading from another.

### Steps

```bash
# Write from Instance 1
ssh -i ~/.ssh/my-learning-key.pem ec2-user@$IP_1 << 'EOF'
echo "Written from Instance 1 at $(date)" > /mnt/efs/shared.txt
for i in {1..5}; do
  echo "Entry $i from Instance-1: $(date)" >> /mnt/efs/log.txt
done
echo "Files written from Instance 1:"
ls -la /mnt/efs/
EOF

# Read from Instance 2
ssh -i ~/.ssh/my-learning-key.pem ec2-user@$IP_2 << 'EOF'
echo "=== Reading from Instance 2 ==="
cat /mnt/efs/shared.txt
cat /mnt/efs/log.txt

# Now write from Instance 2
echo "Written from Instance 2 at $(date)" >> /mnt/efs/shared.txt
echo "=== Updated file ==="
cat /mnt/efs/shared.txt
EOF

# Read back from Instance 1 to see Instance 2's writes
ssh -i ~/.ssh/my-learning-key.pem ec2-user@$IP_1 << 'EOF'
echo "=== Reading updates from Instance 1 ==="
cat /mnt/efs/shared.txt
EOF
```

### Expected Output
```
=== Reading from Instance 2 ===
Written from Instance 1 at [timestamp]
...
=== Reading updates from Instance 1 ===
Written from Instance 1 at [timestamp]
Written from Instance 2 at [timestamp]
```

### Clean Up
```bash
# Terminate instances
aws ec2 terminate-instances --instance-ids $INSTANCE_1 $INSTANCE_2

# Delete mount targets
MOUNT_TARGETS=$(aws efs describe-mount-targets --file-system-id $EFS_ID --query 'MountTargets[*].MountTargetId' --output text)
for MT in $MOUNT_TARGETS; do
  aws efs delete-mount-target --mount-target-id $MT
done

# Wait and delete EFS
sleep 30
aws efs delete-file-system --file-system-id $EFS_ID

# Delete security group
aws ec2 delete-security-group --group-id $SG_ID
```

---

## 6. Interview Practice Scenarios

### Scenario 1: Shared Media Storage
> "You're running a WordPress site across 3 EC2 instances behind a load balancer. Users upload images that need to be accessible from all instances. How would you store these images?"

**Answer**: Use EFS. Mount the EFS volume at the WordPress uploads directory (`/var/www/html/wp-content/uploads`) on all 3 instances. EFS is NFS-based and allows concurrent read/write from all instances across AZs. The alternative (and often better for static files) is to use S3 with a CDN (CloudFront), but EFS works when you need a traditional filesystem interface.

### Scenario 2: Database Backup Strategy
> "How would you automate daily backups of an EBS volume hosting your MySQL database?"

**Answer**: 
1. Create a Lambda function triggered by CloudWatch Events (EventBridge) daily at 2 AM
2. Lambda calls `ec2:CreateSnapshot` on the database volume
3. Apply a lifecycle policy: keep last 7 daily snapshots, 4 weekly, 12 monthly
4. Optionally copy snapshots to another region for disaster recovery
5. Test restore monthly: create a volume from the snapshot, mount it, verify data integrity

### Scenario 3: EBS vs EFS Decision
> "You have an application that processes video files. The processing is CPU-intensive and you plan to run it on 10 EC2 instances in parallel, all needing access to the same source video files. What storage would you choose?"

**Answer**: EFS with Max I/O performance mode. The key requirement is concurrent access from 10 instances. EFS provides this natively. Use Max I/O mode for high-parallelism workloads. For the processed output files, consider writing them to S3 for durability and cost efficiency.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
