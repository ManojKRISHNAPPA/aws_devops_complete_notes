# Amazon EBS and EFS — Complete Guide

## Table of Contents
1. [What is EBS?](#1-what-is-ebs)
2. [EBS Volume Types](#2-ebs-volume-types)
3. [EBS Operations](#3-ebs-operations)
4. [EBS Snapshots](#4-ebs-snapshots)
5. [EBS Encryption](#5-ebs-encryption)
6. [EBS Performance Tips](#6-ebs-performance-tips)
7. [What is EFS?](#7-what-is-efs)
8. [EFS Storage Classes](#8-efs-storage-classes)
9. [EFS vs EBS — When to Use Which](#9-efs-vs-ebs--when-to-use-which)
10. [EFS Setup and Mounting](#10-efs-setup-and-mounting)
11. [EBS and EFS Pricing Overview](#11-ebs-and-efs-pricing-overview)
12. [Common Troubleshooting](#12-common-troubleshooting)
13. [Cheat Sheet](#13-cheat-sheet)
14. [Common Interview Questions](#14-common-interview-questions)

---

## 1. What is EBS?

**Amazon EBS (Elastic Block Store)** is a **network-attached block storage** service for use with EC2 instances. Think of it as a **virtual hard drive in the cloud**.

```
Physical Computer                   AWS EC2 + EBS
─────────────────                   ───────────────────────────
Hard drive (SATA/NVMe)              EBS Volume (network-attached)
Plugged into motherboard            Attached via network
One disk, one computer              One volume, one instance (usually)
Lost if hardware fails              Replicated within AZ for durability
```

**Key EBS characteristics:**
- Lives in a specific **Availability Zone** — cannot attach to an instance in a different AZ
- **Replicated** within its AZ for redundancy
- **Independent** from the EC2 instance — if you terminate an instance, EBS can persist
- Can be **detached and reattached** to another instance in the same AZ
- Supports **snapshots** (backups to S3)
- Can be **encrypted** with AWS KMS

```
AZ: us-east-1a
┌─────────────────────────────────────────────────┐
│                                                 │
│  ┌──────────────────┐      ┌─────────────────┐  │
│  │   EC2 Instance   │─────►│   EBS Volume    │  │
│  │   (i-xxxx)       │      │   (/dev/xvdf)   │  │
│  └──────────────────┘      │   100 GB gp3    │  │
│                             └─────────────────┘  │
│                             ↑ Network attached   │
└─────────────────────────────────────────────────┘

AZ: us-east-1b ← This instance CANNOT attach the above volume
```

---

## 2. EBS Volume Types

### General Purpose SSD (gp2 and gp3)

**gp3 (current generation — recommended)**
- Baseline: **3,000 IOPS**, **125 MB/s** regardless of volume size
- Can provision up to **16,000 IOPS** and **1,000 MB/s** independently
- Best for: most workloads, boot volumes, dev/test, small-medium databases

**gp2 (older generation)**
- IOPS tied to size: 3 IOPS/GB (min 100, max 16,000)
- Baseline for 100 GB volume = 300 IOPS
- **Use gp3 instead** — same or better performance, 20% cheaper

### Provisioned IOPS SSD (io1 and io2)

**io2 Block Express (top tier)**
- Up to **256,000 IOPS** per volume
- Up to **4,000 MB/s throughput**
- 99.999% durability (vs 99.8–99.9% for gp2/gp3)
- Best for: mission-critical databases (Oracle, SQL Server), I/O intensive applications

**io1 (older)**
- Up to **64,000 IOPS**
- Use io2 instead

### Throughput Optimized HDD (st1)

- Not SSD — **magnetic spinning disk**
- Optimized for **high throughput, large sequential reads/writes**
- Low cost
- **Cannot be a boot volume**
- Best for: data warehouses, log processing, Kafka, big data

### Cold HDD (sc1)

- Cheapest EBS option
- Lowest performance (250 IOPS max)
- Best for: infrequently accessed large data, archives
- **Cannot be a boot volume**

### Comparison Table

| Type | Max IOPS | Max Throughput | Boot? | Use Case |
|------|----------|----------------|-------|---------|
| gp3 | 16,000 | 1,000 MB/s | Yes | Most workloads |
| gp2 | 16,000 | 250 MB/s | Yes | Legacy (use gp3) |
| io2 | 256,000 | 4,000 MB/s | Yes | Critical databases |
| io1 | 64,000 | 1,000 MB/s | Yes | I/O intensive |
| st1 | 500 | 500 MB/s | No | Big data, logs |
| sc1 | 250 | 250 MB/s | No | Cold/archive data |

---

## 3. EBS Operations

### Creating a Volume

```bash
# Create a 50 GB gp3 volume in us-east-1a
aws ec2 create-volume \
  --volume-type gp3 \
  --size 50 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=my-data-volume}]'

# Expected output:
# {
#     "VolumeId": "vol-xxxxxxxxxxxxxxxxx",
#     "State": "creating",
#     "Size": 50,
#     "VolumeType": "gp3"
# }
```

### Attaching a Volume to an EC2 Instance

```bash
# The instance MUST be in the same AZ as the volume!
aws ec2 attach-volume \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --instance-id i-xxxxxxxxxxxxxxxxx \
  --device /dev/sdf

# Check attachment status
aws ec2 describe-volumes \
  --volume-ids vol-xxxxxxxxxxxxxxxxx \
  --query 'Volumes[0].Attachments'
```

### Mounting the Volume (inside the EC2 instance via SSH)

```bash
# SSH into your instance first, then:

# Check attached block devices
lsblk

# Check if volume has a filesystem
sudo file -s /dev/xvdf   # Shows "data" if no filesystem

# Create a filesystem (only needed first time)
sudo mkfs -t xfs /dev/xvdf

# Create mount point
sudo mkdir /data

# Mount the volume
sudo mount /dev/xvdf /data

# Verify
df -h /data

# Make mount persistent across reboots
echo "/dev/xvdf /data xfs defaults,nofail 0 2" | sudo tee -a /etc/fstab
```

### Resizing a Volume

```bash
# Modify volume size (can increase, not decrease)
aws ec2 modify-volume \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --size 100   # Increase to 100 GB

# Check modification status
aws ec2 describe-volumes-modifications \
  --volume-ids vol-xxxxxxxxxxxxxxxxx

# After modification is complete, resize the filesystem (on the instance):
sudo growpart /dev/xvdf 1      # Extend partition
sudo xfs_growfs /data          # Resize XFS filesystem
# OR for ext4:
sudo resize2fs /dev/xvdf
```

### Detaching a Volume

```bash
# Unmount first (from inside EC2):
sudo umount /data

# Then detach
aws ec2 detach-volume --volume-id vol-xxxxxxxxxxxxxxxxx

# Delete if no longer needed
aws ec2 delete-volume --volume-id vol-xxxxxxxxxxxxxxxxx
```

---

## 4. EBS Snapshots

Snapshots are **point-in-time backups** of EBS volumes stored in S3. They are **incremental** — only changed blocks are stored after the first snapshot.

```
Snapshot Lifecycle:
───────────────────
Volume (50 GB)
     │
     ├──► Snapshot 1 (Day 1) — Full 50 GB stored
     │         │
     │    Changed 5 GB
     ├──► Snapshot 2 (Day 2) — Only 5 GB delta stored
     │         │
     │    Changed 2 GB
     └──► Snapshot 3 (Day 3) — Only 2 GB delta stored

All snapshots appear as full 50 GB backups to you,
but only the unique blocks are stored → cost efficient
```

### Creating Snapshots

```bash
# Create a snapshot
aws ec2 create-snapshot \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --description "Daily backup $(date +%Y-%m-%d)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=my-daily-backup}]'

# Wait for completion
aws ec2 wait snapshot-completed --snapshot-ids snap-xxxxxxxxxxxxxxxxx

# List your snapshots
aws ec2 describe-snapshots \
  --owner-ids self \
  --output table
```

### Restoring from Snapshot

```bash
# Create a new volume from a snapshot
aws ec2 create-volume \
  --snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --availability-zone us-east-1a \
  --volume-type gp3

# This creates a full volume you can attach to any instance in us-east-1a
```

### Copying Snapshots to Another Region

```bash
# Copy snapshot to eu-west-1 (useful for DR or migration)
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --description "Cross-region copy" \
  --region eu-west-1
```

> **Snapshots are AZ-independent** — stored in S3, they can be used to create volumes in any AZ within the same region.

---

## 5. EBS Encryption

EBS encryption uses **AWS KMS** keys and operates transparently.

- **Data at rest** is encrypted (the bits on disk)
- **Data in transit** between EC2 and EBS is encrypted
- **Snapshots** of encrypted volumes are also encrypted
- **Zero performance impact** — encryption/decryption happens at the hardware level

```bash
# Create an encrypted volume
aws ec2 create-volume \
  --volume-type gp3 \
  --size 50 \
  --availability-zone us-east-1a \
  --encrypted

# Create encrypted volume with a specific KMS key
aws ec2 create-volume \
  --volume-type gp3 \
  --size 50 \
  --availability-zone us-east-1a \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxxxxxxx
```

### Enable Default EBS Encryption for the Account

```bash
# All new volumes will be encrypted by default
aws ec2 enable-ebs-encryption-by-default --region us-east-1

# Verify
aws ec2 get-ebs-encryption-by-default --region us-east-1
```

---

## 6. EBS Performance Tips

| Technique | Benefit |
|-----------|---------|
| Use gp3 over gp2 | 20% cheaper, independently scalable IOPS |
| Use io2 for critical databases | Higher IOPS, 99.999% durability |
| Enable EBS-optimized on EC2 | Dedicated bandwidth between EC2 and EBS |
| Use RAID 0 | Stripe across volumes for higher combined IOPS |
| Snapshot during low traffic | Snapshots can impact IOPS (especially on io1) |
| Pre-warm volumes from snapshots | New volumes from snapshots have lazy loading |

---

## 7. What is EFS?

**Amazon EFS (Elastic File System)** is a **managed NFS (Network File System)** service. Unlike EBS which is block storage for one instance, EFS is a **shared file system** that can be mounted by **many EC2 instances simultaneously**, even across multiple AZs.

```
EBS — One volume, one instance at a time:
┌─────────────┐    ┌─────────────┐
│  EC2 (AZ-a) │───►│ EBS Volume  │
└─────────────┘    └─────────────┘

EFS — One filesystem, many instances across AZs:
┌─────────────┐
│  EC2 (AZ-a) │──────┐
└─────────────┘      │
┌─────────────┐      ▼
│  EC2 (AZ-a) │────►┌──────────────┐
└─────────────┘      │  EFS         │
┌─────────────┐      │  /mnt/efs    │
│  EC2 (AZ-b) │────► │  (NFS)      │
└─────────────┘      └──────────────┘
┌─────────────┐             ▲
│  EC2 (AZ-c) │─────────────┘
└─────────────┘
```

**Key EFS characteristics:**
- Fully managed NFS v4.1 file system
- **Elastic** — grows and shrinks automatically, no capacity management needed
- **Multi-AZ** — data stored redundantly across multiple AZs
- Pay only for what you use (per GB/month)
- Accessible from on-premises via Direct Connect or VPN
- Works only with **Linux** (not Windows)

---

## 8. EFS Storage Classes

| Storage Class | Use Case | Cost |
|---------------|----------|------|
| **EFS Standard** | Frequently accessed files | $$$ |
| **EFS Standard-IA** | Infrequently accessed files, >30 days | $ |
| **EFS One Zone** | Single AZ, frequently accessed | $$ |
| **EFS One Zone-IA** | Single AZ, infrequently accessed | Cheapest |

> Enable **EFS Lifecycle Management** to automatically move files not accessed for 30 (or 7, 14, 60, 90) days to Standard-IA, saving up to 92% on those files.

### Performance Modes

| Mode | Best For |
|------|---------|
| **General Purpose** (default) | Web serving, CMS, home directories |
| **Max I/O** | Big data, media processing, parallelized workloads (10,000+ clients) |

### Throughput Modes

| Mode | Best For |
|------|---------|
| **Bursting** | Scales with storage size, good for most workloads |
| **Provisioned** | Specify throughput independently of storage size |
| **Elastic** | Auto-scales throughput up and down |

---

## 9. EFS vs EBS — When to Use Which

| Feature | EBS | EFS |
|---------|-----|-----|
| **Protocol** | Block (like hard drive) | NFS (network file system) |
| **Concurrent access** | 1 instance at a time (usually) | Thousands of instances simultaneously |
| **AZ scope** | Locked to one AZ | Multi-AZ by default |
| **OS** | Linux and Windows | Linux only |
| **Capacity** | Fixed (you size it) | Elastic (auto-grows) |
| **Price** | Lower | Higher |
| **Use case** | Boot volumes, databases | Shared content, CMS, ECS tasks |

**Use EBS when:**
- Running a database (MySQL, PostgreSQL)
- Boot volume for EC2
- High-performance single-instance storage
- Windows instances

**Use EFS when:**
- Multiple instances need to share the same files
- WordPress/CMS media files across web servers
- Machine learning training data shared across nodes
- Container workloads with shared storage (ECS, EKS)

---

## 10. EFS Setup and Mounting

### Step 1: Create an EFS File System

```bash
# Create EFS file system
EFS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted \
  --tags Key=Name,Value=my-efs \
  --query 'FileSystemId' \
  --output text)

echo "EFS: $EFS_ID"

# Wait until available
aws efs describe-file-systems --file-system-id $EFS_ID --query 'FileSystems[0].LifeCycleState'

# Create mount targets in each subnet/AZ
SUBNET_IDS=$(aws ec2 describe-subnets --query 'Subnets[*].SubnetId' --output text)
SG_ID=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=default" --query 'SecurityGroups[0].GroupId' --output text)

for SUBNET in $SUBNET_IDS; do
  aws efs create-mount-target \
    --file-system-id $EFS_ID \
    --subnet-id $SUBNET \
    --security-groups $SG_ID
  echo "Mount target created in subnet: $SUBNET"
done
```

### Step 2: Mount on EC2 Instance

```bash
# SSH into EC2 instance

# Install EFS mount helper
sudo dnf install -y amazon-efs-utils

# Create mount point
sudo mkdir /mnt/efs

# Mount using EFS mount helper (handles NFS options automatically)
sudo mount -t efs $EFS_ID:/ /mnt/efs

# Or using TLS encryption
sudo mount -t efs -o tls $EFS_ID:/ /mnt/efs

# Verify
df -h /mnt/efs

# Make persistent
echo "$EFS_ID:/ /mnt/efs efs _netdev,tls 0 0" | sudo tee -a /etc/fstab
```

### Step 3: Test Shared Access

```bash
# On Instance 1:
echo "Written from Instance 1" | sudo tee /mnt/efs/shared-test.txt

# On Instance 2 (also mounted to same EFS):
cat /mnt/efs/shared-test.txt
# Output: Written from Instance 1
```

---

## 11. EBS and EFS Pricing Overview

| Storage | Price (us-east-1 approx) |
|---------|--------------------------|
| EBS gp3 | $0.08/GB-month |
| EBS io2 | $0.125/GB-month + $0.065/IOPS-month |
| EBS Snapshots | $0.05/GB-month |
| EFS Standard | $0.30/GB-month |
| EFS Standard-IA | $0.025/GB-month |
| EFS One Zone | $0.16/GB-month |

> EFS is ~4x more expensive than EBS per GB — only use it when you need shared access.

---

## 12. Common Troubleshooting

### EBS volume not visible on instance
```bash
# Check attached devices
lsblk
# Device may show as xvdf instead of sdf on some systems

# Check if filesystem exists
sudo file -s /dev/xvdf
# If output is "data", no filesystem — needs mkfs
```

### EFS mount fails
```bash
# Check: NFS port 2049 open in security group
aws ec2 describe-security-groups --group-ids $SG_ID --query 'SecurityGroups[0].IpPermissions'

# Check: mount target is in Available state
aws efs describe-mount-targets --file-system-id $EFS_ID

# Check: EC2 instance can reach the EFS mount target DNS
nslookup $EFS_ID.efs.us-east-1.amazonaws.com
telnet $EFS_ID.efs.us-east-1.amazonaws.com 2049
```

### EBS performance is lower than expected
```bash
# Check if instance type supports EBS-optimized
aws ec2 describe-instance-types \
  --instance-types m5.large \
  --query 'InstanceTypes[0].EbsInfo.EbsOptimizedSupport'

# Enable EBS-optimized (if instance is stopped)
aws ec2 modify-instance-attribute \
  --instance-id i-xxxx \
  --ebs-optimized
```

---

## 13. Cheat Sheet

```bash
# EBS Operations
aws ec2 create-volume --volume-type gp3 --size 20 --availability-zone us-east-1a
aws ec2 attach-volume --volume-id vol-xxx --instance-id i-xxx --device /dev/sdf
aws ec2 detach-volume --volume-id vol-xxx
aws ec2 modify-volume --volume-id vol-xxx --size 100   # resize

# Snapshots
aws ec2 create-snapshot --volume-id vol-xxx --description "backup"
aws ec2 describe-snapshots --owner-ids self --output table
aws ec2 delete-snapshot --snapshot-id snap-xxx
aws ec2 copy-snapshot --source-region us-east-1 --source-snapshot-id snap-xxx --region eu-west-1

# EFS
aws efs create-file-system --performance-mode generalPurpose --encrypted
aws efs create-mount-target --file-system-id fs-xxx --subnet-id subnet-xxx
aws efs describe-file-systems --output table
aws efs delete-file-system --file-system-id fs-xxx

# On EC2: mount
sudo mount -t efs fs-xxx:/ /mnt/efs
sudo mount -t nfs4 -o nfsvers=4.1 <efs-dns>:/ /mnt/efs
```

---

## 14. Common Interview Questions

**Q: What is the difference between EBS and EFS?**
> EBS is block storage attached to a single EC2 instance — like a hard drive. EFS is a shared NFS file system that can be mounted by multiple EC2 instances simultaneously across AZs. Use EBS for databases and OS volumes; use EFS for shared file storage across multiple instances.

**Q: Are EBS snapshots incremental?**
> Yes. The first snapshot copies all data. Subsequent snapshots only copy changed blocks. However, each snapshot appears as a full backup — you can restore from any single snapshot without needing the previous ones.

**Q: Can you attach an EBS volume to an EC2 instance in a different AZ?**
> No. EBS volumes are tied to a specific AZ. To move data to a different AZ, take a snapshot and create a new volume from that snapshot in the target AZ.

**Q: What happens to EBS data when an EC2 instance terminates?**
> The root EBS volume is deleted by default (DeleteOnTermination = true). Additional data volumes persist by default. You can change the root volume behavior at launch time or via the console.

**Q: What is EBS-optimized?**
> EBS-optimized instances have dedicated network bandwidth between the instance and EBS, preventing I/O from competing with other network traffic. Most modern instance types have it enabled by default and it's critical for production database workloads.

**Q: What EBS type would you use for a high-performance production database?**
> io2 or io2 Block Express for mission-critical databases requiring predictable, high IOPS (up to 256,000) and 99.999% durability. For less demanding databases, gp3 with provisioned IOPS is usually sufficient.

**Q: Can EFS be used on Windows instances?**
> No. EFS uses the NFS protocol which is natively supported on Linux. Windows instances should use EBS, FSx for Windows File Server (SMB), or S3.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
