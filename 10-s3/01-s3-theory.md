# Amazon S3 — Complete Guide

## Table of Contents
1. [What is S3?](#1-what-is-s3)
2. [S3 Core Concepts](#2-s3-core-concepts)
3. [S3 Storage Classes](#3-s3-storage-classes)
4. [S3 Operations (CRUD)](#4-s3-operations-crud)
5. [S3 Versioning](#5-s3-versioning)
6. [S3 Lifecycle Policies](#6-s3-lifecycle-policies)
7. [S3 Security and Access Control](#7-s3-security-and-access-control)
8. [S3 Bucket Policies](#8-s3-bucket-policies)
9. [S3 Static Website Hosting](#9-s3-static-website-hosting)
10. [S3 Replication](#10-s3-replication)
11. [S3 Performance](#11-s3-performance)
12. [S3 Pricing](#12-s3-pricing)
13. [Cheat Sheet](#13-cheat-sheet)
14. [Common Interview Questions](#14-common-interview-questions)

---

## 1. What is S3?

**Amazon S3 (Simple Storage Service)** is AWS's **object storage** service. It stores files (objects) in flat containers called buckets.

```
S3 vs Traditional File System:
───────────────────────────────
File System                    S3
───────────                    ──────────────────
/home/user/photos/dog.jpg      s3://my-bucket/photos/dog.jpg
Folder hierarchy               Flat namespace (prefix simulation)
File system path               Object key (URL-based)
Local disk                     Distributed, global storage
Limited capacity               Virtually unlimited
You manage the disk            AWS manages everything
```

**Key S3 facts:**
- **Infinitely scalable** — no storage limits
- **Highly durable** — 99.999999999% (11 nines) durability
- **Highly available** — 99.99% SLA for Standard storage
- Objects can be **0 bytes to 5 TB** in size
- Max **single PUT** is 5 GB (use multipart upload for larger files)
- Buckets are **region-specific** but bucket names are **globally unique** across all accounts
- Objects are accessed via **HTTPS URLs** or AWS SDK

---

## 2. S3 Core Concepts

### Bucket
A **bucket** is a container for objects. Think of it like a top-level folder.

- Must have a globally unique name (no two S3 buckets in the world can have the same name)
- Created in a specific region
- Name rules: lowercase letters, numbers, hyphens; 3-63 characters; no underscores

### Object
An **object** is a file stored in S3. It consists of:
- **Key**: The object's name/path (e.g., `photos/2026/trip.jpg`)
- **Value**: The actual file content (bytes)
- **Metadata**: Key-value pairs (content-type, custom tags)
- **Version ID**: If versioning is enabled
- **Access Control**: Who can read/write this object

```
S3 Object anatomy:
──────────────────
Bucket: my-company-bucket
Key:    photos/2026/beach.jpg    ← The "path" — it's just a prefix, not a real folder
Value:  [binary data — the image file]
Size:   2.4 MB
Metadata:
  Content-Type: image/jpeg
  x-amz-meta-author: manoj
  x-amz-meta-date-taken: 2026-01-01
```

### S3 URL Formats

```
Path-style (older):
  https://s3.amazonaws.com/my-bucket/photos/beach.jpg

Virtual-hosted style (current standard):
  https://my-bucket.s3.amazonaws.com/photos/beach.jpg
  https://my-bucket.s3.us-east-1.amazonaws.com/photos/beach.jpg

Static website URL:
  http://my-bucket.s3-website-us-east-1.amazonaws.com/
```

---

## 3. S3 Storage Classes

Choose the storage class based on how often you access the data.

| Storage Class | Durability | Availability | Min Storage | Use Case |
|---------------|-----------|--------------|-------------|---------|
| **Standard** | 11 9's | 99.99% | None | Frequently accessed data |
| **Standard-IA** | 11 9's | 99.9% | 30 days | Infrequent access, rapid retrieval |
| **One Zone-IA** | 11 9's | 99.5% | 30 days | Infrequent, single AZ okay |
| **Glacier Instant** | 11 9's | 99.9% | 90 days | Archive, millisecond retrieval |
| **Glacier Flexible** | 11 9's | 99.99% | 90 days | Archive, minutes-hours retrieval |
| **Glacier Deep Archive** | 11 9's | 99.99% | 180 days | Compliance archive, 12-hour retrieval |
| **Intelligent-Tiering** | 11 9's | 99.9% | None | Unknown/changing access patterns |

```
Cost (expensive to cheap):
Standard → Standard-IA → One Zone-IA → Glacier Instant → Glacier Flexible → Deep Archive

Access Speed (fast to slow):
Standard = Standard-IA = One Zone-IA (milliseconds)
Glacier Instant (milliseconds)
Glacier Flexible (minutes to hours)
Deep Archive (12-48 hours)
```

> **Intelligent-Tiering** is the "set it and forget it" option. S3 automatically moves objects between access tiers based on usage. No retrieval fees. Small monitoring fee. Best when you don't know the access pattern.

---

## 4. S3 Operations (CRUD)

### Create Bucket and Upload

```bash
# Create a bucket (bucket name must be globally unique)
aws s3api create-bucket \
  --bucket my-unique-bucket-$(date +%s) \
  --region us-east-1

# For regions other than us-east-1, add CreateBucketConfiguration:
aws s3api create-bucket \
  --bucket my-bucket-mumbai \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

# Set bucket variable
BUCKET=my-bucket-name   # replace with your bucket name

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket $BUCKET \
  --versioning-configuration Status=Enabled
```

### Upload Objects

```bash
# Upload a single file
aws s3 cp file.txt s3://$BUCKET/file.txt

# Upload with specific storage class
aws s3 cp archive.zip s3://$BUCKET/archives/archive.zip --storage-class GLACIER

# Upload entire directory (recursive)
aws s3 cp ./my-website/ s3://$BUCKET/ --recursive

# Sync local directory to S3 (only uploads new/changed files)
aws s3 sync ./my-website/ s3://$BUCKET/ --delete
# --delete removes S3 objects not in local directory

# Upload with metadata
aws s3 cp image.jpg s3://$BUCKET/images/image.jpg \
  --metadata '{"author":"manoj","project":"aws-notes"}' \
  --content-type "image/jpeg"
```

### Download Objects

```bash
# Download a file
aws s3 cp s3://$BUCKET/file.txt ./file.txt

# Download entire bucket/prefix
aws s3 cp s3://$BUCKET/images/ ./local-images/ --recursive

# Sync S3 to local
aws s3 sync s3://$BUCKET/ ./local-copy/
```

### List Objects

```bash
# List all buckets
aws s3 ls

# List objects in a bucket
aws s3 ls s3://$BUCKET/

# List with details (size, date)
aws s3 ls s3://$BUCKET/ --human-readable --summarize

# List all objects recursively
aws s3 ls s3://$BUCKET/ --recursive

# Using s3api for more detail
aws s3api list-objects-v2 \
  --bucket $BUCKET \
  --query 'Contents[*].[Key,Size,LastModified]' \
  --output table
```

### Delete Objects

```bash
# Delete a single file
aws s3 rm s3://$BUCKET/file.txt

# Delete all files with a prefix
aws s3 rm s3://$BUCKET/temp/ --recursive

# Delete all files in bucket (before deleting bucket)
aws s3 rm s3://$BUCKET/ --recursive

# Delete the bucket itself (must be empty)
aws s3api delete-bucket --bucket $BUCKET
```

---

## 5. S3 Versioning

When versioning is enabled, S3 keeps all versions of an object. Overwrites and deletes don't truly destroy data.

```
Versioning Example:
───────────────────
Upload report.pdf (v1) → VersionId: ABC123
Modify and upload report.pdf (v2) → VersionId: DEF456
Modify and upload report.pdf (v3) → VersionId: GHI789

ls s3://bucket/report.pdf shows only latest (v3)
But all 3 versions are stored and retrievable!

Delete report.pdf → Creates a "delete marker" (not actual deletion)
ls shows nothing, but versions ABC123, DEF456, GHI789 still exist!
```

```bash
# Enable versioning
aws s3api put-bucket-versioning \
  --bucket $BUCKET \
  --versioning-configuration Status=Enabled

# List all versions of a file
aws s3api list-object-versions \
  --bucket $BUCKET \
  --prefix report.pdf

# Download a specific version
aws s3api get-object \
  --bucket $BUCKET \
  --key report.pdf \
  --version-id ABC123 \
  old-report.pdf

# Delete a specific version permanently
aws s3api delete-object \
  --bucket $BUCKET \
  --key report.pdf \
  --version-id ABC123
```

---

## 6. S3 Lifecycle Policies

Automate transitioning objects between storage classes and deleting old versions.

```
Lifecycle Example:
──────────────────
0 days:    Upload to Standard storage
30 days:   Automatically move to Standard-IA
90 days:   Automatically move to Glacier Flexible
365 days:  Automatically delete
```

```bash
# Create lifecycle policy
aws s3api put-bucket-lifecycle-configuration \
  --bucket $BUCKET \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "ArchiveAndExpire",
        "Status": "Enabled",
        "Prefix": "logs/",
        "Transitions": [
          {
            "Days": 30,
            "StorageClass": "STANDARD_IA"
          },
          {
            "Days": 90,
            "StorageClass": "GLACIER"
          }
        ],
        "Expiration": {
          "Days": 365
        },
        "NoncurrentVersionExpiration": {
          "NoncurrentDays": 30
        }
      }
    ]
  }'

# View lifecycle configuration
aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET
```

---

## 7. S3 Security and Access Control

### Public Access Block (Default: ON)

AWS now blocks all public access by default — a great security default. To host a public website or allow public access, you must explicitly unblock.

```bash
# Check current public access block settings
aws s3api get-public-access-block --bucket $BUCKET

# Allow public access (needed for static websites or public data)
aws s3api delete-public-access-block --bucket $BUCKET
# Or selectively configure:
aws s3api put-public-access-block \
  --bucket $BUCKET \
  --public-access-block-configuration \
    BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
```

### ACLs (Legacy — avoid for new setups)
- Object-level control
- Recommended: Disable ACLs and use bucket policies instead

### Bucket Policies (Preferred method)
- JSON-based policies attached to the bucket
- Can allow/deny specific actions for specific principals (accounts, users, roles)
- More powerful and manageable than ACLs

---

## 8. S3 Bucket Policies

```bash
# Make specific prefix publicly readable (e.g., static website)
aws s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "PublicReadGetObject",
        "Effect": "Allow",
        "Principal": "*",
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::'$BUCKET'/public/*"
      }
    ]
  }'

# Allow a specific IAM role to access the bucket
aws s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::123456789012:role/MyAppRole"
        },
        "Action": ["s3:GetObject", "s3:PutObject"],
        "Resource": "arn:aws:s3:::'$BUCKET'/*"
      }
    ]
  }'

# Deny all non-HTTPS access (force encryption in transit)
aws s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:*",
        "Resource": ["arn:aws:s3:::'$BUCKET'", "arn:aws:s3:::'$BUCKET'/*"],
        "Condition": {
          "Bool": {"aws:SecureTransport": "false"}
        }
      }
    ]
  }'
```

---

## 9. S3 Static Website Hosting

S3 can host static websites (HTML, CSS, JS — but not server-side code).

```bash
# 1. Create bucket with your domain name as bucket name (optional for CloudFront)
BUCKET=my-static-site.example.com

aws s3api create-bucket --bucket $BUCKET --region us-east-1

# 2. Disable block public access
aws s3api delete-public-access-block --bucket $BUCKET

# 3. Enable static website hosting
aws s3api put-bucket-website \
  --bucket $BUCKET \
  --website-configuration '{
    "IndexDocument": {"Suffix": "index.html"},
    "ErrorDocument": {"Key": "error.html"}
  }'

# 4. Add public read policy
aws s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::'$BUCKET'/*"
    }]
  }'

# 5. Upload website files
echo "<h1>Hello from S3!</h1>" > index.html
aws s3 cp index.html s3://$BUCKET/ --content-type text/html

# 6. Access at:
echo "http://$BUCKET.s3-website-us-east-1.amazonaws.com"
```

---

## 10. S3 Replication

### CRR (Cross-Region Replication)
- Replicates objects to a bucket in a different region
- Use cases: disaster recovery, compliance (data in EU), lower latency for global users

### SRR (Same-Region Replication)
- Replicates within the same region to a different bucket
- Use cases: aggregate logs from multiple accounts, production to test environment

```bash
# Enable versioning on both buckets (required)
aws s3api put-bucket-versioning --bucket $SOURCE --versioning-configuration Status=Enabled
aws s3api put-bucket-versioning --bucket $DEST --versioning-configuration Status=Enabled --region eu-west-1

# Create replication configuration
aws s3api put-bucket-replication \
  --bucket $SOURCE \
  --replication-configuration '{
    "Role": "arn:aws:iam::ACCOUNT_ID:role/S3ReplicationRole",
    "Rules": [{
      "Status": "Enabled",
      "Destination": {
        "Bucket": "arn:aws:s3:::my-dest-bucket",
        "StorageClass": "STANDARD_IA"
      }
    }]
  }'
```

---

## 11. S3 Performance

- S3 can handle **3,500 PUT/COPY/POST/DELETE** and **5,500 GET/HEAD requests per second per prefix**
- For large file uploads (>100MB), use **multipart upload** (better speed, resume on failure)
- Use **Transfer Acceleration** for faster uploads from distant locations (uses CloudFront edge locations)

```bash
# Multipart upload example (for large files)
aws s3 cp large-file.zip s3://$BUCKET/ \
  --expected-size 5000000000 \
  --multipart-threshold 64MB

# Enable Transfer Acceleration
aws s3api put-bucket-accelerate-configuration \
  --bucket $BUCKET \
  --accelerate-configuration Status=Enabled

# Upload using accelerated endpoint
aws s3 cp large-file.zip s3://$BUCKET/ \
  --endpoint-url https://s3-accelerate.amazonaws.com
```

---

## 12. S3 Pricing

| Component | Price (us-east-1, approx) |
|-----------|--------------------------|
| Standard storage | $0.023/GB-month |
| Standard-IA storage | $0.0125/GB-month |
| Glacier Flexible | $0.004/GB-month |
| Glacier Deep Archive | $0.00099/GB-month |
| PUT/COPY/POST requests | $0.005 per 1,000 |
| GET requests | $0.0004 per 1,000 |
| Data transfer OUT | $0.09/GB (first 10TB) |
| Data transfer IN | Free |

> **Free within same region**: Data transfer between S3 and EC2 in the same region is free.

---

## 13. Cheat Sheet

```bash
# Bucket operations
aws s3 ls                                          # List buckets
aws s3api create-bucket --bucket <name> --region us-east-1
aws s3api delete-bucket --bucket <name>

# Object operations
aws s3 cp <local> s3://<bucket>/<key>              # Upload
aws s3 cp s3://<bucket>/<key> <local>              # Download
aws s3 mv s3://<bucket>/<key1> s3://<bucket>/<key2> # Move
aws s3 rm s3://<bucket>/<key>                      # Delete
aws s3 ls s3://<bucket>/ --recursive --human-readable

# Sync
aws s3 sync ./local/ s3://<bucket>/prefix/ --delete

# Presigned URL (temporary access, 1 hour)
aws s3 presign s3://<bucket>/<key> --expires-in 3600

# Storage class
aws s3 cp file.txt s3://<bucket>/ --storage-class STANDARD_IA

# Versioning
aws s3api put-bucket-versioning --bucket <name> --versioning-configuration Status=Enabled
aws s3api list-object-versions --bucket <name> --prefix <key>
```

---

## 14. Common Interview Questions

**Q: What is the difference between S3 Standard and S3 Standard-IA?**
> Both offer the same durability (11 nines) and similar availability. Standard is for frequently accessed data — no retrieval fee. Standard-IA is cheaper for storage but charges a per-GB retrieval fee and has a minimum 30-day storage charge. Use Standard-IA for files accessed less than once a month.

**Q: What is S3 versioning?**
> Versioning keeps all versions of an object. When you upload an object with the same key, the old version is preserved with a version ID. Deletes create a "delete marker" rather than permanent deletion. Versioning protects against accidental overwrites and deletes but increases storage costs.

**Q: How do you make S3 data public?**
> You need three steps: (1) Disable the bucket's Block Public Access settings, (2) Add a bucket policy allowing `s3:GetObject` for `Principal: "*"`, and (3) Ensure there's no conflicting SCP. AWS makes all S3 buckets private by default with Block Public Access enabled.

**Q: What is the maximum size of an S3 object?**
> Maximum object size is 5 TB. A single PUT request can upload up to 5 GB. For files larger than 5 GB (recommended threshold: 100 MB), use multipart upload which divides the file into parts (5 MB to 5 GB each) uploaded in parallel and assembled by S3.

**Q: What is the difference between S3 bucket policy and IAM policy for S3 access?**
> IAM policies are attached to users/roles/groups and define what that principal can do across AWS services. Bucket policies are attached to S3 buckets and define who can access that specific bucket. For cross-account access, bucket policies are required. For same-account access, either works. For public access, bucket policies are used.

**Q: What is S3 CRR and when would you use it?**
> CRR (Cross-Region Replication) automatically copies objects from a source bucket to a destination bucket in a different region. Use it for: disaster recovery (data survives regional outage), compliance (copy EU data to another EU region), reducing latency (copy to a region closer to users), or security auditing (replicate to a compliance account).

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
