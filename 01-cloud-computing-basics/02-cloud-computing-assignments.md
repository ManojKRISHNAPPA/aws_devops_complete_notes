# Cloud Computing Basics — Practical Assignments

## Table of Contents
1. [Assignment 1 — Create and Secure Your AWS Account](#1-assignment-1--create-and-secure-your-aws-account)
2. [Assignment 2 — Explore the AWS Management Console](#2-assignment-2--explore-the-aws-management-console)
3. [Assignment 3 — Install and Configure AWS CLI](#3-assignment-3--install-and-configure-aws-cli)
4. [Assignment 4 — Set Up Billing Alerts](#4-assignment-4--set-up-billing-alerts)
5. [Assignment 5 — Explore Regions and AZs via CLI](#5-assignment-5--explore-regions-and-azs-via-cli)
6. [Assignment 6 — AWS Pricing Calculator](#6-assignment-6--aws-pricing-calculator)
7. [Interview Practice Scenarios](#7-interview-practice-scenarios)

---

## 1. Assignment 1 — Create and Secure Your AWS Account

### Objective
Create an AWS account, enable MFA on the root user, and set up a billing budget.

### Steps

**Step 1: Create Account**
1. Go to `aws.amazon.com` → click "Create an AWS Account"
2. Provide email, password, account name
3. Choose **Personal** account type for learning
4. Enter credit card (needed even for free tier)
5. Verify identity by phone

**Step 2: Secure the Root User (Critical!)**
1. Log in to AWS Console with your root email
2. Click your account name (top right) → **Security Credentials**
3. Under "Multi-factor authentication (MFA)" → **Assign MFA device**
4. Choose **Authenticator app** (Google Authenticator or Authy)
5. Scan QR code, enter two consecutive codes to confirm

> **Why this matters**: The root account has unlimited access to everything including billing. A compromised root account can lead to massive unexpected bills. Always enable MFA.

**Step 3: Create an Admin IAM User (Best Practice)**
1. Go to **IAM** in the console
2. Click **Users** → **Create user**
3. Username: `admin-yourname`
4. Check "Provide user access to the AWS Management Console"
5. Attach policy: **AdministratorAccess**
6. Download credentials CSV
7. **Use this user for daily work, never root**

### Expected Outcome
- Root account has MFA enabled
- You have a non-root admin user for daily use
- You can log in with both accounts

---

## 2. Assignment 2 — Explore the AWS Management Console

### Objective
Familiarize yourself with the console layout, service navigation, and resource explorer.

### Steps

**Step 1: Console Layout**
1. Log in at `console.aws.amazon.com`
2. Notice the top navigation bar:
   - **Services menu** (top left) — all AWS services
   - **Search bar** — fastest way to find services
   - **Region selector** (top right) — very important!
   - **Account menu** (top right) — billing, credentials
   - **CloudShell icon** — browser-based terminal

**Step 2: Navigate to Key Services**
```
Open each of these and spend 2 minutes exploring the UI:
□ EC2 → note Dashboard, Instances, Volumes sections
□ S3 → note Buckets list view
□ VPC → note the network overview
□ IAM → note Users, Groups, Roles, Policies
□ CloudWatch → note Dashboards, Alarms
□ Billing → go to Cost Explorer
```

**Step 3: Try the Resource Groups & Tag Editor**
1. Search for "Resource Groups" in the console
2. Click **Tag Editor**
3. Select your region, all resource types
4. Click **Search resources**
5. See all resources you have created

**Step 4: CloudShell Exploration**
1. Click the **CloudShell** icon in the top navigation bar
2. A browser-based terminal opens (no setup needed)
3. Run:
   ```bash
   aws sts get-caller-identity    # Shows your current account/user
   aws ec2 describe-regions --output table   # Lists all regions
   ```

### Expected Outcome
- You can navigate to any service in under 30 seconds
- You understand the region selector (accidentally using wrong region is the #1 beginner mistake)

---

## 3. Assignment 3 — Install and Configure AWS CLI

### Objective
Install AWS CLI v2 on your local machine and configure it with IAM access keys.

### Steps

**Step 1: Create IAM Access Keys**
1. Log in as your admin IAM user (not root)
2. Go to **IAM → Users → your-username → Security credentials**
3. Click **Create access key**
4. Use case: **CLI**
5. Download the CSV — store it securely, you won't see the secret again!

**Step 2: Install AWS CLI**
```bash
# macOS (using Homebrew)
brew install awscli

# macOS (manual)
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Windows
# Download and run the MSI installer from AWS website

# Verify installation
aws --version
# Expected: aws-cli/2.x.x Python/3.x.x ...
```

**Step 3: Configure CLI**
```bash
aws configure
# AWS Access Key ID: [paste from CSV]
# AWS Secret Access Key: [paste from CSV]
# Default region name: ap-south-1      (or your preferred region)
# Default output format: json           (or table, text)
```

**Step 4: Verify Configuration**
```bash
# Test connectivity
aws sts get-caller-identity
# Should return your account ID, user ID, and ARN

# List S3 buckets (empty list is fine)
aws s3 ls

# List EC2 instances (empty is fine if none created)
aws ec2 describe-instances --output table
```

**Step 5: Configure Multiple Profiles (Optional but Useful)**
```bash
# Set up a named profile for a different account or region
aws configure --profile dev-account

# Use a specific profile
aws s3 ls --profile dev-account

# Set a profile as default for the session
export AWS_PROFILE=dev-account
```

### Expected Outcome
```
$ aws sts get-caller-identity
{
    "UserId": "AIDAIOSFODNN7EXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/admin-yourname"
}
```

---

## 4. Assignment 4 — Set Up Billing Alerts

### Objective
Set up cost budgets and billing alerts so you are notified before unexpected charges occur.

### Steps

**Step 1: Enable Billing Alerts in CloudWatch**
1. Log in as **root user** (billing settings require root)
2. Click your account name → **Account**
3. Scroll to "IAM user and role access to billing information"
4. Click **Edit** → check "Activate IAM Access" → Update

**Step 2: Create a Budget**
1. Go to **AWS Budgets** (search in console)
2. Click **Create budget**
3. Choose: **Cost budget**
4. Budget name: `monthly-learning-budget`
5. Budgeted amount: `$10` (or your comfortable limit)
6. Alert threshold: 80% of budget
7. Email notification: your email
8. Click **Create budget**

**Step 3: Free Tier Usage Alert**
1. In **Billing** → **Billing preferences**
2. Enable **AWS Free Tier alerts**
3. Enter your email address

**Step 4: Verify via CLI**
```bash
# List your budgets
aws budgets describe-budgets --account-id $(aws sts get-caller-identity --query Account --output text)
```

### Expected Outcome
- You receive an email when spending exceeds 80% of your budget
- Free tier alert sends notification when approaching limits

---

## 5. Assignment 5 — Explore Regions and AZs via CLI

### Objective
Use the AWS CLI to explore global infrastructure — regions, AZs, and available services.

### Steps

**Step 1: List All Regions**
```bash
aws ec2 describe-regions --output table
```
**Expected output:**
```
-----------------------------------------------
|              DescribeRegions               |
+---------------+-------------------+---------+
| OptInStatus   |   RegionName      | Endpoint|
+---------------+-------------------+---------+
| opt-in-not-required | ap-south-1 | ...      |
| opt-in-not-required | us-east-1  | ...      |
...
```

**Step 2: List Availability Zones in Your Region**
```bash
aws ec2 describe-availability-zones --region us-east-1 --output table
```

**Step 3: Check Which Services Are Available in a Region**
```bash
# List all services available in a region
aws ssm get-parameters-by-path \
  --path "/aws/service/global-infrastructure/regions/us-east-1/services" \
  --output json 2>/dev/null | python3 -c "
import sys, json
data = json.load(sys.stdin)
services = [p['Name'].split('/')[-1] for p in data['Parameters']]
print(f'Services in us-east-1: {len(services)}')
"
```

**Step 4: Compare Regions**
```bash
# Count AZs in different regions
for region in us-east-1 ap-south-1 eu-west-1; do
  count=$(aws ec2 describe-availability-zones --region $region --query 'length(AvailabilityZones)' --output text)
  echo "$region: $count AZs"
done
```

### Expected Outcome
- You understand why `us-east-1` is the default (most services, lowest prices)
- You can navigate regions and AZs programmatically

---

## 6. Assignment 6 — AWS Pricing Calculator

### Objective
Estimate costs for a basic architecture using the AWS Pricing Calculator.

### Architecture to Estimate
```
Simple 3-tier web application:
┌───────────────────────────────────────────┐
│  ALB (Application Load Balancer)          │
├───────────────────────────────────────────┤
│  2x EC2 t3.medium (web servers)           │
├───────────────────────────────────────────┤
│  RDS db.t3.medium MySQL (Multi-AZ)        │
├───────────────────────────────────────────┤
│  S3 (static assets, 50 GB)                │
└───────────────────────────────────────────┘
```

### Steps
1. Go to `calculator.aws`
2. Click **Create estimate**
3. Select region: **Asia Pacific (Mumbai)**
4. Add services one by one:
   - **EC2**: 2x t3.medium, Linux, On-Demand, 730 hours/month
   - **RDS**: db.t3.medium, MySQL, Multi-AZ, 730 hours/month
   - **S3**: 50 GB standard storage, 100,000 GET requests
   - **ALB**: 1 ALB, 5 LCU hours/month estimate
5. Review the **total monthly cost**
6. Change EC2 to Reserved (1 year) and note the savings
7. Export the estimate (Share → Save and share)

### Reflection Questions
- What is the biggest cost driver in this architecture?
- How much would you save with 1-year Reserved Instances on EC2?
- What would happen to cost if traffic doubled?

---

## 7. Interview Practice Scenarios

### Scenario 1: Architecture Decision
> "Your startup is deploying a new application. You expect 100 users on day 1, but potentially 10,000 users in 6 months. How would you design the infrastructure?"

**Answer framework**:
- Start with Auto Scaling Groups (handles growth automatically)
- Use On-Demand initially, switch to Reserved after traffic pattern stabilizes
- Deploy across 2+ AZs for high availability
- Use ALB for traffic distribution
- RDS Multi-AZ for database resilience

### Scenario 2: Cost Optimization
> "Your company spends $50,000/month on AWS. How would you approach reducing costs?"

**Answer framework**:
- Review AWS Cost Explorer for biggest spenders
- Identify idle/underutilized EC2 instances → rightsize or terminate
- Convert steady-state workloads from On-Demand to Reserved/Savings Plans
- Enable S3 lifecycle policies to move old data to Glacier
- Review data transfer costs (egress is expensive)
- Use Spot Instances for batch workloads

### Scenario 3: Compliance Requirement
> "Your client requires that their data never leave the EU. How do you ensure this in AWS?"

**Answer framework**:
- Deploy exclusively in EU regions (eu-west-1, eu-central-1, eu-south-1, eu-north-1)
- Use IAM SCPs (Service Control Policies) to prevent deploying in non-EU regions
- Enable CloudTrail to audit all API calls
- Use AWS Config rules to detect any non-EU resources
- Document your data residency approach for compliance audits

---

*Notes by ITkannadigaru | AWS Certification Series 2026*
