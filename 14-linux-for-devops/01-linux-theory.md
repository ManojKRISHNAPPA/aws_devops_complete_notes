# Linux for DevOps Engineers — Complete Guide

## Table of Contents
1. [Why Linux for DevOps?](#1-why-linux-for-devops)
2. [Linux Architecture](#2-linux-architecture)
3. [File System Hierarchy](#3-file-system-hierarchy)
4. [Essential Commands](#4-essential-commands)
5. [File Permissions and Ownership](#5-file-permissions-and-ownership)
6. [Process Management](#6-process-management)
7. [Disk and Storage Management](#7-disk-and-storage-management)
8. [Networking on Linux](#8-networking-on-linux)
9. [Package Management](#9-package-management)
10. [Systemd and Services](#10-systemd-and-services)
11. [Logs and Monitoring](#11-logs-and-monitoring)
12. [Users and Groups](#12-users-and-groups)
13. [SSH and Remote Access](#13-ssh-and-remote-access)
14. [Cron Jobs and Scheduling](#14-cron-jobs-and-scheduling)
15. [Linux Performance Tuning](#15-linux-performance-tuning)
16. [Security Hardening](#16-security-hardening)
17. [Real-World DevOps Use Cases](#17-real-world-devops-use-cases)
18. [Common Interview Questions](#18-common-interview-questions)

---

## 1. Why Linux for DevOps?

```
DevOps Tool Stack — % Running on Linux
──────────────────────────────────────
Docker containers         → Linux kernel (100%)
Kubernetes nodes          → Linux (99%+)
Jenkins/CI servers        → Linux
Nginx/Apache web servers  → Linux
MySQL/PostgreSQL databases → Linux
AWS EC2 instances         → Linux (most workloads)
Terraform/Ansible control → Linux/macOS
Cloud servers globally    → ~70% Linux
```

**DevOps Daily Linux Tasks:**
- SSH into servers for debugging
- Read and tail application logs
- Monitor CPU/memory/disk usage
- Manage services (start/stop/restart)
- Configure networking and firewall rules
- Manage cron jobs and scheduled tasks
- Write shell scripts for automation
- Manage packages and dependencies
- Manage file permissions and users
- Troubleshoot performance issues

---

## 2. Linux Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    User Applications                     │
│         (nginx, mysql, docker, kubectl, etc.)            │
├─────────────────────────────────────────────────────────┤
│                    Shell / CLI                           │
│              (bash, zsh, sh, fish)                       │
├─────────────────────────────────────────────────────────┤
│                  System Libraries                        │
│                    (glibc, etc.)                         │
├─────────────────────────────────────────────────────────┤
│                   System Calls                           │
│            (open, read, write, fork, etc.)               │
├─────────────────────────────────────────────────────────┤
│                   Linux Kernel                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │ Process  │ │ Memory   │ │File Sys  │ │Network   │  │
│  │ Mgmt     │ │ Mgmt     │ │Driver    │ │Stack     │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
├─────────────────────────────────────────────────────────┤
│                   Hardware                               │
│         (CPU, RAM, Disk, Network, GPU)                   │
└─────────────────────────────────────────────────────────┘
```

**Key Linux Distributions for DevOps:**

| Distro | Package Manager | Use Case |
|--------|----------------|----------|
| **Ubuntu/Debian** | apt | Containers, development, EC2 |
| **Amazon Linux 2/2023** | yum/dnf | AWS-optimized, EC2 |
| **RHEL/CentOS/Rocky** | yum/dnf | Enterprise, on-prem |
| **Alpine** | apk | Docker base images (tiny ~5MB) |
| **Arch Linux** | pacman | Cutting-edge, personal use |

---

## 3. File System Hierarchy

```
/ (root)
├── /bin        → Essential binaries (ls, cp, mv, bash)
├── /sbin       → System binaries (fdisk, ifconfig, reboot)
├── /etc        → Configuration files (nginx.conf, /etc/hosts, fstab)
├── /home       → User home directories (/home/ubuntu, /home/ec2-user)
├── /root       → Root user's home directory
├── /var        → Variable data (logs, databases, spool files)
│   ├── /var/log    → System and application logs
│   ├── /var/lib    → Application state (mysql data, docker storage)
│   └── /var/run    → Runtime PID files
├── /tmp        → Temporary files (cleared on reboot)
├── /usr        → User utilities and applications
│   ├── /usr/bin    → Non-essential user commands
│   ├── /usr/local  → Locally installed software
│   └── /usr/share  → Architecture-independent data
├── /opt        → Optional/third-party software
├── /proc       → Virtual filesystem — kernel & process info
│   ├── /proc/cpuinfo   → CPU information
│   ├── /proc/meminfo   → Memory information
│   └── /proc/[pid]/    → Per-process info
├── /sys        → Virtual filesystem — hardware/kernel info
├── /dev        → Device files (disks, terminals)
│   ├── /dev/sda    → First SATA disk
│   ├── /dev/xvda   → First Xen disk (AWS EC2)
│   └── /dev/nvme0  → First NVMe disk
├── /mnt        → Temporary mount point
└── /media      → Removable media mount point
```

**DevOps Important Paths:**
```
/etc/nginx/nginx.conf          → Nginx config
/etc/ssh/sshd_config           → SSH daemon config
/etc/crontab                   → System cron jobs
/etc/hosts                     → Local DNS overrides
/etc/hostname                  → System hostname
/etc/fstab                     → Filesystem mount table
/var/log/syslog                → System log (Ubuntu)
/var/log/messages              → System log (RHEL/CentOS)
/var/log/nginx/access.log      → Nginx access log
/var/log/nginx/error.log       → Nginx error log
~/.bashrc                      → User bash config
~/.bash_profile / ~/.profile   → Login shell config
~/.ssh/authorized_keys         → SSH public keys
```

---

## 4. Essential Commands

### Navigation and File Operations

```bash
# Navigation
pwd                     # Print working directory
ls -la                  # List all files with details
ls -lah                 # Human-readable sizes
cd /var/log             # Change directory
cd ~                    # Go to home directory
cd ..                   # Go up one level
cd -                    # Go to previous directory

# File Operations
cp file.txt /tmp/       # Copy file
cp -r dir/ /tmp/        # Copy directory recursively
mv old.txt new.txt      # Move/rename
rm file.txt             # Remove file
rm -rf directory/       # Remove directory (careful!)
mkdir -p /opt/app/logs  # Create nested directories
touch newfile.txt       # Create empty file or update timestamp

# Viewing Files
cat file.txt            # Print entire file
less file.txt           # Paginated viewer (q to quit)
head -n 20 file.txt     # First 20 lines
tail -n 20 file.txt     # Last 20 lines
tail -f /var/log/app.log  # Follow log file in real-time
tail -F /var/log/app.log  # Follow even if file rotates
```

### Searching and Finding

```bash
# Find files
find /etc -name "*.conf"              # Find by name
find /var/log -name "*.log" -mtime -1 # Modified in last 1 day
find /home -type f -size +10M         # Files > 10MB
find . -name "*.py" -exec grep -l "import os" {} \;

# Search content
grep "ERROR" /var/log/app.log                  # Search in file
grep -r "database" /etc/                       # Recursive search
grep -n "ERROR" app.log                        # Show line numbers
grep -i "error" app.log                        # Case insensitive
grep -v "DEBUG" app.log                        # Exclude matches
grep -c "ERROR" app.log                        # Count matches
grep -A 3 -B 3 "ERROR" app.log                # 3 lines context

# Advanced: grep with regex
grep -E "ERR(OR)?|WARN" app.log               # Extended regex
grep -P "\d{3}-\d{4}" contacts.txt            # Perl regex

# Text processing
awk '{print $1, $3}' access.log               # Print columns 1 and 3
awk '/ERROR/ {print $0}' app.log              # Print matching lines
awk -F: '{print $1}' /etc/passwd              # Custom delimiter
sed 's/old/new/g' file.txt                    # Replace text
sed -n '10,20p' file.txt                      # Print lines 10-20
sed '/^#/d' config.conf                       # Delete comment lines
cut -d: -f1 /etc/passwd                       # Cut first field
sort -k2 -n file.txt                          # Sort by column 2 numerically
sort | uniq -c | sort -rn                     # Count unique, sort by freq
wc -l file.txt                                # Count lines
```

### Pipes and Redirection

```bash
# Redirection
command > file.txt        # Redirect stdout (overwrite)
command >> file.txt       # Redirect stdout (append)
command 2> error.txt      # Redirect stderr
command 2>&1              # Redirect stderr to stdout
command &> all.txt        # Redirect both stdout and stderr
command < input.txt       # Redirect stdin

# Pipes
cat access.log | grep "404" | wc -l        # Count 404 errors
ps aux | grep nginx | grep -v grep         # Find nginx processes
cat /etc/passwd | cut -d: -f1 | sort      # List users sorted
du -sh /var/* | sort -h                    # Disk usage sorted
netstat -tulpn | grep :80                  # Check port 80

# Tee — write to file AND stdout
command | tee output.txt             # Write and display
command | tee -a output.txt          # Append
```

---

## 5. File Permissions and Ownership

### Permission Structure

```
-rwxr-xr-- 1 ubuntu devops 4096 May 27 10:00 script.sh
│││││││││││
│││││││││└─ Other: r-- = read only (4)
││││││││└── Other execute
│││││││└─── Other write
││││││└──── Other read
│││││└───── Group: r-x = read + execute (5)
││││└────── Group execute
│││└─────── Group write
││└──────── Group read
│└───────── User: rwx = full (7)
│           User execute
│           User write
│           User read
└─────────── File type: - = regular, d = dir, l = symlink
```

### Permission Numbers

```bash
# Octal notation
# r=4, w=2, x=1
chmod 755 script.sh     # rwxr-xr-x (owner: all, group/other: read+exec)
chmod 644 config.conf   # rw-r--r-- (owner: read+write, others: read only)
chmod 600 private.key   # rw------- (owner only: read+write)
chmod 777 public.txt    # rwxrwxrwx (everyone: full — AVOID!)
chmod 400 secret.pem    # r-------- (read only, for SSH keys)

# Symbolic notation
chmod u+x script.sh     # Add execute for user
chmod g-w file.txt      # Remove write for group
chmod o= file.txt       # Remove all permissions for others
chmod a+r file.txt      # Add read for all (a = all)
chmod u+s binary        # Set SUID bit
chmod g+s directory/    # Set SGID bit

# Recursive
chmod -R 755 /opt/app/
chown -R ubuntu:www-data /var/www/html/

# Change ownership
chown ubuntu:devops file.txt    # owner:group
chown ubuntu file.txt           # change owner only
chgrp devops file.txt           # change group only

# Special permissions
chmod +t /tmp                   # Sticky bit (only owner can delete)
chmod 4755 /usr/bin/sudo        # SUID — runs as file owner
chmod 2755 /var/log             # SGID — files inherit group
```

### Real-World Permission Examples

```bash
# Web server files
chown -R www-data:www-data /var/www/html/
chmod -R 755 /var/www/html/
chmod -R 644 /var/www/html/*.html

# SSH key permissions (MUST be exact)
chmod 700 ~/.ssh/
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub

# Application deployment
chown -R app-user:app-group /opt/myapp/
chmod 750 /opt/myapp/
chmod 640 /opt/myapp/config.env  # Only app-user can read
```

---

## 6. Process Management

```bash
# View processes
ps aux                          # All processes (user, pid, cpu%, mem%)
ps aux | grep nginx             # Find specific process
ps -ef --forest                 # Process tree
top                             # Real-time process viewer
htop                            # Enhanced top (install separately)
pgrep nginx                     # Find PID by name
pidof nginx                     # Find PID by name

# Process control
kill 1234                       # Send SIGTERM (graceful stop)
kill -9 1234                    # Send SIGKILL (force kill)
kill -HUP 1234                  # Send SIGHUP (reload config)
killall nginx                   # Kill by name
pkill -f "python app.py"        # Kill by command pattern

# Background processes
command &                       # Run in background
jobs                            # List background jobs
fg %1                           # Bring job 1 to foreground
bg %1                           # Continue job 1 in background
nohup command &                 # Run immune to hangup
disown %1                       # Detach job from shell

# Process priority
nice -n 10 command              # Start with lower priority (10)
renice -n 5 -p 1234            # Change priority of running process
# nice values: -20 (highest) to 19 (lowest)

# Process resource usage
lsof -p 1234                    # Files opened by PID
lsof -i :80                     # Processes using port 80
strace -p 1234                  # Trace system calls of process
```

---

## 7. Disk and Storage Management

```bash
# Disk usage
df -h                           # Filesystem disk space usage
df -h /var                      # Specific filesystem
du -sh /var/log/                # Directory size
du -sh /var/* | sort -h         # All subdirs sorted by size
du -sh --max-depth=2 /          # 2 levels deep
ncdu /                          # Interactive disk usage (install separately)

# Disk operations
lsblk                           # List block devices
fdisk -l                        # List partitions (root needed)
blkid                           # Show UUID and filesystem type
mount                           # Show all mounts
mount /dev/xvdf /mnt/data       # Mount device
umount /mnt/data                # Unmount device

# Format and partition (DevOps use case: attaching EBS volume)
# 1. List devices
lsblk
# 2. Create partition (if needed)
fdisk /dev/xvdf
# 3. Format
mkfs.ext4 /dev/xvdf
# 4. Create mount point
mkdir /mnt/data
# 5. Mount
mount /dev/xvdf /mnt/data
# 6. Persist across reboots
echo '/dev/xvdf /mnt/data ext4 defaults 0 0' >> /etc/fstab
# 7. Verify
df -h /mnt/data

# Check filesystem
fsck /dev/xvdf                  # Filesystem check (unmounted)
e2fsck -f /dev/xvdf             # Force check ext4

# Extend filesystem after EBS resize
# After resizing EBS volume in AWS console:
sudo growpart /dev/xvda 1       # Extend partition
sudo resize2fs /dev/xvda1       # Extend ext4 filesystem
# or for xfs:
sudo xfs_growfs /               # Extend XFS filesystem
```

---

## 8. Networking on Linux

```bash
# Interface and IP info
ip addr show                    # Show all interfaces and IPs
ip addr show eth0               # Show specific interface
hostname -I                     # Show all IPs
ifconfig                        # Older command (deprecated)

# Routing
ip route show                   # Show routing table
route -n                        # Show routing table (older)
ip route add 10.0.0.0/8 via 192.168.1.1  # Add static route

# Network testing
ping -c 4 8.8.8.8               # Test connectivity (4 packets)
ping -c 4 google.com            # DNS + connectivity test
traceroute google.com           # Trace network path
mtr google.com                  # Better traceroute (dynamic)
telnet host 80                  # Test TCP connectivity to port
nc -zv host 80                  # Netcat: test port (verbose)
nc -zv host 80-100              # Test port range
curl -I https://example.com     # HTTP headers only
curl -v https://example.com     # Verbose HTTP request

# Port and socket info
ss -tulpn                       # Show listening ports (modern)
netstat -tulpn                  # Show listening ports (older)
ss -an | grep :80               # Check port 80 connections
lsof -i :8080                   # Who's using port 8080

# DNS
nslookup example.com            # DNS lookup
dig example.com                 # Detailed DNS info
dig @8.8.8.8 example.com        # Use specific DNS server
dig example.com MX              # Look up MX records
host example.com                # Simple DNS lookup
cat /etc/resolv.conf            # DNS server config

# Firewall (iptables/nftables/firewalld)
# Ubuntu — UFW (Uncomplicated Firewall)
ufw status verbose              # Check firewall status
ufw allow 22/tcp                # Allow SSH
ufw allow 80/tcp                # Allow HTTP
ufw deny 8080                   # Deny port 8080
ufw enable                      # Enable firewall
ufw disable                     # Disable firewall

# RHEL/CentOS — firewalld
firewall-cmd --state            # Check state
firewall-cmd --list-all         # List all rules
firewall-cmd --add-port=80/tcp --permanent  # Allow port
firewall-cmd --reload           # Apply changes

# Network statistics
sar -n DEV 1 5                  # Network interface stats
nethogs                         # Bandwidth per process (install)
iftop                           # Real-time bandwidth by connection
```

---

## 9. Package Management

### Ubuntu/Debian (apt)

```bash
apt update                          # Update package index
apt upgrade                         # Upgrade all packages
apt install nginx                   # Install package
apt install nginx=1.18.0-0ubuntu1   # Install specific version
apt remove nginx                    # Remove package
apt purge nginx                     # Remove + config files
apt autoremove                      # Remove unused deps
apt search nginx                    # Search for package
apt show nginx                      # Package details
dpkg -l | grep nginx                # Check if installed
dpkg -L nginx                       # List files installed by package
```

### RHEL/CentOS/Amazon Linux (yum/dnf)

```bash
yum update -y                       # Update all packages
yum install nginx -y                # Install package
yum remove nginx                    # Remove package
yum search nginx                    # Search packages
yum info nginx                      # Package info
yum list installed                  # List installed
rpm -qa | grep nginx                # Check if installed
rpm -ql nginx                       # List installed files
rpm -qf /usr/sbin/nginx             # Which package owns file

# dnf (newer, RHEL8+, Amazon Linux 2023)
dnf install nginx -y
dnf update --security -y            # Security updates only
dnf module list                     # List available modules
```

---

## 10. Systemd and Services

Systemd is the init system managing services on modern Linux:

```bash
# Service management
systemctl start nginx               # Start service
systemctl stop nginx                # Stop service
systemctl restart nginx             # Stop then start
systemctl reload nginx              # Reload config (graceful)
systemctl enable nginx              # Enable on boot
systemctl disable nginx             # Disable on boot
systemctl status nginx              # Check status and recent logs
systemctl is-active nginx           # Returns "active" or "inactive"
systemctl is-enabled nginx          # Returns "enabled" or "disabled"

# List services
systemctl list-units --type=service           # Running services
systemctl list-units --type=service --all     # All services
systemctl list-unit-files --type=service      # All with state

# System power
systemctl reboot                    # Reboot system
systemctl poweroff                  # Shutdown
systemctl halt                      # Halt

# Journald logs
journalctl -u nginx                 # Logs for nginx service
journalctl -u nginx --since "1 hour ago"  # Last hour logs
journalctl -u nginx -f              # Follow (like tail -f)
journalctl -u nginx --no-pager | tail -50  # Last 50 lines
journalctl -p err                   # Only error-level logs
journalctl --since "2024-01-01" --until "2024-01-02"  # Date range
journalctl --disk-usage             # Journal disk usage
```

### Creating a Custom Service

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application Service
After=network.target
Requires=network.target

[Service]
Type=simple
User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/app.py
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
Environment=ENV=production
EnvironmentFile=/opt/myapp/.env

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload             # Reload after creating/editing unit
systemctl enable --now myapp        # Enable and start immediately
```

---

## 11. Logs and Monitoring

```bash
# System logs
tail -f /var/log/syslog             # System log (Ubuntu)
tail -f /var/log/messages           # System log (RHEL)
tail -f /var/log/auth.log           # Auth log — SSH logins, sudo
cat /var/log/boot.log               # Boot messages
dmesg                               # Kernel ring buffer
dmesg | grep -i error               # Kernel errors
dmesg | tail -20                    # Recent kernel messages

# Application logs
tail -f /var/log/nginx/access.log   # Nginx access
tail -f /var/log/nginx/error.log    # Nginx errors
tail -f /var/log/mysql/error.log    # MySQL errors

# Log analysis patterns
# Count errors per hour
awk '{print $4}' access.log | cut -d: -f2 | sort | uniq -c

# Top 10 IP addresses
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head 10

# All 500 errors
grep '" 500 ' access.log

# Logrotate config (automatic log rotation)
cat /etc/logrotate.conf
ls /etc/logrotate.d/

# System monitoring
top                         # CPU and memory (press 1 for per-core)
htop                        # Better top
free -h                     # Memory usage
free -h | grep Mem          # Just RAM line
vmstat 1 5                  # Virtual memory stats (1sec, 5 times)
iostat -xz 1                # I/O stats per device
sar -u 1 5                  # CPU usage history
uptime                      # Load average (1, 5, 15 min)
w                           # Who is logged in + load average
last                        # Recent logins
lastlog                     # Last login for all users
```

---

## 12. Users and Groups

```bash
# User management
useradd username                    # Create user (minimal)
useradd -m -s /bin/bash username    # Create with home dir and bash shell
useradd -m -G sudo,docker username  # Add to groups on creation
usermod -aG docker ubuntu           # Add user to group (append)
usermod -s /bin/bash username       # Change shell
usermod -L username                 # Lock user account
usermod -U username                 # Unlock user account
userdel username                    # Delete user
userdel -r username                 # Delete user + home dir
passwd username                     # Set/change password
passwd -e username                  # Expire password (force change)

# View user info
id username                         # UID, GID, groups
whoami                              # Current user
cat /etc/passwd                     # All users (user:x:uid:gid:comment:home:shell)
cat /etc/shadow                     # Hashed passwords (root only)
getent passwd username              # User details

# Group management
groupadd developers                 # Create group
groupdel developers                 # Delete group
groups username                     # Show user's groups
cat /etc/group                      # All groups

# Sudo access
visudo                              # Edit sudoers file safely
# Add to /etc/sudoers.d/myuser:
echo "myuser ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/myuser
chmod 440 /etc/sudoers.d/myuser

# Switch user
su - username                       # Switch to user (full env)
sudo -u username command            # Run command as user
sudo -i                             # Switch to root with env
sudo su -                           # Switch to root
```

---

## 13. SSH and Remote Access

```bash
# SSH connection
ssh user@hostname                   # Basic SSH
ssh -p 2222 user@hostname           # Custom port
ssh -i ~/.ssh/mykey.pem user@host   # Specific key
ssh -L 8080:localhost:80 user@host  # Local port forwarding
ssh -R 8080:localhost:80 user@host  # Remote port forwarding
ssh -D 1080 user@host               # SOCKS proxy

# SSH config (~/.ssh/config)
Host myserver
    HostName 54.123.45.67
    User ubuntu
    IdentityFile ~/.ssh/aws-key.pem
    Port 22
    ServerAliveInterval 60

# Now just: ssh myserver

# SSH key management
ssh-keygen -t ed25519 -C "user@company.com"  # Generate key (Ed25519)
ssh-keygen -t rsa -b 4096                    # Generate RSA key
ssh-copy-id user@hostname                     # Copy pub key to server
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys  # Manual copy
ssh-add ~/.ssh/id_ed25519                    # Add key to agent
ssh-add -l                                   # List loaded keys
eval $(ssh-agent)                            # Start agent

# SCP (secure copy)
scp file.txt user@host:/tmp/               # Upload file
scp user@host:/tmp/file.txt .              # Download file
scp -r directory/ user@host:/opt/          # Upload directory
scp -P 2222 file.txt user@host:/tmp/       # Custom port

# Rsync (efficient sync)
rsync -avz ./local/ user@host:/remote/     # Sync directory
rsync -avz --delete ./local/ user@host:/remote/  # Mirror (delete extra)
rsync -avz --exclude='*.log' ./app/ user@host:/app/  # Exclude files

# SSH Hardening (/etc/ssh/sshd_config)
PermitRootLogin no              # Disable root SSH
PasswordAuthentication no       # Keys only
MaxAuthTries 3                  # Limit auth attempts
AllowUsers ubuntu deploy        # Whitelist users
Port 2222                       # Change default port
# After editing: systemctl restart sshd
```

---

## 14. Cron Jobs and Scheduling

```bash
# Crontab
crontab -e                      # Edit user crontab
crontab -l                      # List user crontab
crontab -r                      # Remove user crontab
crontab -u username -l          # List another user's crontab

# Cron syntax:
# ┌─── minute (0-59)
# │ ┌─── hour (0-23)
# │ │ ┌─── day of month (1-31)
# │ │ │ ┌─── month (1-12)
# │ │ │ │ ┌─── day of week (0-7, 0=Sunday)
# │ │ │ │ │
# * * * * * command

# Examples
*/5 * * * * /opt/scripts/health-check.sh        # Every 5 minutes
0 * * * * /opt/scripts/hourly-report.sh         # Every hour
0 2 * * * /opt/scripts/backup.sh                # Daily at 2AM
0 2 * * 0 /opt/scripts/weekly-cleanup.sh        # Weekly, Sunday 2AM
0 2 1 * * /opt/scripts/monthly-report.sh        # Monthly, 1st at 2AM
*/15 9-17 * * 1-5 /opt/scripts/check.sh         # Every 15min, 9-5, weekdays

# Redirect cron output
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1

# At command (one-time scheduling)
at 10:30 tomorrow                # Schedule for specific time
at -f script.sh 10:30 tomorrow   # Run script
atq                              # List scheduled jobs
atrm 2                           # Remove job #2
```

---

## 15. Linux Performance Tuning

```bash
# CPU Performance
cat /proc/cpuinfo                   # CPU details
nproc                               # Number of CPU cores
uptime                              # Load averages
# Load average meaning:
# Value = 1.0 → 100% utilization on 1 core
# 1-core: load 1.0 = saturated; load 0.7 = healthy
# 4-core: load 4.0 = saturated; load 2.0 = healthy

# Memory Performance
free -h                             # RAM and swap usage
cat /proc/meminfo                   # Detailed memory info
sysctl vm.swappiness                # Swap preference (default 60)
sysctl -w vm.swappiness=10          # Reduce swapping
vmstat -s                           # Memory statistics

# I/O Performance
iostat -xz 1                        # Per-device I/O stats (extended)
iotop                               # I/O by process (like top)
hdparm -Tt /dev/sda                 # Disk read benchmark
dd if=/dev/zero of=/tmp/test bs=1G count=1  # Write speed test

# Network Performance
iperf3 -s                           # Server mode
iperf3 -c server-ip                 # Client test
ss -s                               # Socket statistics
cat /proc/net/dev                   # Network interface stats

# System call tracing
strace command                      # Trace system calls
strace -p 1234                      # Trace running process
ltrace command                      # Trace library calls

# Key sysctl tunings for servers
sysctl net.core.somaxconn           # Max listen queue
sysctl net.ipv4.tcp_max_syn_backlog # TCP SYN backlog
# Apply permanently: /etc/sysctl.conf or /etc/sysctl.d/99-custom.conf
```

---

## 16. Security Hardening

```bash
# Check for failed login attempts
grep "Failed password" /var/log/auth.log | tail -20
grep "Failed password" /var/log/secure | tail -20  # RHEL

# Check sudo usage
grep "sudo" /var/log/auth.log | tail -20

# Find SUID/SGID files (potential security risks)
find / -perm /4000 -type f 2>/dev/null     # SUID files
find / -perm /2000 -type f 2>/dev/null     # SGID files

# Find world-writable files
find / -perm -002 -type f 2>/dev/null

# Check open ports
ss -tulpn
nmap -sV localhost

# Check running services
systemctl list-units --type=service --state=running

# Check for unauthorized users
cat /etc/passwd | grep "/bin/bash"         # Users with shell access
awk -F: '$3 == 0' /etc/passwd              # UID 0 (root) users

# File integrity monitoring
md5sum /usr/sbin/sshd                      # Get current hash
sha256sum /etc/passwd                      # Hash important files

# Fail2ban (automated IP banning)
fail2ban-client status                     # Overall status
fail2ban-client status sshd               # SSH jail status
fail2ban-client set sshd banip 1.2.3.4   # Manually ban IP
fail2ban-client set sshd unbanip 1.2.3.4 # Unban IP
```

---

## 17. Real-World DevOps Use Cases

### Use Case 1: Server Health Check Script
```bash
#!/bin/bash
# Run this to quickly assess server health
echo "=== CPU Load ==="
uptime
echo "=== Memory ==="
free -h
echo "=== Disk ==="
df -h | grep -v tmpfs
echo "=== Top 5 CPU Processes ==="
ps aux --sort=-%cpu | head -6
echo "=== Top 5 Memory Processes ==="
ps aux --sort=-%mem | head -6
echo "=== Failed Services ==="
systemctl list-units --state=failed
echo "=== Last 10 SSH Logins ==="
last | head -10
```

### Use Case 2: Disk Space Alert
```bash
#!/bin/bash
THRESHOLD=80
df -h | grep -vE '^Filesystem|tmpfs|cdrom' | awk '{print $5 " " $6}' | while read output; do
    usage=$(echo $output | awk '{print $1}' | sed 's/%//')
    partition=$(echo $output | awk '{print $2}')
    if [ $usage -ge $THRESHOLD ]; then
        echo "WARNING: $partition at ${usage}% usage" | mail -s "Disk Alert" admin@company.com
    fi
done
```

### Use Case 3: Log Monitoring and Alerting
```bash
# Watch nginx error log for 500 errors and send alert
tail -F /var/log/nginx/error.log | while read line; do
    if echo "$line" | grep -q "500"; then
        echo "$line" | mail -s "Nginx 500 Error" ops@company.com
    fi
done
```

### Use Case 4: Automated User Provisioning
```bash
#!/bin/bash
# Add developer with proper groups and SSH key
create_dev_user() {
    USERNAME=$1
    SSH_PUB_KEY=$2
    
    useradd -m -s /bin/bash -G docker,sudo "$USERNAME"
    mkdir -p /home/$USERNAME/.ssh
    echo "$SSH_PUB_KEY" >> /home/$USERNAME/.ssh/authorized_keys
    chmod 700 /home/$USERNAME/.ssh
    chmod 600 /home/$USERNAME/.ssh/authorized_keys
    chown -R $USERNAME:$USERNAME /home/$USERNAME/.ssh
    echo "$USERNAME created successfully"
}
```

---

## 18. Common Interview Questions

**Q1: What is the difference between a process and a thread in Linux?**
> A process is an independent program with its own memory space, file descriptors, and PID. A thread is a lightweight execution unit within a process, sharing the same memory space. Processes communicate via IPC (pipes, sockets, shared memory); threads share memory directly.

**Q2: How do you find which process is using port 80?**
> `ss -tulpn | grep :80` or `lsof -i :80`. This shows the PID and process name. Then `ps aux | grep <PID>` for more details.

**Q3: Explain the difference between soft links and hard links.**
> Hard link: another directory entry pointing to the same inode (same data). Deleting one doesn't affect the other. Cannot cross filesystems. Soft/symbolic link: a file containing a path to another file. Deleting the target breaks the link. Can cross filesystems and link to directories.

**Q4: A server has high load average but low CPU usage. What could be causing this?**
> Load average includes processes waiting for I/O, not just CPU. High I/O wait (`iostat`, `top` shows `wa%`) means disk bottleneck — slow disk, disk failure, or excessive I/O. Other causes: memory pressure causing swap, zombie processes.

**Q5: How do you find a file containing a specific string across all files in a directory?**
> `grep -r "string" /path/to/dir/` — recursive search. Add `-l` to only show filenames, `-n` for line numbers, `-i` for case-insensitive.

**Q6: What is /proc filesystem?**
> /proc is a virtual filesystem (not on disk) exposing kernel and process information. `/proc/cpuinfo` — CPU details; `/proc/meminfo` — memory stats; `/proc/[pid]/` — per-process info (maps, fd, status). Reading these doesn't do disk I/O; kernel generates data on-demand.

**Q7: How do you check if a service will start on boot?**
> `systemctl is-enabled service-name`. Returns `enabled`, `disabled`, or `static`. Use `systemctl enable service-name` to enable it.

**Q8: What is the sticky bit and where is it used?**
> Sticky bit on a directory means only the file owner (or root) can delete files within it, even if others have write permission on the directory. Classic example: `/tmp` has sticky bit (`drwxrwxrwt`) — all users can create files, but can only delete their own.

**Q9: How do you check memory usage and identify memory leaks?**
> `free -h` for overall; `ps aux --sort=-%mem` for top consumers; `top` then press `M` to sort by memory; `valgrind` for C/C++ leak detection; monitor `/proc/[pid]/status` for `VmRSS` growth over time.

**Q10: Explain runlevels / systemd targets.**
> Runlevels (SysV) → Targets (systemd): 0=poweroff, 1=rescue (single-user), 3=multi-user.target (no GUI), 5=graphical.target, 6=reboot. `systemctl get-default` shows current target. `systemctl set-default multi-user.target` sets it permanently.

**Q11: How do you troubleshoot SSH connection refused?**
> 1. Verify SSH service running: `systemctl status sshd`. 2. Check port: `ss -tulpn | grep :22`. 3. Check firewall: `ufw status` or `firewall-cmd --list-all`. 4. Check security group (AWS). 5. Check `/var/log/auth.log` for errors. 6. Verify sshd_config: `sshd -t` to test config.

**Q12: What is inode and when do you run out of disk space even with free space available?**
> Inode is a data structure storing file metadata (permissions, owner, timestamps, block locations). Every file uses one inode. `df -i` shows inode usage. You can run out of inodes while disk has free space — common with many small files (caches, email). Solution: delete many small files or reformat with more inodes.

**Q13: How do you increase the open file limit for a process?**
> `ulimit -n 65536` (for current shell session). Permanently: add to `/etc/security/limits.conf`: `* soft nofile 65536` and `* hard nofile 65536`. For systemd services: add `LimitNOFILE=65536` in the `[Service]` section.

**Q14: How do you find and kill zombie processes?**
> Zombies are dead processes waiting for parent to read exit status. `ps aux | grep Z` finds them. You cannot kill zombies (already dead); kill the parent process: `kill -SIGCHLD <parent_pid>` or if parent is stuck, `kill -9 <parent_pid>`.

**Q15: What is a named pipe (FIFO)?**
> Named pipe is a special file used for IPC between unrelated processes. Unlike regular pipes (only related processes), named pipes persist in the filesystem. Create: `mkfifo /tmp/mypipe`. One process writes, another reads. Data flows in FIFO order.

---

*Next: [Linux Assignments](02-linux-assignments.md)*
