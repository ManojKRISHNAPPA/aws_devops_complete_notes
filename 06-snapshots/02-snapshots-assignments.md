# Snapshots — Practical Assignments

## Assignment 1 — Automated Snapshot Policy

### Objective
Set up Data Lifecycle Manager to automatically take daily snapshots of production volumes.

### Steps

```bash
# Step 1: Tag your production volume
aws ec2 create-tags \
  --resources vol-xxxxxxxxxxxxxxxxx \
  --tags Key=Environment,Value=production

# Step 2: Get the DLM role ARN
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DLM_ROLE="arn:aws:iam::${ACCOUNT_ID}:role/AWSDataLifecycleManagerDefaultRole"

# Create the default DLM role if it doesn't exist
aws iam create-service-linked-role --aws-service-name dlm.amazonaws.com 2>/dev/null || true

# Step 3: Create the lifecycle policy
POLICY_ID=$(aws dlm create-lifecycle-policy \
  --description "Daily backup for production volumes" \
  --state ENABLED \
  --execution-role-arn $DLM_ROLE \
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
      "RetainRule": {"Count": 7},
      "TagsToAdd": [{"Key": "CreatedBy", "Value": "DLM"}],
      "CopyTags": true
    }]
  }' \
  --query 'PolicyId' --output text)

echo "DLM Policy created: $POLICY_ID"

# Step 4: Verify
aws dlm get-lifecycle-policy --policy-id $POLICY_ID --output table
```

---

## Assignment 2 — Disaster Recovery Simulation

### Objective
Simulate a disaster scenario where the original volume is destroyed, then recover from a snapshot.

```bash
# 1. Create volume and write critical data
VOL=$(aws ec2 create-volume --volume-type gp3 --size 5 --availability-zone us-east-1a --query 'VolumeId' --output text)
aws ec2 wait volume-available --volume-ids $VOL

# Attach and write data (on EC2 instance)
aws ec2 attach-volume --volume-id $VOL --instance-id $INSTANCE_ID --device /dev/sdg
# SSH in and write data...

# 2. Take snapshot
SNAP=$(aws ec2 create-snapshot \
  --volume-id $VOL \
  --description "DR test snapshot" \
  --query 'SnapshotId' --output text)
aws ec2 wait snapshot-completed --snapshot-ids $SNAP
echo "Snapshot taken: $SNAP"

# 3. DISASTER! Delete the volume
aws ec2 detach-volume --volume-id $VOL
aws ec2 wait volume-available --volume-ids $VOL
aws ec2 delete-volume --volume-id $VOL
echo "DISASTER: Volume deleted!"

# 4. Recover from snapshot
RECOVERED=$(aws ec2 create-volume \
  --snapshot-id $SNAP \
  --availability-zone us-east-1a \
  --volume-type gp3 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=dr-recovered}]' \
  --query 'VolumeId' --output text)
aws ec2 wait volume-available --volume-ids $RECOVERED
echo "Recovery successful: $RECOVERED"

# 5. Attach and verify data
aws ec2 attach-volume --volume-id $RECOVERED --instance-id $INSTANCE_ID --device /dev/sdh
# SSH in and verify: all data is there!
```

---

## Interview Practice — Snapshot Scenarios

### Scenario: Database Backup
> "How would you set up automated, tested backups for a MySQL database on EC2?"

**Answer**:
1. Create an EBS volume for MySQL data (`/var/lib/mysql` on separate volume, not root)
2. Tag the volume: `Environment=production`, `Type=database`
3. Create a DLM policy: daily snapshots at 2 AM, retain 7 daily + 4 weekly
4. Add cross-region copy in DLM for DR (another region)
5. Monthly restore test: create volume from snapshot, mount on test instance, run MySQL, verify data integrity
6. Automate the restore test with a Lambda function

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
