# Shell Scripting for DevOps — Complete Guide

## Table of Contents
1. [Why Shell Scripting?](#1-why-shell-scripting)
2. [Script Basics](#2-script-basics)
3. [Variables and Data Types](#3-variables-and-data-types)
4. [Conditionals](#4-conditionals)
5. [Loops](#5-loops)
6. [Functions](#6-functions)
7. [Input and Output](#7-input-and-output)
8. [String Manipulation](#8-string-manipulation)
9. [Arrays](#9-arrays)
10. [Error Handling](#10-error-handling)
11. [File Operations](#11-file-operations)
12. [Regular Expressions](#12-regular-expressions)
13. [Advanced Patterns](#13-advanced-patterns)
14. [Real-World DevOps Scripts](#14-real-world-devops-scripts)
15. [Common Interview Questions](#15-common-interview-questions)

---

## 1. Why Shell Scripting?

Shell scripting is a core DevOps skill for automating repetitive tasks:

```
DevOps Automation with Shell Scripts:
──────────────────────────────────────────────────────────
• Server provisioning and configuration       → bootstrap.sh
• Application deployment                      → deploy.sh
• Health checks and monitoring                → health-check.sh
• Log rotation and cleanup                    → cleanup.sh
• Database backup and restore                 → backup.sh
• CI/CD pipeline steps                        → build.sh, test.sh
• User account management                     → create-user.sh
• Automated reporting                         → daily-report.sh
• Environment setup                           → setup-dev-env.sh
• Infrastructure maintenance                  → maintenance.sh
```

---

## 2. Script Basics

### Shebang and Execution

```bash
#!/bin/bash          # Use bash (most common for DevOps)
#!/bin/sh            # Use POSIX sh (more portable)
#!/usr/bin/env bash  # Find bash in PATH (recommended for portability)
#!/usr/bin/env python3  # Can use other interpreters too

# Make executable and run
chmod +x script.sh
./script.sh
bash script.sh       # Run without chmod

# Debug mode
bash -x script.sh    # Print each command before executing
bash -n script.sh    # Syntax check only (no execution)
bash -v script.sh    # Print each line as read
```

### Script Header Best Practice

```bash
#!/usr/bin/env bash
set -euo pipefail  # CRUCIAL for robust scripts
# -e: exit on error (any command fails → script exits)
# -u: exit on undefined variable
# -o pipefail: pipe fails if any command in pipe fails

# Script metadata
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
SCRIPT_NAME="$(basename "$0")"
LOG_FILE="/var/log/${SCRIPT_NAME%.sh}.log"

# Logging function
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

log "Script started"
```

---

## 3. Variables and Data Types

```bash
# Variable assignment (no spaces around =)
NAME="DevOps"
COUNT=42
PI=3.14

# Accessing variables
echo $NAME
echo ${NAME}         # Preferred — avoids ambiguity

# Command substitution
CURRENT_DATE=$(date +%Y-%m-%d)    # Preferred syntax
HOSTNAME=`hostname`               # Older backtick syntax (avoid)
FILE_COUNT=$(ls /tmp | wc -l)

# Arithmetic
NUM1=10
NUM2=3
SUM=$((NUM1 + NUM2))      # Integer arithmetic
PRODUCT=$((NUM1 * NUM2))
QUOTIENT=$((NUM1 / NUM2))
MODULO=$((NUM1 % NUM2))
POWER=$((NUM1 ** 2))

# Float arithmetic (bash doesn't support it natively)
FLOAT_RESULT=$(echo "scale=2; 10/3" | bc)
echo $FLOAT_RESULT  # 3.33

# Special variables
echo $0     # Script name
echo $1     # First argument
echo $2     # Second argument
echo $@     # All arguments (as separate words)
echo $*     # All arguments (as single word)
echo $#     # Number of arguments
echo $$     # Current PID
echo $!     # PID of last background process
echo $?     # Exit code of last command

# Variable defaults
echo ${VAR:-"default"}          # Use default if unset or empty
echo ${VAR:="default"}          # Set and use default if unset or empty
echo ${VAR:?"Error: not set"}   # Error if unset
echo ${VAR:+"alternate"}        # Use alternate if set

# Read-only variables
readonly DB_PORT=5432
declare -r MAX_RETRIES=3

# Environment variables
export APP_ENV="production"     # Export to child processes
printenv APP_ENV                # Print env var
env | grep APP_                 # Filter env vars
```

---

## 4. Conditionals

```bash
# if-elif-else
if [ "$USER" == "root" ]; then
    echo "Running as root"
elif [ "$USER" == "ubuntu" ]; then
    echo "Running as ubuntu"
else
    echo "Running as $USER"
fi

# Test conditions — File tests
[ -f file.txt ]         # File exists and is regular file
[ -d directory/ ]       # Directory exists
[ -e path ]             # Path exists (file or dir)
[ -r file ]             # File is readable
[ -w file ]             # File is writable
[ -x file ]             # File is executable
[ -s file ]             # File exists and is non-empty
[ -L file ]             # File is a symbolic link
[ ! -f file ]           # File does NOT exist

# String tests
[ -z "$VAR" ]           # String is empty
[ -n "$VAR" ]           # String is non-empty
[ "$A" == "$B" ]        # Strings are equal
[ "$A" != "$B" ]        # Strings are not equal
[ "$A" < "$B" ]         # String A sorts before B (alphabetically)

# Integer tests
[ $A -eq $B ]           # Equal
[ $A -ne $B ]           # Not equal
[ $A -lt $B ]           # Less than
[ $A -le $B ]           # Less than or equal
[ $A -gt $B ]           # Greater than
[ $A -ge $B ]           # Greater than or equal

# Compound conditions
[ -f file.txt ] && [ -r file.txt ]    # AND
[ -f a.txt ] || [ -f b.txt ]          # OR
[[ "$STR" =~ ^[0-9]+$ ]]             # Regex match (bash only)

# [[ ]] vs [ ]
# [[ ]] is bash-specific but safer, supports regex, no word splitting
[[ -n "$VAR" && "$VAR" == "prod" ]]   # Preferred in bash
[ -n "$VAR" -a "$VAR" = "prod" ]      # POSIX but error-prone

# Case statement
case "$ENVIRONMENT" in
    production|prod)
        DB_HOST="prod-db.internal"
        LOG_LEVEL="ERROR"
        ;;
    staging|stg)
        DB_HOST="staging-db.internal"
        LOG_LEVEL="WARN"
        ;;
    development|dev)
        DB_HOST="localhost"
        LOG_LEVEL="DEBUG"
        ;;
    *)
        echo "Unknown environment: $ENVIRONMENT"
        exit 1
        ;;
esac
```

---

## 5. Loops

```bash
# For loop
for i in 1 2 3 4 5; do
    echo "Item: $i"
done

# For loop with range
for i in {1..10}; do
    echo $i
done

# For loop with step
for i in {0..100..10}; do
    echo $i
done

# C-style for loop
for ((i=0; i<5; i++)); do
    echo "Counter: $i"
done

# For loop over array
SERVERS=("web-01" "web-02" "db-01" "cache-01")
for SERVER in "${SERVERS[@]}"; do
    echo "Checking $SERVER..."
    ping -c 1 "$SERVER" > /dev/null 2>&1 && echo "  UP" || echo "  DOWN"
done

# For loop over files
for FILE in /var/log/*.log; do
    SIZE=$(du -sh "$FILE" | cut -f1)
    echo "$FILE: $SIZE"
done

# While loop
COUNT=0
while [ $COUNT -lt 5 ]; do
    echo "Count: $COUNT"
    ((COUNT++))
done

# While with file reading
while IFS= read -r LINE; do
    echo "Processing: $LINE"
done < /etc/hosts

# While reading from pipe
cat servers.txt | while read SERVER; do
    ssh "$SERVER" "uptime"
done

# Until loop (opposite of while)
RETRY=0
until curl -s http://localhost:8080/health > /dev/null 2>&1; do
    RETRY=$((RETRY+1))
    if [ $RETRY -ge 10 ]; then
        echo "Service not responding after 10 retries"
        exit 1
    fi
    echo "Waiting for service... attempt $RETRY"
    sleep 5
done

# Break and Continue
for i in {1..10}; do
    [ $i -eq 5 ] && break       # Stop at 5
    [ $((i % 2)) -eq 0 ] && continue  # Skip even numbers
    echo $i
done
```

---

## 6. Functions

```bash
# Basic function
greet() {
    echo "Hello, $1!"
}
greet "DevOps"

# Function with return value (exit code)
check_service() {
    local SERVICE=$1
    systemctl is-active --quiet "$SERVICE"
    return $?    # 0 = active, non-zero = inactive
}

if check_service "nginx"; then
    echo "Nginx is running"
else
    echo "Nginx is not running"
fi

# Function with local variables (IMPORTANT)
calculate() {
    local NUM1=$1    # local prevents polluting global scope
    local NUM2=$2
    local RESULT=$((NUM1 + NUM2))
    echo $RESULT     # "Return" value via stdout
}

TOTAL=$(calculate 10 20)
echo "Total: $TOTAL"

# Function with error handling
deploy_app() {
    local APP=$1
    local VERSION=$2
    
    log "Deploying $APP version $VERSION"
    
    # Check prerequisites
    if ! command -v docker &> /dev/null; then
        log "ERROR: Docker not found"
        return 1
    fi
    
    # Pull image
    docker pull "$APP:$VERSION" || {
        log "ERROR: Failed to pull image"
        return 1
    }
    
    # Deploy
    docker-compose up -d || {
        log "ERROR: Failed to start services"
        return 1
    }
    
    log "Deployment of $APP:$VERSION successful"
    return 0
}

# Calling and checking return value
if deploy_app "myapp" "1.2.3"; then
    echo "Deployment succeeded"
else
    echo "Deployment failed"
    exit 1
fi
```

---

## 7. Input and Output

```bash
# Reading user input
read -p "Enter username: " USERNAME
read -sp "Enter password: " PASSWORD  # -s = silent (no echo)
echo ""  # New line after password

# Reading with timeout
read -t 10 -p "Continue? (y/n): " CONFIRM || echo "Timeout!"

# Reading multiple values
read -p "Enter name and age: " NAME AGE

# Script arguments with validation
usage() {
    cat << EOF
Usage: $0 [OPTIONS] <environment>
OPTIONS:
  -h, --help        Show this help
  -v, --verbose     Verbose output
  -d, --dry-run     Dry run mode
ENVIRONMENT:
  dev, staging, production
EOF
    exit 1
}

# Parse arguments
VERBOSE=false
DRY_RUN=false
ENV=""

while [[ $# -gt 0 ]]; do
    case $1 in
        -h|--help)
            usage
            ;;
        -v|--verbose)
            VERBOSE=true
            shift
            ;;
        -d|--dry-run)
            DRY_RUN=true
            shift
            ;;
        dev|staging|production)
            ENV=$1
            shift
            ;;
        *)
            echo "Unknown argument: $1"
            usage
            ;;
    esac
done

[ -z "$ENV" ] && { echo "Environment required"; usage; }

# Here documents
cat << 'EOF' > /tmp/config.conf
server {
    listen 80;
    server_name example.com;
    root /var/www/html;
}
EOF

# Here strings
MD5=$(md5sum <<< "some string")

# printf (better than echo for formatted output)
printf "%-20s %10s %10s\n" "Hostname" "CPU%" "MEM%"
printf "%-20s %10.2f %10.2f\n" "web-01" 12.5 45.3
```

---

## 8. String Manipulation

```bash
STR="Hello, DevOps World!"

# Length
echo ${#STR}                    # 20

# Substring
echo ${STR:7}                   # DevOps World!
echo ${STR:7:6}                 # DevOps

# Replace
echo ${STR/DevOps/Automation}   # Replace first occurrence
echo ${STR//o/0}                # Replace all occurrences

# Case conversion
echo ${STR,,}                   # lowercase all
echo ${STR^^}                   # UPPERCASE ALL
echo ${STR,}                    # lowercase first char
echo ${STR^}                    # Uppercase first char

# Trim prefix/suffix
FILENAME="backup-2024-01-15.tar.gz"
echo ${FILENAME#backup-}        # Remove shortest prefix: 2024-01-15.tar.gz
echo ${FILENAME##*.}            # Remove longest prefix (get extension): gz
echo ${FILENAME%.tar.gz}        # Remove suffix: backup-2024-01-15
echo ${FILENAME%%.*}            # Remove longest suffix: backup-2024-01-15

# Split string
IFS=',' read -ra PARTS <<< "web-01,web-02,db-01"
for PART in "${PARTS[@]}"; do
    echo $PART
done

# String contains
if [[ "$STR" == *"DevOps"* ]]; then
    echo "String contains DevOps"
fi

# String starts/ends with
if [[ "$STR" == Hello* ]]; then
    echo "Starts with Hello"
fi

if [[ "$STR" == *"!" ]]; then
    echo "Ends with !"
fi

# String comparison
[[ "abc" < "def" ]] && echo "abc sorts before def"
```

---

## 9. Arrays

```bash
# Declare arrays
FRUITS=("apple" "banana" "cherry")
declare -a SERVERS    # Explicit declaration

# Add elements
SERVERS+=("web-01")
SERVERS+=("web-02")
SERVERS[2]="db-01"

# Access elements
echo ${SERVERS[0]}          # First element
echo ${SERVERS[-1]}         # Last element
echo ${SERVERS[@]}          # All elements
echo ${#SERVERS[@]}         # Array length
echo ${!SERVERS[@]}         # All indices

# Slice
echo ${SERVERS[@]:1:2}      # Elements 1 and 2

# Loop over array
for SERVER in "${SERVERS[@]}"; do
    echo "Server: $SERVER"
done

# Loop with index
for i in "${!SERVERS[@]}"; do
    echo "[$i] ${SERVERS[$i]}"
done

# Associative arrays (dictionaries)
declare -A CONFIG
CONFIG["db_host"]="localhost"
CONFIG["db_port"]="5432"
CONFIG["db_name"]="myapp"

# Access
echo ${CONFIG["db_host"]}
echo ${CONFIG[@]}           # All values
echo ${!CONFIG[@]}          # All keys

# Loop associative array
for KEY in "${!CONFIG[@]}"; do
    echo "$KEY = ${CONFIG[$KEY]}"
done

# Remove element
unset SERVERS[1]

# Delete array
unset SERVERS
```

---

## 10. Error Handling

```bash
#!/usr/bin/env bash
set -euo pipefail

# Trap signals and errors
cleanup() {
    local EXIT_CODE=$?
    echo "Script exiting with code: $EXIT_CODE"
    # Cleanup temp files, release locks, etc.
    rm -f /tmp/script.lock
    exit $EXIT_CODE
}

trap cleanup EXIT           # Run on any exit
trap 'echo "Interrupted"' INT   # CTRL+C
trap 'echo "Script terminated"' TERM

# Custom error function
error() {
    echo "ERROR at line $1: $2" >&2  # stderr
    exit 1
}

# Trap errors with line number
trap 'error $LINENO "Command failed"' ERR

# Check command success
run_command() {
    if ! "$@"; then
        echo "Command failed: $*" >&2
        return 1
    fi
}

run_command mkdir -p /opt/app

# || pattern (run on failure)
mkdir /opt/data || { echo "Failed to create dir"; exit 1; }

# Check exit codes explicitly
systemctl start myapp
EXIT_CODE=$?
if [ $EXIT_CODE -ne 0 ]; then
    echo "Service failed to start (exit code: $EXIT_CODE)"
    journalctl -u myapp -n 50
    exit $EXIT_CODE
fi

# Retry pattern
retry() {
    local MAX_RETRIES=3
    local DELAY=5
    local CMD=("$@")
    
    for ((i=1; i<=MAX_RETRIES; i++)); do
        "${CMD[@]}" && return 0
        echo "Attempt $i failed. Retrying in ${DELAY}s..."
        sleep $DELAY
    done
    
    echo "Command failed after $MAX_RETRIES attempts: ${CMD[*]}"
    return 1
}

retry curl -s https://api.example.com/health
```

---

## 11. File Operations

```bash
# Check and create
[ -d "/opt/app" ] || mkdir -p "/opt/app"
[ -f "/etc/app.conf" ] || touch "/etc/app.conf"

# Lock file pattern (prevent concurrent runs)
LOCK_FILE="/tmp/script.lock"
if [ -f "$LOCK_FILE" ]; then
    PID=$(cat "$LOCK_FILE")
    if kill -0 "$PID" 2>/dev/null; then
        echo "Script already running (PID: $PID)"
        exit 1
    else
        echo "Removing stale lock file"
        rm "$LOCK_FILE"
    fi
fi
echo $$ > "$LOCK_FILE"
trap "rm -f $LOCK_FILE" EXIT

# Temp files (always use mktemp)
TEMP_FILE=$(mktemp /tmp/script.XXXXXX)
TEMP_DIR=$(mktemp -d /tmp/script.XXXXXX)
trap "rm -rf $TEMP_FILE $TEMP_DIR" EXIT

# Process file line by line
while IFS= read -r LINE || [ -n "$LINE" ]; do  # Handle no final newline
    echo "Processing: $LINE"
done < input.txt

# Write to file
cat > /etc/app.conf << 'EOF'
[database]
host = localhost
port = 5432
EOF

# Append to file
cat >> /var/log/deploy.log << EOF
[$(date)] Deployment started
EOF

# CSV processing
while IFS=',' read -r SERVER USER ACTION; do
    echo "Server: $SERVER, User: $USER, Action: $ACTION"
done < servers.csv
```

---

## 12. Regular Expressions

```bash
# Basic regex with grep
grep '^ERROR' app.log                   # Line starts with ERROR
grep 'ERROR$' app.log                   # Line ends with ERROR
grep '[0-9]\{3\}-[0-9]{4}' contacts    # Phone numbers
grep -E 'ERROR|WARN|FATAL' app.log      # Multiple patterns

# Regex in bash conditionals
IP="192.168.1.100"
if [[ "$IP" =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$ ]]; then
    echo "Valid IP address"
fi

EMAIL="user@example.com"
if [[ "$EMAIL" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
    echo "Valid email"
fi

# Extract with sed
VERSION=$(echo "App version 2.3.1 released" | sed 's/.*version \([0-9.]*\).*/\1/')

# Extract IP from log
echo "2024-01-15 10:00:00 client 192.168.1.100 connected" | \
    grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}'

# awk field extraction
awk '/ERROR/ {print $1, $2, $NF}' app.log   # Timestamp and last field

# sed multi-command
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt
```

---

## 13. Advanced Patterns

### Parallel Execution

```bash
#!/usr/bin/env bash
# Run SSH commands in parallel
SERVERS=("web-01" "web-02" "web-03" "web-04")
PIDS=()

for SERVER in "${SERVERS[@]}"; do
    ssh "$SERVER" "uptime" &   # Background
    PIDS+=($!)                 # Store PID
done

# Wait for all and collect results
FAILED=0
for i in "${!PIDS[@]}"; do
    wait "${PIDS[$i]}" || {
        echo "Command failed on ${SERVERS[$i]}"
        FAILED=$((FAILED+1))
    }
done

[ $FAILED -gt 0 ] && exit 1 || echo "All servers responded"
```

### Configuration File Management

```bash
#!/usr/bin/env bash
# Deploy config from template
TEMPLATE="/opt/templates/nginx.conf.tmpl"
OUTPUT="/etc/nginx/nginx.conf"
BACKUP="/etc/nginx/nginx.conf.$(date +%Y%m%d%H%M%S)"

# Backup existing config
[ -f "$OUTPUT" ] && cp "$OUTPUT" "$BACKUP"

# Substitute variables in template
sed -e "s|{{SERVER_NAME}}|${SERVER_NAME}|g" \
    -e "s|{{UPSTREAM_HOST}}|${UPSTREAM_HOST}|g" \
    -e "s|{{MAX_WORKERS}}|${MAX_WORKERS}|g" \
    "$TEMPLATE" > "$OUTPUT"

# Test config
nginx -t 2>&1 || {
    echo "Config test failed, restoring backup"
    cp "$BACKUP" "$OUTPUT"
    exit 1
}

nginx -s reload
echo "Nginx config updated successfully"
```

---

## 14. Real-World DevOps Scripts

### Script 1: Automated Deployment

```bash
#!/usr/bin/env bash
set -euo pipefail

# deploy.sh — Deploy application to server
APP_NAME="${1:-}"
VERSION="${2:-}"
ENVIRONMENT="${3:-staging}"

[ -z "$APP_NAME" ] || [ -z "$VERSION" ] && {
    echo "Usage: $0 <app-name> <version> [environment]"
    exit 1
}

log() { echo "[$(date '+%H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; exit 1; }

DEPLOY_DIR="/opt/deployments/$APP_NAME"
CURRENT_LINK="$DEPLOY_DIR/current"
RELEASES_DIR="$DEPLOY_DIR/releases"
RELEASE_DIR="$RELEASES_DIR/$VERSION"

log "Deploying $APP_NAME version $VERSION to $ENVIRONMENT"

# Create release directory
mkdir -p "$RELEASE_DIR"

# Download artifact
log "Downloading artifact..."
aws s3 cp "s3://artifacts/$APP_NAME/$VERSION/app.tar.gz" "$RELEASE_DIR/"

# Extract
log "Extracting..."
tar -xzf "$RELEASE_DIR/app.tar.gz" -C "$RELEASE_DIR/"

# Set permissions
chown -R app-user:app-group "$RELEASE_DIR"

# Run pre-deploy hooks
[ -f "$RELEASE_DIR/hooks/pre-deploy.sh" ] && {
    log "Running pre-deploy hooks..."
    bash "$RELEASE_DIR/hooks/pre-deploy.sh"
}

# Atomic symlink update
log "Updating symlink..."
PREV_RELEASE=$(readlink "$CURRENT_LINK" 2>/dev/null || echo "")
ln -sfn "$RELEASE_DIR" "$CURRENT_LINK"

# Restart service
log "Restarting service..."
systemctl restart "$APP_NAME" || {
    # Rollback on failure
    log "Restart failed, rolling back..."
    [ -n "$PREV_RELEASE" ] && ln -sfn "$PREV_RELEASE" "$CURRENT_LINK"
    systemctl restart "$APP_NAME"
    error "Deployment failed, rolled back to previous version"
}

# Health check
log "Running health check..."
sleep 5
for i in {1..12}; do
    if curl -sf "http://localhost:8080/health" > /dev/null; then
        log "Health check passed"
        break
    fi
    [ $i -eq 12 ] && {
        log "Health check failed, rolling back..."
        [ -n "$PREV_RELEASE" ] && ln -sfn "$PREV_RELEASE" "$CURRENT_LINK"
        systemctl restart "$APP_NAME"
        error "Deployment failed health check, rolled back"
    }
    sleep 5
done

# Cleanup old releases (keep last 5)
log "Cleaning up old releases..."
ls -t "$RELEASES_DIR" | tail -n +6 | xargs -I {} rm -rf "$RELEASES_DIR/{}"

log "Deployment of $APP_NAME $VERSION successful!"
```

### Script 2: Server Health Check

```bash
#!/usr/bin/env bash
# health-check.sh — Comprehensive server health report

ALERT_EMAIL="ops@company.com"
HOSTNAME=$(hostname -f)
REPORT=""
ALERTS=()

check() {
    local NAME=$1
    local STATUS=$2
    local VALUE=$3
    local THRESHOLD=$4
    
    if (( $(echo "$VALUE > $THRESHOLD" | bc -l) )); then
        ALERTS+=("ALERT: $NAME is $VALUE (threshold: $THRESHOLD)")
        REPORT+="❌ $NAME: $VALUE (ALERT)\n"
    else
        REPORT+="✅ $NAME: $VALUE\n"
    fi
}

# CPU load
LOAD=$(uptime | awk '{print $(NF-2)}' | tr -d ',')
CPU_CORES=$(nproc)
check "CPU Load (1min)" "load" "$LOAD" "$CPU_CORES"

# Memory usage
MEM_TOTAL=$(free -m | awk '/Mem:/ {print $2}')
MEM_USED=$(free -m | awk '/Mem:/ {print $3}')
MEM_PCT=$(echo "scale=1; $MEM_USED * 100 / $MEM_TOTAL" | bc)
check "Memory Usage%" "mem" "$MEM_PCT" "85"

# Disk usage
while read -r USAGE MOUNT; do
    check "Disk $MOUNT" "disk" "$USAGE" "85"
done < <(df -h | awk 'NR>1 && /\// {gsub(/%/,"",$5); print $5, $6}')

# Print report
echo -e "Health Report for $HOSTNAME — $(date)"
echo -e "$REPORT"

# Send alerts
if [ ${#ALERTS[@]} -gt 0 ]; then
    ALERT_MSG=$(printf '%s\n' "${ALERTS[@]}")
    echo -e "ALERTS:\n$ALERT_MSG\n\nFull Report:\n$REPORT" | \
        mail -s "⚠️ Server Alert: $HOSTNAME" "$ALERT_EMAIL"
fi
```

### Script 3: Database Backup

```bash
#!/usr/bin/env bash
set -euo pipefail

# backup-db.sh — Backup MySQL database to S3
DB_HOST="${DB_HOST:-localhost}"
DB_USER="${DB_USER:-root}"
DB_PASS="${DB_PASS:?DB_PASS must be set}"
DB_NAME="${DB_NAME:?DB_NAME must be set}"
S3_BUCKET="${S3_BUCKET:?S3_BUCKET must be set}"
BACKUP_DIR="/tmp/db-backups"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz"

mkdir -p "$BACKUP_DIR"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

log "Starting backup of $DB_NAME"

# Dump database
mysqldump \
    --host="$DB_HOST" \
    --user="$DB_USER" \
    --password="$DB_PASS" \
    --single-transaction \
    --routines \
    --triggers \
    "$DB_NAME" | gzip > "$BACKUP_FILE"

SIZE=$(du -sh "$BACKUP_FILE" | cut -f1)
log "Backup created: $BACKUP_FILE ($SIZE)"

# Upload to S3
log "Uploading to S3..."
aws s3 cp "$BACKUP_FILE" "s3://$S3_BUCKET/backups/$DB_NAME/$(basename $BACKUP_FILE)" \
    --storage-class STANDARD_IA

# Clean local backup
rm -f "$BACKUP_FILE"

# Delete S3 backups older than 30 days
log "Cleaning old S3 backups..."
aws s3 ls "s3://$S3_BUCKET/backups/$DB_NAME/" | \
    awk '{print $4}' | \
    while read FILE; do
        FILE_DATE=$(echo "$FILE" | grep -oP '\d{4}-\d{2}-\d{2}')
        DAYS_OLD=$(( ($(date +%s) - $(date -d "$FILE_DATE" +%s)) / 86400 ))
        if [ "$DAYS_OLD" -gt 30 ]; then
            aws s3 rm "s3://$S3_BUCKET/backups/$DB_NAME/$FILE"
            log "Deleted old backup: $FILE ($DAYS_OLD days old)"
        fi
    done

log "Backup completed successfully"
```

---

## 15. Common Interview Questions

**Q1: What is `set -euo pipefail` and why use it?**
> `set -e` exits on error; `set -u` exits on undefined variables; `set -o pipefail` makes pipes fail if any command fails (not just the last). Critical for production scripts — without it, a failed command is silently ignored and the script continues with potentially wrong state.

**Q2: What's the difference between `$@` and `$*`?**
> Both hold all script arguments, but behavior differs in quotes: `"$@"` expands to individual quoted words `"$1" "$2" ...` (preserves spaces in args). `"$*"` expands to one single string `"$1 $2 ..."`. Always use `"$@"` when passing arguments.

**Q3: How do you debug a shell script?**
> `bash -x script.sh` (print each command). `bash -v script.sh` (print each line as read). `bash -n script.sh` (syntax check only). Inside script: `set -x` to enable, `set +x` to disable debug mode for a section.

**Q4: What is the difference between single quotes and double quotes?**
> Single quotes preserve everything literally — no variable expansion, no command substitution. Double quotes allow variable expansion (`$VAR`), command substitution (`$(cmd)`), and escape sequences. Use single quotes for literal strings, double quotes when variables need to expand.

**Q5: How do you safely read a file line by line?**
> `while IFS= read -r line; do echo "$line"; done < file.txt`. `IFS=` prevents stripping leading/trailing whitespace. `-r` prevents backslash interpretation. The `|| [ -n "$line" ]` handles files without trailing newline.

**Q6: What is the difference between `>` and `>>`?**
> `>` overwrites the file (creates new). `>>` appends to existing file (creates if not exists). For logs, always use `>>`. Be careful with `>` — `> file` truncates file instantly.

**Q7: How do you use a lock file to prevent concurrent script execution?**
> Create a lock file with the PID. On start, check if lock exists and if that PID is still running. Use `trap` to delete lock file on exit. `flock` command is even better for file-based locking.

**Q8: Explain process substitution `<(command)` and when to use it.**
> Process substitution treats command output as a file. `diff <(ssh server1 "cat /etc/hosts") <(ssh server2 "cat /etc/hosts")` — compare files from two servers. Useful with commands that don't accept pipes. Different from command substitution `$(command)` which gives a string.

**Q9: What is `getopts` and how is it used?**
> `getopts` is a built-in for parsing short options: `while getopts "hv:d" opt; do case $opt in h) usage;;  v) VERSION=$OPTARG;; d) DEBUG=true;; esac; done`. The colon after option means it requires an argument. More portable than manual case statements for flags.

**Q10: How do you make a script idempotent?**
> An idempotent script produces the same result whether run once or multiple times. Techniques: `mkdir -p` instead of `mkdir`; `[ -f file ] || create_file`; use `--ignore-existing` flags; check state before making changes; use `set -e` to stop on first error.

---

*Next: [Shell Scripting Assignments](02-shell-scripting-assignments.md)*
