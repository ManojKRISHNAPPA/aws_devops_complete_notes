# Linux for DevOps — Assignments

## Assignment 1: File System Navigation and Operations

**Tasks:**
```bash
# 1. Create the following directory structure:
/opt/devops/
├── scripts/
│   ├── backup/
│   ├── monitoring/
│   └── deploy/
├── logs/
│   ├── app/
│   └── system/
└── configs/

# 2. Create files in each directory
touch /opt/devops/scripts/backup/backup.sh
touch /opt/devops/scripts/monitoring/health-check.sh
touch /opt/devops/scripts/deploy/deploy.sh
touch /opt/devops/configs/app.conf

# 3. Set appropriate permissions:
# - Scripts: owner rwx, group rx, others no access
# - Configs: owner rw, group r, others no access
# - Logs dir: all can write (sticky bit)

# 4. Find all .sh files and make them executable
find /opt/devops -name "*.sh" -exec chmod +x {} \;

# 5. Create symlinks:
# /opt/devops/latest-backup → /opt/devops/logs/backup/
```

---

## Assignment 2: Process Investigation

**Scenario:** Production server is running slow. Investigate and resolve.

**Tasks:**
1. Find the top 5 CPU-consuming processes
2. Find the top 5 memory-consuming processes
3. Find all processes owned by `www-data` user
4. Check if nginx is running and get its PID
5. Find all processes listening on ports 80, 443, 8080
6. Find which process has a specific file open
7. Kill a process using its name (not PID)
8. Create a one-liner that kills all zombie processes

**Commands to practice:**
```bash
# Check load average and interpret it (4-core machine, load=3.2)
uptime && nproc

# Find memory hog and reduce its priority
ps aux --sort=-%mem | head -5
renice 10 <PID>

# List all files opened by nginx
lsof -u www-data | head -20

# Find which process has /var/log/nginx/access.log open
lsof /var/log/nginx/access.log
```

---

## Assignment 3: Log Analysis

**Simulate an nginx access log with errors and analyze it:**

```bash
# Generate test log
cat > /tmp/access.log << 'EOF'
192.168.1.1 - - [15/Jan/2024:10:00:01] "GET /api/users HTTP/1.1" 200 1234
192.168.1.2 - - [15/Jan/2024:10:00:02] "POST /api/login HTTP/1.1" 200 567
192.168.1.1 - - [15/Jan/2024:10:01:00] "GET /api/data HTTP/1.1" 500 89
10.0.0.5 - - [15/Jan/2024:10:01:30] "GET /api/missing HTTP/1.1" 404 156
10.0.0.5 - - [15/Jan/2024:10:02:00] "POST /api/data HTTP/1.1" 500 89
192.168.1.100 - - [15/Jan/2024:10:02:30] "GET /api/users HTTP/1.1" 200 2345
EOF
```

**Analysis tasks:**
```bash
# 1. Count total requests
wc -l /tmp/access.log

# 2. Count requests per status code
awk '{print $9}' /tmp/access.log | sort | uniq -c | sort -rn

# 3. Top 5 IP addresses by request count
awk '{print $1}' /tmp/access.log | sort | uniq -c | sort -rn | head 5

# 4. List all 500 errors with timestamps
grep '" 500 ' /tmp/access.log

# 5. Total bytes transferred
awk '{sum += $NF} END {print sum, "bytes"}' /tmp/access.log

# 6. Count requests per minute
awk '{print $4}' /tmp/access.log | cut -d: -f2 | sort | uniq -c
```

---

## Assignment 4: User and Permission Management

**Scenario:** Set up a web server with proper security

**Tasks:**
```bash
# 1. Create groups
groupadd webadmin
groupadd developers
groupadd readonly

# 2. Create users with appropriate groups
useradd -m -s /bin/bash -G webadmin,sudo alice
useradd -m -s /bin/bash -G developers bob
useradd -m -s /bin/bash -G readonly charlie

# 3. Set passwords (use password from vault in real scenario)
echo "alice:SecurePass123" | chpasswd
echo "bob:SecurePass456" | chpasswd

# 4. Create web directory structure
mkdir -p /var/www/mysite/{html,logs,uploads}

# 5. Set ownership and permissions:
# - html/: webadmin can read/write, developers can read, others nothing
# - logs/: webadmin only
# - uploads/: all authenticated users can write (sticky bit)

# 6. Verify permissions
ls -la /var/www/mysite/

# 7. Test access by switching users
su - bob -c "ls /var/www/mysite/html/"
su - charlie -c "cat /var/www/mysite/logs/access.log"  # Should fail
```

---

## Assignment 5: Networking Troubleshooting

**Scenario:** Application can't connect to database. Diagnose.

