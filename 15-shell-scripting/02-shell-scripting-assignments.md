# Shell Scripting — Assignments

## Assignment 1: Server Health Check Script

**Build a complete health check script:**

```bash
#!/usr/bin/env bash
# health-check.sh
# Requirements:
# 1. Check CPU load average — alert if > 80% of cores
# 2. Check memory usage — alert if > 85%
# 3. Check disk usage on all partitions — alert if > 80%
# 4. Check if critical services are running: nginx, sshd, cron
# 5. Check if port 80 and 443 are listening
# 6. Output: green "OK" for healthy, red "CRITICAL" for issues
# 7. Exit code 0 if all OK, 1 if any alert
# 8. Accept -v flag for verbose output
# 9. Send email if any CRITICAL condition (use mail command)
# 10. Log results to /var/log/health-check.log with timestamp
```

**Expected output:**
```
[2024-01-15 10:00:00] Health Check Report for web-01
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ CPU Load: 1.2 (4 cores, 30% utilized)
✅ Memory: 65% (used 5.2G / 8G)
✅ Disk /: 45% (used 40G / 100G)
❌ Disk /var: 92% CRITICAL - Alert sent!
✅ nginx: RUNNING (PID: 1234)
✅ sshd: RUNNING (PID: 567)
✅ Port 80: LISTENING
✅ Port 443: LISTENING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Summary: 1 CRITICAL issue found
```

---

## Assignment 2: Automated Backup Script

**Requirements:**
```bash
#!/usr/bin/env bash
# backup.sh — Backup specified directories to S3
#
# Usage: backup.sh -s <source_dirs> -b <s3_bucket> [-r <retention_days>] [-d]
#
# Features:
# 1. Accept -s for source directory (can specify multiple: -s /etc -s /var/www)
# 2. Accept -b for S3 bucket name
# 3. Accept -r for retention days (default 30)
# 4. Accept -d for dry run mode
# 5. Create gzipped tar archive: backup-YYYYMMDD-HHMMSS.tar.gz
# 6. Calculate and verify MD5 checksum
# 7. Upload to S3 with proper path: s3://bucket/hostname/date/
# 8. Delete local backup file after successful upload
# 9. Delete S3 backups older than retention days
# 10. Log all actions to /var/log/backup.log
# 11. Send Slack webhook notification on success/failure
# 12. Lock file to prevent concurrent runs

# Test cases:
# ./backup.sh -s /etc -s /var/www -b my-backup-bucket -r 30
# ./backup.sh -s /etc -b my-backup-bucket -d  (dry run)
# ./backup.sh  (should show usage/help)
```

---

## Assignment 3: Log Parser and Reporter

**Task:** Parse nginx access logs and generate a report

```bash
#!/usr/bin/env bash
# log-report.sh — Generate nginx access log report
#
# Input: /var/log/nginx/access.log (or specified file)
# Output: HTML or text report with:
#
# 1. Total requests
# 2. Requests by status code (200, 404, 500, etc.)
# 3. Top 10 IP addresses
# 4. Top 10 URLs
# 5. Requests per hour graph (ASCII bar chart)
# 6. Average response size
# 7. Error rate percentage
# 8. Unique visitors count
#
# Usage: ./log-report.sh [--log-file path] [--format text|html] [--since "1 hour ago"]
```

**ASCII bar chart example:**
```
Requests per hour:
10:00 ██████████████████████████████ 150
11:00 ████████████████████████████████████████ 200
12:00 ████████████████████████████████████████████ 220
13:00 ██████████████████ 90
```

---

## Assignment 4: Multi-Server Deployment Script

**Build a rolling deployment script:**

```bash
#!/usr/bin/env bash
# deploy.sh — Rolling deployment to multiple servers
#
# Usage: ./deploy.sh --app myapp --version 2.1.0 --env production
#
# Server list: servers.txt (one per line)
# Format: server-ip user deploy-key
#
# Workflow:
# 1. Read server list from file
# 2. Validate all servers are reachable
# 3. For each server (in batches of 2):
#    a. Print "Deploying to server-N..."
#    b. SSH and pull new Docker image
#    c. SSH and restart container
#    d. Wait for health check to pass (max 60s, retry every 5s)
#    e. If health check fails: rollback this server, stop deployment
# 4. Print deployment summary
# 5. Total time taken
#
# Handle:
# - Server unreachable → skip with warning
# - Health check timeout → rollback and abort
# - CTRL+C → graceful rollback of in-progress deploys

# servers.txt format:
# 10.0.1.10 ubuntu /path/to/key.pem
# 10.0.1.11 ubuntu /path/to/key.pem
# 10.0.1.12 ubuntu /path/to/key.pem
```

---

## Assignment 5: Environment Setup Script

**Build a one-command developer environment setup:**

```bash
#!/usr/bin/env bash
# setup-dev-env.sh — Set up complete DevOps development environment
# Idempotent — safe to run multiple times
#
# Install and configure:
# 1. Check OS (Ubuntu/macOS) and use appropriate package manager
# 2. Install: git, curl, wget, jq, tree, htop
# 3. Install Docker (if not present)
#    - Add user to docker group
#    - Start and enable docker service
# 4. Install kubectl (latest stable)
# 5. Install Helm
# 6. Install AWS CLI v2
# 7. Install Terraform (latest)
# 8. Install Python 3 + pip
# 9. Configure git (prompt for name and email)
# 10. Configure ~/.bashrc with aliases:
#     alias k='kubectl'
#     alias tf='terraform'
#     alias dc='docker-compose'
# 11. Print summary of installed versions
# 12. Run self-test (verify each tool works)
#
# Expected output:
# ✅ git: 2.39.0
# ✅ docker: 24.0.5
# ✅ kubectl: v1.29.0
# ✅ helm: v3.13.0
# ✅ aws: 2.15.0
# ✅ terraform: 1.7.0
```

