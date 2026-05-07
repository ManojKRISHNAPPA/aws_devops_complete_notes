# AWS IAM — Practical Assignments

## Assignment 1 — User, Group, and Policy Setup

```bash
# 1. Create users
aws iam create-user --user-name dev-alice
aws iam create-user --user-name dev-bob
aws iam create-user --user-name ops-carol

# 2. Create groups
aws iam create-group --group-name Developers
aws iam create-group --group-name Operations

# 3. Create a custom policy: Developers can read EC2 and full S3
cat > dev-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2ReadOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:List*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3FullAccess",
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchRead",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics",
        "cloudwatch:DescribeAlarms"
      ],
      "Resource": "*"
    }
  ]
}
EOF

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DEV_POLICY=$(aws iam create-policy \
  --policy-name DeveloperPolicy \
  --policy-document file://dev-policy.json \
  --query 'Policy.Arn' --output text)
echo "Developer policy: $DEV_POLICY"

# 4. Attach policy to group
aws iam attach-group-policy --group-name Developers --policy-arn $DEV_POLICY
aws iam attach-group-policy --group-name Operations --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess

# 5. Add users to groups
aws iam add-user-to-group --user-name dev-alice --group-name Developers
aws iam add-user-to-group --user-name dev-bob --group-name Developers
aws iam add-user-to-group --user-name ops-carol --group-name Operations

# 6. Create console passwords
for USER in dev-alice dev-bob ops-carol; do
  aws iam create-login-profile \
    --user-name $USER \
    --password "TempPass@2026!" \
    --password-reset-required
  echo "Created login for: $USER"
done

# 7. Verify
echo "=== Developers Group ==="
aws iam get-group --group-name Developers --query 'Users[*].UserName' --output table

echo "=== Alice's Permissions ==="
aws iam list-attached-group-policies --group-name Developers --output table
```

---

## Assignment 2 — EC2 IAM Role Setup

```bash
# 1. Create trust policy
cat > ec2-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# 2. Create role
aws iam create-role \
  --role-name EC2S3AccessRole \
  --assume-role-policy-document file://ec2-trust.json \
  --description "Allows EC2 to read/write S3"

# 3. Create a least-privilege policy (specific bucket only)
BUCKET="my-app-bucket-$(date +%s)"
aws s3api create-bucket --bucket $BUCKET --region us-east-1

cat > s3-role-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::$BUCKET"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::$BUCKET/*"
    }
  ]
}
EOF

POLICY_ARN=$(aws iam create-policy \
  --policy-name EC2S3BucketPolicy \
  --policy-document file://s3-role-policy.json \
  --query 'Policy.Arn' --output text)

# 4. Attach policy to role
aws iam attach-role-policy --role-name EC2S3AccessRole --policy-arn $POLICY_ARN

# 5. Create instance profile
aws iam create-instance-profile --instance-profile-name EC2S3Profile
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2S3Profile \
  --role-name EC2S3AccessRole

# 6. Launch EC2 with this role
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query 'Parameter.Value' --output text)

INST=$(aws ec2 run-instances \
  --image-id $AMI --instance-type t2.micro --key-name my-learning-key \
  --iam-instance-profile Name=EC2S3Profile \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=iam-role-test}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $INST
IP=$(aws ec2 describe-instances --instance-ids $INST --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

# 7. Test on the instance — NO access keys needed!
ssh -i ~/.ssh/my-learning-key.pem ec2-user@$IP << EOF
# Verify role is attached
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Use AWS CLI without any configuration — role provides credentials!
aws s3 ls                                    # Lists buckets
aws s3 cp /etc/hostname s3://$BUCKET/hostname.txt   # Upload works
aws s3 ls s3://$BUCKET/                     # List works

# Test: try EC2 operations (should FAIL — role only allows S3)
aws ec2 describe-instances 2>&1 | head -5   # Should get AccessDenied
EOF
```

---

## Assignment 3 — Policy Simulation