**Tasks:**
```bash
# 1. Check if database port is open
nc -zv db-hostname 5432
telnet db-hostname 5432

# 2. Check DNS resolution
nslookup db-hostname
dig db-hostname

# 3. Check routing to database
traceroute db-hostname

# 4. Check if firewall is blocking
iptables -L -n | grep 5432
ufw status

# 5. Check if database is listening locally
ss -tulpn | grep :5432

# 6. Check established connections
ss -an | grep :5432 | grep ESTABLISHED

# 7. Capture network traffic (requires root)
tcpdump -i eth0 port 5432 -n

# Practice: Find all listening ports and identify processes
ss -tulpn
# Match each port to a process using:
lsof -i :<port>
```

---

## Assignment 6: Systemd Service Creation

**Task:** Create a systemd service for a Python application

```python
# /opt/myapp/app.py
from http.server import HTTPServer, BaseHTTPRequestHandler
import json

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == '/health':
            self.send_response(200)
            self.end_headers()
            self.wfile.write(json.dumps({'status': 'healthy'}).encode())
        else:
            self.send_response(404)
            self.end_headers()

if __name__ == '__main__':
    server = HTTPServer(('0.0.0.0', 8080), Handler)
    print("Starting server on port 8080")
    server.serve_forever()
```

**Tasks:**
1. Save the Python app to `/opt/myapp/app.py`
2. Create system user `myapp`
3. Set ownership: `chown -R myapp:myapp /opt/myapp`
4. Create `/etc/systemd/system/myapp.service`
5. Enable and start the service
6. Verify it starts on reboot
7. Test the health endpoint: `curl http://localhost:8080/health`
8. View service logs: `journalctl -u myapp -f`
9. Simulate crash: `kill -9 <pid>` and verify auto-restart

---

## Assignment 7: Disk Management (EBS scenario)

**Simulate attaching and configuring a new disk:**

```bash
# For practice, create a file-backed disk
dd if=/dev/zero of=/tmp/fake-disk.img bs=1M count=1024
losetup /dev/loop0 /tmp/fake-disk.img

# Tasks:
# 1. Format the disk with ext4
mkfs.ext4 /dev/loop0

# 2. Create mount point
mkdir /mnt/appdata

# 3. Mount it
mount /dev/loop0 /mnt/appdata

# 4. Verify
df -h /mnt/appdata

# 5. Add to /etc/fstab (for real EBS volumes)
# Get UUID: blkid /dev/loop0

# 6. Check filesystem usage
du -sh /mnt/appdata

# 7. Unmount
umount /mnt/appdata
losetup -d /dev/loop0
```

---

## Interview Assignment: Server Performance Investigation

**Scenario:** Your production web server is responding slowly (response time > 5s, normally < 200ms). You SSH in and have 15 minutes before emergency escalation.

**Investigate in this order and document findings:**

```bash
# Step 1: Quick health snapshot (30 seconds)
uptime              # Load average
free -h             # Memory
df -h               # Disk space

# Step 2: CPU investigation (2 minutes)
top                 # What's eating CPU?
# Press 1 for per-core view
# Press P to sort by CPU
# Press M to sort by memory

# Step 3: Memory investigation (2 minutes)
free -h
cat /proc/meminfo | grep -E 'MemTotal|MemFree|SwapUsed'
# Is swap being used? That's a problem.
vmstat 1 5          # Page-in/page-out activity

# Step 4: Disk I/O (2 minutes)
iostat -xz 1 3
# Look for %util > 90% — disk bottleneck
iotop               # Per-process I/O

# Step 5: Network (2 minutes)
ss -s               # Connection statistics
ss -an | grep :80 | grep ESTABLISHED | wc -l  # Active connections
ss -an | grep :80 | grep TIME_WAIT | wc -l    # Connections in TIME_WAIT

# Step 6: Application logs (2 minutes)
tail -100 /var/log/nginx/error.log
journalctl -u myapp --since "10 minutes ago"

# Step 7: Document and escalate with findings
```

**Write a 1-page incident report covering:**
- What you found
- Root cause (hypothetical)
- Actions taken
- Immediate mitigation
- Long-term fix recommendation

---

## Cheat Sheet

```bash
# Quick server health (one-liner)
echo "=== LOAD ===" && uptime && echo "=== MEM ===" && free -h && echo "=== DISK ===" && df -h | grep -v tmpfs && echo "=== PROCESSES ===" && ps aux --sort=-%cpu | head -6

# Find large files
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -rh | head 10

# Watch log file for errors
tail -f /var/log/app.log | grep --line-buffered -i 'error\|exception\|fatal'

# Count connections by IP
ss -an | grep :80 | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head 10

# Check which service is using most memory
ps aux --sort=-%mem | awk 'NR>1{print $4, $11}' | head 10

# Disk usage excluding tmpfs
df -h --exclude-type=tmpfs
```