---

## Assignment 6: Monitoring and Alerting Script

**Build a continuous monitoring daemon:**

```bash
#!/usr/bin/env bash
# monitor.sh — Continuous system monitoring
#
# Runs in background, checks every 60 seconds
# Sends alerts via Slack webhook (set SLACK_WEBHOOK env var)
#
# Monitors:
# 1. CPU > 90% for 3 consecutive checks → ALERT
# 2. Memory > 90% → ALERT
# 3. Disk > 85% → ALERT
# 4. Service down (nginx, mysql) → ALERT
# 5. Failed SSH login attempts > 10 in last minute → SECURITY ALERT
# 6. New user login outside business hours → SECURITY ALERT
#
# Alert format:
# {
#   "text": "🚨 ALERT: CPU usage is 95% on web-01\nTime: 2024-01-15 10:00:00"
# }
#
# Usage:
# Start:  ./monitor.sh start
# Stop:   ./monitor.sh stop
# Status: ./monitor.sh status
# Log:    ./monitor.sh log
```

---

## Interview Assignment: Parse and Process CSV Data

**Scenario:** You receive a CSV file of EC2 instances and need to process it.

```bash
# instances.csv
InstanceId,Name,State,Type,PrivateIP,PublicIP,LaunchTime
i-0a1b2c3d,web-01,running,t3.micro,10.0.1.10,54.1.2.3,2024-01-10
i-0e5f6g7h,web-02,running,t3.micro,10.0.1.11,54.1.2.4,2024-01-10
i-0i9j0k1l,db-01,running,t3.large,10.0.2.10,,2024-01-10
i-0m2n3o4p,staging,stopped,t3.small,10.0.3.10,,2024-01-15
```

**Tasks (write a single script `process-instances.sh`):**

```bash
# 1. Count running vs stopped instances
# 2. List all instances with no public IP
# 3. Calculate running instances older than 7 days
# 4. Generate report:
#    - Grouped by state
#    - Show instance type distribution
# 5. Export running instances to JSON format
# 6. Send start command for stopped instances (dry run flag)

# Expected output:
Running: 3
Stopped: 1

Instances without public IP:
  - db-01 (i-0i9j0k1l)
  - staging (i-0m2n3o4p)

Instance type distribution:
  t3.micro  : 2
  t3.small  : 1
  t3.large  : 1

Stopped instances (would start with --execute):
  DRY RUN: aws ec2 start-instances --instance-ids i-0m2n3o4p
```

---

## Workflow: CI/CD Deployment Pipeline Script

**Build a complete deployment pipeline script used in Jenkins:**

```bash
#!/usr/bin/env bash
# pipeline.sh — CI/CD Pipeline orchestrator
#
# Stages:
# 1. CHECKOUT    - Verify git state
# 2. TEST        - Run unit tests, collect coverage
# 3. BUILD       - Build Docker image
# 4. SCAN        - Security scan with trivy
# 5. PUSH        - Push to ECR
# 6. DEPLOY_DEV  - Deploy to dev environment
# 7. SMOKE_TEST  - Run smoke tests against dev
# 8. DEPLOY_PROD - Deploy to production (if branch = main)
#
# Each stage:
# - Logs start time
# - Reports duration on completion
# - On failure: sends Slack alert, logs details, exits 1
# - On success: logs "✅ Stage X passed"
#
# Final output:
# ═══════════════════════════════════
# Pipeline Summary: myapp v1.2.3
# ═══════════════════════════════════
# ✅ checkout   : 2s
# ✅ test       : 45s (coverage: 87%)
# ✅ build      : 120s (image: 245MB)
# ✅ scan       : 30s (0 critical CVEs)
# ✅ push       : 15s
# ✅ deploy_dev : 60s
# ✅ smoke_test : 20s
# ✅ deploy_prod: 60s
# ═══════════════════════════════════
# Total: 352s | Status: SUCCESS
```

---

## Cheat Sheet for Common Script Patterns

```bash
# Check if running as root
[[ $EUID -eq 0 ]] || { echo "Must run as root"; exit 1; }

# Check required commands
for cmd in curl jq aws docker; do
    command -v $cmd &>/dev/null || { echo "Required: $cmd"; exit 1; }
done

# Confirm before destructive operation
read -rp "Are you sure? [y/N] " confirm
[[ $confirm == [yY] ]] || exit 0

# Retry with backoff
for i in {1..5}; do
    curl -s http://api && break
    sleep $((i * 2))
done

# Progress indicator
while sleep 1; do echo -n "."; done &
DOTS_PID=$!
do_long_task
kill $DOTS_PID

# Validate IP address
[[ $IP =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$ ]] || die "Invalid IP"

# Parse JSON without jq (basic)
python3 -c "import json,sys; d=json.load(sys.stdin); print(d['key'])"

# Colorized output
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; NC='\033[0m'
echo -e "${RED}ERROR${NC}: something went wrong"
echo -e "${GREEN}OK${NC}: all good"
```