```bash
# Test what permissions a user has before using them
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::${ACCOUNT_ID}:user/dev-alice \
  --action-names s3:GetObject s3:PutObject ec2:RunInstances iam:CreateUser \
  --resource-arns "*" \
  --query 'EvaluationResults[*].[EvalActionName,EvalDecision]' \
  --output table

# Expected output:
# s3:GetObject       →  allowed
# s3:PutObject       →  allowed
# ec2:RunInstances   →  implicitDeny
# iam:CreateUser     →  implicitDeny
```

---

## Assignment 4 — Audit IAM

```bash
# Generate credential report
aws iam generate-credential-report
sleep 5
aws iam get-credential-report --query 'Content' --output text | base64 -d > credential-report.csv
cat credential-report.csv

# Find users without MFA enabled
aws iam list-users --query 'Users[*].UserName' --output text | \
  tr '\t' '\n' | while read USER; do
    MFA=$(aws iam list-mfa-devices --user-name $USER --query 'MFADevices' --output text)
    if [ -z "$MFA" ]; then
      echo "⚠️  No MFA: $USER"
    else
      echo "✅ MFA enabled: $USER"
    fi
  done

# Find overly permissive policies (contain *)
aws iam list-policies --scope Local --query 'Policies[*].Arn' --output text | \
  tr '\t' '\n' | while read ARN; do
    VERSION=$(aws iam get-policy --policy-arn $ARN --query 'Policy.DefaultVersionId' --output text)
    DOC=$(aws iam get-policy-version --policy-arn $ARN --version-id $VERSION --query 'PolicyVersion.Document' --output json)
    if echo "$DOC" | grep -q '"Action": "\*"'; then
      echo "⚠️  Wildcard action: $ARN"
    fi
  done
```

---

## Assignment 5 — Cross-Account Access

```bash
# Account A creates a role that Account B can assume
# In Account A:
cat > cross-account-trust.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::<ACCOUNT_B_ID>:root"},
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {"aws:PrincipalTag/Role": "admin"}
    }
  }]
}
EOF

aws iam create-role \
  --role-name CrossAccountReadRole \
  --assume-role-policy-document file://cross-account-trust.json

aws iam attach-role-policy \
  --role-name CrossAccountReadRole \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# In Account B, assume the role:
aws sts assume-role \
  --role-arn "arn:aws:iam::<ACCOUNT_A_ID>:role/CrossAccountReadRole" \
  --role-session-name "audit-session" \
  --duration-seconds 3600

# Use the temporary credentials returned by assume-role
export AWS_ACCESS_KEY_ID=<returned-key-id>
export AWS_SECRET_ACCESS_KEY=<returned-secret>
export AWS_SESSION_TOKEN=<returned-token>

# Now you can query Account A's resources as a read-only role!
aws s3 ls   # Lists Account A's buckets
```

---

## Interview Practice Scenarios

### Scenario 1: Leaked Access Keys
> "You receive an alert that an IAM access key was committed to a public GitHub repo. What do you do?"

**Answer** (order matters):
1. **Immediately deactivate the key**: `aws iam update-access-key --access-key-id XXXXX --status Inactive`
2. **Check CloudTrail** for any activity with that key in the past 24-48 hours
3. **Delete the key**: `aws iam delete-access-key --access-key-id XXXXX`
4. **Investigate impact**: What resources did the key access? Was any data exfiltrated? Were new IAM users/keys created?
5. **Rotate affected resources**: Change any passwords/secrets the key had access to
6. **Create a new key** for the legitimate user if needed
7. **Post-mortem**: Add a git pre-commit hook to scan for secrets (git-secrets, detect-secrets)

### Scenario 2: EC2 Access to DynamoDB
> "Your application runs on EC2 and needs to read from a DynamoDB table. How do you configure this securely?"

**Answer**: Create an IAM role with least-privilege policy:
```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:GetItem", "dynamodb:Query", "dynamodb:Scan"],
  "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/my-table"
}
```
Attach the role to the EC2 instance profile. The application uses the AWS SDK — it automatically picks up credentials from the instance metadata service. No access keys in code, no rotation needed, credentials auto-expire and refresh.

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
