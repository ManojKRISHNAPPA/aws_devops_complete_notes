# AWS IAM — Complete Guide

## Table of Contents
1. [What is IAM?](#1-what-is-iam)
2. [IAM Core Components](#2-iam-core-components)
3. [IAM Users](#3-iam-users)
4. [IAM Groups](#4-iam-groups)
5. [IAM Roles](#5-iam-roles)
6. [IAM Policies](#6-iam-policies)
7. [Policy Evaluation Logic](#7-policy-evaluation-logic)
8. [IAM Best Practices](#8-iam-best-practices)
9. [IAM for EC2 (Instance Profiles)](#9-iam-for-ec2-instance-profiles)
10. [Service Control Policies (SCPs)](#10-service-control-policies-scps)
11. [IAM Access Analyzer](#11-iam-access-analyzer)
12. [Cheat Sheet](#12-cheat-sheet)
13. [Common Interview Questions](#13-common-interview-questions)

---

## 1. What is IAM?

**IAM (Identity and Access Management)** is the service that controls **who** can access AWS and **what** they can do.

```
IAM answers two questions:
──────────────────────────
Authentication: WHO are you?
   → Users, Roles, Federated identities

Authorization: WHAT can you do?
   → Policies (allow/deny specific actions on resources)
```

**Key IAM facts:**
- **Global service** — IAM is not region-specific; users and roles work everywhere
- **Free** — No charge for IAM usage
- **Default deny** — Everything is denied unless explicitly allowed
- **Principal** — The entity making a request (user, role, service)
- **ARN (Amazon Resource Name)** — Unique identifier for any AWS resource

```
ARN format:
arn:aws:iam::123456789012:user/john
arn:aws:iam::123456789012:role/MyEC2Role
arn:aws:s3:::my-bucket
arn:aws:ec2:us-east-1:123456789012:instance/i-xxxx
```

---

## 2. IAM Core Components

```
IAM Components:
───────────────
┌─────────────────────────────────────────────────────────────┐
│  IAM                                                        │
│                                                             │
│  Users  ─── can be in ───► Groups                          │
│    │                          │                             │
│    │                          │                             │
│    └── have attached ────►  Policies ◄── also attached to  │
│                               │             Roles           │
│                               │                             │
│                               │                             │
│                               ▼                             │
│                        Allow/Deny Actions                   │
│                        on Resources                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. IAM Users

An **IAM user** represents a person or application that interacts with AWS.

**Two types of access:**
- **Console access**: Username + password (for web console)
- **Programmatic access**: Access Key ID + Secret Access Key (for CLI/SDK)

```bash
# Create an IAM user
aws iam create-user --user-name john-developer

# Create login profile (console password)
aws iam create-login-profile \
  --user-name john-developer \
  --password "TempPassword@123" \
  --password-reset-required

# Create access keys (CLI/SDK access)
aws iam create-access-key --user-name john-developer
# IMPORTANT: Save the SecretAccessKey — it's only shown once!

# List users
aws iam list-users --output table

# Get user details
aws iam get-user --user-name john-developer

# Delete user (must remove access keys, policies, group memberships first)
aws iam delete-login-profile --user-name john-developer
aws iam delete-access-key --user-name john-developer --access-key-id AKIAIOSFODNN7EXAMPLE
aws iam delete-user --user-name john-developer
```

### MFA for Users

```bash
# Enable virtual MFA device
aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name john-developer-mfa \
  --outfile /tmp/mfa-qr.png \
  --bootstrap-method QRCodePNG

# Associate MFA (after scanning QR code and getting 2 codes)
aws iam enable-mfa-device \
  --user-name john-developer \
  --serial-number arn:aws:iam::123456789012:mfa/john-developer-mfa \
  --authentication-code1 123456 \
  --authentication-code2 654321
```

---

## 4. IAM Groups

A **group** is a collection of users. Attach policies to groups, not individual users.

```
Developer Group ──────────────► DeveloperPolicy
   ├── Alice                     (EC2 read/write, S3 read)
   ├── Bob
   └── Carol

Operations Group ─────────────► OpsPolicy
   ├── Dave                      (EC2 full, CloudWatch full)
   └── Eve

New developer joins? → Add to Developer Group → Gets correct permissions instantly!
Developer leaves? → Remove from group → Permissions removed instantly!
```

```bash
# Create a group
aws iam create-group --group-name Developers

# Add user to group
aws iam add-user-to-group \
  --user-name john-developer \
  --group-name Developers

# Attach policy to group
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess

# List groups for a user
aws iam list-groups-for-user --user-name john-developer

# List users in a group
aws iam get-group --group-name Developers

# Remove user from group
aws iam remove-user-from-group \
  --user-name john-developer \
  --group-name Developers
```

---

## 5. IAM Roles

A **role** is an IAM identity with specific permissions that can be **assumed** by trusted entities. Unlike users, roles have no permanent credentials — temporary credentials are issued when the role is assumed.

**Who can assume a role?**
- EC2 instances (and other AWS services)
- IAM users in the same or different account
- Federated users (SAML, OIDC, Google, Active Directory)
- AWS services (Lambda, ECS, etc.)

```
Role Use Cases:
───────────────
1. EC2 Instance Role:
   EC2 instance assumes a role → gets temporary credentials → accesses S3
   (No access keys in the code!)

2. Cross-Account Access:
   Account A user → assumes role in Account B → accesses Account B resources

3. Federation:
   Corporate user (Active Directory) → assumes AWS role → accesses AWS console

4. Lambda Execution Role:
   Lambda function → assumes role → writes to DynamoDB
```

### Create and Use an EC2 Role

```bash
# 1. Create trust policy (who can assume this role)
cat > ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# 2. Create the role
aws iam create-role \
  --role-name MyEC2S3Role \
  --assume-role-policy-document file://ec2-trust-policy.json \
  --description "Allows EC2 to access S3"

# 3. Attach permissions policy to the role
aws iam attach-role-policy \
  --role-name MyEC2S3Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 4. Create an instance profile (required to attach role to EC2)
aws iam create-instance-profile \
  --instance-profile-name MyEC2S3Profile

# 5. Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name MyEC2S3Profile \
  --role-name MyEC2S3Role

# 6. Launch EC2 with this role
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --iam-instance-profile Name=MyEC2S3Profile \
  ...

# 7. Now from within EC2, you can access S3 without any access keys!
# aws s3 ls  ← works automatically using role credentials
```

---

## 6. IAM Policies

**Policies** are JSON documents that define permissions. They are attached to users, groups, or roles.

### Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",            
      "Effect": "Allow",               
      "Action": [                      
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [                    
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {                   
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        }
      }
    }
  ]
}
```

**Required fields**:
- `Effect`: `Allow` or `Deny`
- `Action`: API action(s) (e.g., `s3:GetObject`, `ec2:*`, `*`)
- `Resource`: ARN of the resource(s)

**Optional fields**:
- `Sid`: Statement ID (label)
- `Condition`: Additional conditions for the rule

### Policy Types

| Type | Description |
|------|-------------|
| **AWS Managed** | Created and maintained by AWS (e.g., `AmazonS3FullAccess`) |
| **Customer Managed** | You create and manage them |
| **Inline** | Embedded directly in a user/group/role (not reusable) |

### Common Policy Examples

```bash
# Allow full EC2 access in specific region only
cat > ec2-region-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "ec2:*",
    "Resource": "*",
    "Condition": {
      "StringEquals": {"aws:RequestedRegion": "ap-south-1"}
    }
  }]
}
EOF

# Allow S3 access to specific bucket only
cat > s3-specific-bucket.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::company-reports"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::company-reports/*"
    }
  ]
}
EOF

# Create customer managed policy
aws iam create-policy \
  --policy-name S3BucketAccess \
  --policy-document file://s3-specific-bucket.json

# Attach to user
aws iam attach-user-policy \
  --user-name john-developer \
  --policy-arn arn:aws:iam::123456789012:policy/S3BucketAccess

# Attach to group
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::123456789012:policy/S3BucketAccess
```

---

## 7. Policy Evaluation Logic

AWS evaluates policies in a specific order:

```
Request comes in
      │
      ▼
1. Is there an explicit DENY anywhere?
   ↳ YES → DENY immediately (Deny always wins)
      │
      ▼ (no explicit deny)
2. Is there an explicit ALLOW?
   ↳ YES → ALLOW
      │
      ▼ (no explicit allow)
3. Default: DENY
```

```
Priority Order:
────────────────
1. Explicit Deny  (always wins, even if Allow exists)
2. Allow in policy
3. Default Deny (implicit)

Example:
SCP (org level):  Deny EC2 in us-west-2
IAM Policy:       Allow EC2 full access

Result: EC2 in us-west-2 is DENIED (SCP wins)
        EC2 in us-east-1 is ALLOWED
```

---

## 8. IAM Best Practices

| Practice | Why |
|----------|-----|
| **Never use root account** | Root has unlimited access, cannot be restricted |
| **Enable MFA everywhere** | Especially root and admin users |
| **Use roles for EC2/Lambda** | Avoid embedding access keys in code |
| **Principle of least privilege** | Grant only the permissions needed |
| **Use groups for permissions** | Easier to manage than per-user policies |
| **Rotate access keys** | Rotated credentials limit exposure window |
| **Monitor with CloudTrail** | Log all API calls for audit and detection |
| **Use IAM Access Analyzer** | Detect overly permissive policies |
| **Password policy** | Enforce length, complexity, rotation |
| **Don't share access keys** | One set per person/application |

### Set Password Policy

```bash
aws iam update-account-password-policy \
  --minimum-password-length 12 \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --require-numbers \
  --require-symbols \
  --max-password-age 90 \
  --password-reuse-prevention 5 \
  --allow-users-to-change-password
```

---

## 9. IAM for EC2 (Instance Profiles)

The most important pattern for EC2 security: **use roles, never access keys on EC2**.

```
BAD pattern (hardcoded credentials):
──────────────────────────────────────
# In your app code or ~/.aws/credentials on the EC2 instance:
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Problem: Keys can be stolen, leaked in git, hard to rotate

GOOD pattern (IAM role):
──────────────────────────
1. Create IAM role with the permissions your app needs
2. Attach role to EC2 at launch
3. AWS automatically provides temporary credentials via metadata service
4. Your app uses boto3/SDK/CLI and credentials are pulled automatically
# No keys in code, no keys to rotate, auto-expires and refreshes
```

```bash
# Check what role is attached to current EC2 instance
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Get the temporary credentials (for debugging)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/MyEC2S3Role

# Attach/replace role on running instance
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxx \
  --iam-instance-profile Name=MyEC2S3Profile
```

---

## 10. Service Control Policies (SCPs)

SCPs are organization-level policies in AWS Organizations. They define the **maximum permissions** for all accounts in an organizational unit.

```
AWS Organization
└── Root
    ├── SCP: DenyDeleteCloudTrail
    │
    ├── OU: Production
    │   ├── SCP: LimitToApprovedRegions (us-east-1, ap-south-1 only)
    │   ├── Account: prod-app
    │   └── Account: prod-db
    │
    └── OU: Development
        ├── SCP: RestrictEC2InstanceTypes (only t2.micro, t3.micro)
        └── Account: dev-team
```

SCPs do NOT grant permissions — they only restrict what IAM policies can grant.

---

## 11. IAM Access Analyzer

Analyzes your IAM policies and resource policies to find overly permissive access.

```bash
# Create an Access Analyzer for your account
aws accessanalyzer create-analyzer \
  --analyzer-name my-account-analyzer \
  --type ACCOUNT

# List findings (resources accessible outside your account)
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/my-account-analyzer \
  --output table
```

---

## 12. Cheat Sheet

```bash
# Users
aws iam create-user --user-name <name>
aws iam create-login-profile --user-name <name> --password <pass>
aws iam create-access-key --user-name <name>
aws iam list-users --output table

# Groups
aws iam create-group --group-name <name>
aws iam add-user-to-group --user-name <user> --group-name <group>
aws iam attach-group-policy --group-name <group> --policy-arn <arn>

# Roles
aws iam create-role --role-name <name> --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name <name> --policy-arn <arn>
aws iam create-instance-profile --instance-profile-name <name>
aws iam add-role-to-instance-profile --instance-profile-name <name> --role-name <role>

# Policies
aws iam create-policy --policy-name <name> --policy-document file://policy.json
aws iam attach-user-policy --user-name <user> --policy-arn <arn>
aws iam list-attached-user-policies --user-name <user>

# Simulate a policy
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/john \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/*
```

---

## 13. Common Interview Questions

**Q: What is the difference between an IAM user and an IAM role?**
> An IAM user is a permanent identity with long-term credentials (password, access keys). A role is a temporary identity that can be assumed by trusted entities — it issues short-term credentials (15 min to 12 hours). Users are for people/applications that need consistent long-term access. Roles are for EC2 instances, Lambda functions, cross-account access, and federation.

**Q: What is the principle of least privilege?**
> Grant only the minimum permissions needed to perform a task. If a Lambda function only reads from DynamoDB, give it only `dynamodb:GetItem` — not `dynamodb:*` and certainly not `*`. Over-privileged accounts increase blast radius if compromised.

**Q: How would you give an EC2 instance access to S3 without using access keys?**
> Create an IAM role with the S3 permissions needed, attach the role to the EC2 instance as an instance profile. The EC2 instance automatically receives temporary credentials via the Instance Metadata Service. Applications use the AWS SDK, which automatically retrieves these credentials. No keys in code, no rotation needed.

**Q: What is the difference between IAM policies and bucket policies?**
> IAM policies are attached to identities (users, roles, groups) and define what those identities can do. Bucket policies are attached to S3 buckets and define who can access that bucket. For cross-account access, bucket policies are required because IAM policies in Account A don't apply to resources in Account B. For same-account access, either approach works.

**Q: What happens if an IAM policy allows an action but an SCP denies it?**
> The SCP wins — the action is denied. SCPs define the maximum permissions allowed in an account. An explicit deny at the SCP level cannot be overridden by any IAM policy. The effective permission is the intersection of what's allowed by the SCP AND what's allowed by IAM policies.

**Q: What is an IAM role trust policy?**
> The trust policy (also called assume-role policy) is attached to an IAM role and defines which principals (services, accounts, users) are allowed to assume the role. It answers "who can use this role?". The permissions policy (attached separately) answers "what can this role do?". Both are required for a role to work.

**Q: How do you audit who has access to what in IAM?**
> Use IAM Access Analyzer to find resources shared externally. Generate Credential Reports for user-level auditing. Use IAM Policy Simulator to test effective permissions. Enable CloudTrail to log all IAM API calls. Use AWS Config to track policy changes over time.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
