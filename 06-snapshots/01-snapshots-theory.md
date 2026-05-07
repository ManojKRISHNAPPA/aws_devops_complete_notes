# Amazon Snapshots — Complete Guide

## Table of Contents
1. [What is an EBS Snapshot?](#1-what-is-an-ebs-snapshot)
2. [How Snapshots Work (Incremental)](#2-how-snapshots-work-incremental)
3. [Creating Snapshots](#3-creating-snapshots)
4. [Restoring from Snapshots](#4-restoring-from-snapshots)
5. [Snapshot Lifecycle Manager](#5-snapshot-lifecycle-manager)
6. [Cross-Region and Cross-Account Copies](#6-cross-region-and-cross-account-copies)
7. [Snapshot Encryption](#7-snapshot-encryption)
8. [Snapshot Pricing](#8-snapshot-pricing)
9. [Snapshot Best Practices](#9-snapshot-best-practices)
10. [Cheat Sheet](#10-cheat-sheet)
11. [Common Interview Questions](#11-common-interview-questions)

---

## 1. What is an EBS Snapshot?

An **EBS Snapshot** is a **point-in-time backup** of an EBS volume, stored durably in **Amazon S3** (managed by AWS — you don't see the S3 bucket directly).

```
EBS Volume (50 GB)          S3 (AWS-managed)
┌──────────────┐            ┌──────────────────┐
│  OS files    │            │                  │
│  App data    │── Snapshot►│  snap-xxxx       │
│  User data   │            │  (50 GB backup)  │
│  Log files   │            │                  │
└──────────────┘            └──────────────────┘
                                    │
                              Restore from here
                              to create new volume
                              in any AZ or region
```

**Key facts:**
- Stored in S3 (not in your bucket — AWS manages it)
- Can be used to create new EBS volumes in any AZ
- Can be copied to other regions (for disaster recovery)
- Can be shared with other AWS accounts
- Form the basis of **AMIs** (AMI = snapshot + metadata)
- Billed per GB of actual stored data (incremental after first)

---

## 2. How Snapshots Work (Incremental)

This is one of the most important concepts about snapshots.

```
Day 1: First snapshot (full)
Volume:  [Block A] [Block B] [Block C] [Block D] [Block E]
                                                        ↓
Snapshot 1: Stores ALL blocks → 50 GB

──────────────────────────────────────────────────────────

Day 2: Changes to Block B and D
Volume:  [Block A] [Block B'] [Block C] [Block D'] [Block E]
                                                         ↓
Snapshot 2: Stores ONLY changed blocks → B' + D' = small

──────────────────────────────────────────────────────────

Day 3: Changes to Block A
Volume:  [Block A'] [Block B'] [Block C] [Block D'] [Block E]
                                                          ↓
Snapshot 3: Stores ONLY Block A' = tiny

BUT: You can restore the full 50 GB volume from any single snapshot.
AWS assembles the blocks from previous snapshots as needed.
```

### Why This Matters
- **Cost**: You only pay for unique changed blocks across all snapshots, not 50 GB per day
- **Restoration**: Any snapshot acts as a complete, self-sufficient backup
- **Deletion**: You can delete older snapshots — data is automatically migrated to preserve the integrity of newer snapshots

---

## 3. Creating Snapshots

### Manual Snapshots

```bash
# Create snapshot of a volume
SNAP_ID=$(aws ec2 create-snapshot \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --description "Pre-deployment backup $(date +%Y-%m-%d %H:%M)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[
    {Key=Name,Value=web-server-backup},
    {Key=Environment,Value=production},
    {Key=CreatedBy,Value=manual}
  ]' \
  --query 'SnapshotId' \
  --output text)

echo "Creating snapshot: $SNAP_ID"

# Wait for completion (can take minutes for large volumes)
aws ec2 wait snapshot-completed --snapshot-ids $SNAP_ID
echo "Snapshot complete: $SNAP_ID"
```

### Check Snapshot Progress

```bash
# Watch progress
aws ec2 describe-snapshots \
  --snapshot-ids $SNAP_ID \
  --query 'Snapshots[0].[SnapshotId,State,Progress,StartTime]' \
  --output table

# List all your snapshots
aws ec2 describe-snapshots \
  --owner-ids self \
  --query 'Snapshots[*].[SnapshotId,Description,State,StartTime,VolumeSize]' \
  --output table
```

### Snapshot of Root Volume While Instance is Running

You can snapshot a running instance's root volume. For databases, it's best to:
1. Flush and pause writes if possible
2. Take snapshot
3. Resume writes

```bash
# Snapshot all volumes attached to an instance
INSTANCE_ID=i-xxxxxxxxxxxxxxxxx

# Get all volume IDs attached to this instance
VOLUMES=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings[*].Ebs.VolumeId' \
  --output text)

echo "Volumes: $VOLUMES"

# Create snapshots for all volumes
for VOL in $VOLUMES; do
  SNAP=$(aws ec2 create-snapshot \
    --volume-id $VOL \
    --description "Backup of $VOL from $INSTANCE_ID" \
    --query 'SnapshotId' --output text)
  echo "Snapshot for $VOL: $SNAP"
done
```

---

## 4. Restoring from Snapshots

### Restore to a New Volume

```bash
# Create new volume from snapshot
RESTORED_VOL=$(aws ec2 create-volume \
  --snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --availability-zone us-east-1a \
  --volume-type gp3 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=restored-volume}]' \
  --query 'VolumeId' \
  --output text)

aws ec2 wait volume-available --volume-ids $RESTORED_VOL
echo "Restored volume: $RESTORED_VOL"

# Attach to an instance and mount
aws ec2 attach-volume \
  --volume-id $RESTORED_VOL \
  --instance-id $INSTANCE_ID \
  --device /dev/sdg
```

### Replace Root Volume (Instance Recovery)

If your instance's root volume is corrupted:

```bash
# 1. Stop the failed instance
aws ec2 stop-instances --instance-ids $INSTANCE_ID

# 2. Detach the corrupted root volume
aws ec2 detach-volume --volume-id <corrupted-vol-id>

# 3. Create a new root volume from a clean snapshot
CLEAN_VOL=$(aws ec2 create-volume \
  --snapshot-id <clean-snapshot> \
  --availability-zone $AZ \
  --volume-type gp3 \
  --query 'VolumeId' --output text)

aws ec2 wait volume-available --volume-ids $CLEAN_VOL

# 4. Attach as root device
aws ec2 attach-volume \
  --volume-id $CLEAN_VOL \
  --instance-id $INSTANCE_ID \
  --device /dev/xvda    # Root device

# 5. Start instance
aws ec2 start-instances --instance-ids $INSTANCE_ID
```

---

## 5. Snapshot Lifecycle Manager

**Amazon Data Lifecycle Manager (DLM)** automates snapshot creation and deletion. Instead of writing Lambda functions, you define policies.

```
DLM Policy:
───────────
Target: Volumes tagged Environment=production
Schedule: Daily at 02:00 UTC
Retain: Last 7 daily, 4 weekly snapshots
Cross-region copy: eu-west-1 (DR copy)
```

### Create an Automated Snapshot Policy via CLI

```bash
# Create IAM role for DLM first (one-time setup)
# AWS provides a default role: AWSDataLifecycleManagerDefaultRole

# Create lifecycle policy
aws dlm create-lifecycle-policy \
  --description "Daily backups for production volumes" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/AWSDataLifecycleManagerDefaultRole \
  --policy-details '{
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "ResourceTypes": ["VOLUME"],
    "TargetTags": [{"Key": "Environment", "Value": "production"}],
    "Schedules": [{
      "Name": "DailyBackup",
      "CreateRule": {
        "Interval": 24,
        "IntervalUnit": "HOURS",
        "Times": ["02:00"]
      },
      "RetainRule": {
        "Count": 7
      },
      "CopyTags": true
    }]
  }'
```

### View Your Lifecycle Policies

```bash
aws dlm get-lifecycle-policies --output table
```

---

## 6. Cross-Region and Cross-Account Copies

### Copy to Another Region (Disaster Recovery)

```bash
# Copy snapshot to eu-west-1
COPIED_SNAP=$(aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --description "DR copy for compliance" \
  --region eu-west-1 \
  --query 'SnapshotId' \
  --output text)

echo "Snapshot being copied to eu-west-1: $COPIED_SNAP"

# Wait in the destination region
aws ec2 wait snapshot-completed \
  --snapshot-ids $COPIED_SNAP \
  --region eu-west-1
```

### Share Snapshot with Another AWS Account

```bash
# Share with specific account
aws ec2 modify-snapshot-attribute \
  --snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --attribute createVolumePermission \
  --operation-type add \
  --user-ids 123456789012

# Make public (not recommended for production data!)
aws ec2 modify-snapshot-attribute \
  --snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --attribute createVolumePermission \
  --operation-type add \
  --group-names all

# Verify sharing settings
aws ec2 describe-snapshot-attribute \
  --snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --attribute createVolumePermission
```

---

## 7. Snapshot Encryption

```bash
# Create encrypted snapshot from unencrypted volume
aws ec2 create-snapshot \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --description "Encrypted backup" \
  # Encryption is inherited from source volume

# To create encrypted copy of an unencrypted snapshot:
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --description "Encrypted copy" \
  --encrypted \
  --kms-key-id alias/aws/ebs \    # or your custom key
  --region us-east-1
```

> **Key rule**: Snapshots inherit the encryption status of the source volume. You cannot create an unencrypted snapshot from an encrypted volume.

---

## 8. Snapshot Pricing

- **$0.05 per GB-month** (us-east-1)
- Billed for unique data stored (incremental — not full volume size per snapshot)
- Cross-region copy: same $0.05/GB plus data transfer charges

**Cost calculation example:**
```
Volume: 100 GB
Day 1 snapshot: 60 GB actual unique data = $3.00/month
Day 2 snapshot: 5 GB changed data       = $0.25/month
Day 3 snapshot: 2 GB changed data       = $0.10/month
Total storage cost: ~$3.35/month for 3 days of backups
```

---

## 9. Snapshot Best Practices

| Practice | Why |
|----------|-----|
| **Tag snapshots** | Add name, date, environment, source volume |
| **Automate with DLM** | Don't rely on manual backups |
| **Cross-region copy** | True DR requires backups in a different region |
| **Test restores** | A backup you've never restored is not a backup |
| **Delete old snapshots** | Unused old snapshots cost money |
| **Snapshot before changes** | Always snapshot before major deployments or OS changes |
| **Check snapshot age** | Monitor oldest snapshot vs your RPO requirement |

---

## 10. Cheat Sheet

```bash
# Create snapshot
aws ec2 create-snapshot --volume-id <vol-id> --description "backup"

# List snapshots (yours)
aws ec2 describe-snapshots --owner-ids self --output table

# Wait for completion
aws ec2 wait snapshot-completed --snapshot-ids <snap-id>

# Create volume from snapshot
aws ec2 create-volume --snapshot-id <snap-id> --availability-zone us-east-1a --volume-type gp3

# Copy to another region
aws ec2 copy-snapshot --source-region us-east-1 --source-snapshot-id <snap-id> --region eu-west-1

# Share with account
aws ec2 modify-snapshot-attribute --snapshot-id <snap-id> --attribute createVolumePermission --operation-type add --user-ids <account-id>

# Delete snapshot
aws ec2 delete-snapshot --snapshot-id <snap-id>

# List DLM policies
aws dlm get-lifecycle-policies --output table
```

---

## 11. Common Interview Questions

**Q: Are EBS snapshots incremental?**
> Yes. The first snapshot stores all data. Subsequent snapshots store only the blocks that changed since the last snapshot. However, each snapshot is independent — you can restore from any single snapshot without needing earlier ones. AWS manages the block-level deduplication transparently.

**Q: Where are EBS snapshots stored?**
> Snapshots are stored in Amazon S3, but in AWS-managed buckets that are not visible in your S3 console. They appear in the EC2 Snapshots section. They are durably replicated across multiple AZs in the region automatically.

**Q: Can you create a snapshot of a running instance?**
> Yes. You can snapshot EBS volumes while the instance is running. For file system consistency, it's recommended to use application-level quiescing (flushing writes) before snapshotting databases. For general file system consistency, using tools like fsfreeze ensures a consistent state.

**Q: How would you automate snapshot management?**
> Use Amazon Data Lifecycle Manager (DLM). You define policies that target volumes by tag, specify creation schedules (e.g., every 24 hours at 2 AM), and retention rules (keep last 7 daily, 4 weekly). DLM creates and deletes snapshots automatically, eliminating manual work and ensuring compliance with backup SLAs.

**Q: What is the difference between a snapshot and an AMI?**
> A snapshot is a backup of a single EBS volume — just raw data. An AMI is built on top of snapshots and includes additional metadata: launch permissions, block device mappings, architecture information, and the virtualization type. An AMI is what you use to launch an EC2 instance; a snapshot is just storage.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
