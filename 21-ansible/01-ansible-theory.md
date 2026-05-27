# Ansible — Complete Guide

## Table of Contents
1. [What is Ansible?](#1-what-is-ansible)
2. [Ansible Architecture](#2-ansible-architecture)
3. [Installation and Setup](#3-installation-and-setup)
4. [Inventory](#4-inventory)
5. [Playbooks](#5-playbooks)
6. [Variables and Facts](#6-variables-and-facts)
7. [Tasks and Modules](#7-tasks-and-modules)
8. [Handlers](#8-handlers)
9. [Roles](#9-roles)
10. [Templates (Jinja2)](#10-templates-jinja2)
11. [Ansible Vault](#11-ansible-vault)
12. [Ansible Galaxy](#12-ansible-galaxy)
13. [Ansible with AWS](#13-ansible-with-aws)
14. [Real-World Playbooks](#14-real-world-playbooks)
15. [Common Interview Questions](#15-common-interview-questions)

---

## 1. What is Ansible?

**Ansible** is an agentless IT automation tool for configuration management, application deployment, and task automation.

```
Traditional Server Config (Manual):      Ansible (Automated):
──────────────────────────────────────   ──────────────────────────────────
SSH to each server                        Run one playbook
Run same commands 50 times               Executes on 50 servers in parallel
Forget one step = broken server          Idempotent — same result always
No documentation of what was done        Playbook IS the documentation
Drift between servers over time          All servers identical
Hours of work                            Minutes
```

**Ansible vs Chef vs Puppet:**
| Feature | Ansible | Chef | Puppet |
|---------|---------|------|--------|
| Agent required | No (agentless) | Yes | Yes |
| Language | YAML | Ruby (DSL) | Puppet DSL |
| Communication | SSH/WinRM | Pull (chef-client) | Pull (puppet agent) |
| Learning curve | Easy | Hard | Medium |
| Execution | Push (control node) | Pull | Pull |
| State management | Limited | Full | Full |
| Best for | Ad-hoc + config mgmt | Complex orgs | Large enterprises |

---

## 2. Ansible Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Control Node                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 Ansible Engine                       │   │
│  │  ┌──────────┐  ┌───────────┐  ┌──────────────────┐  │   │
│  │  │Inventory │  │ Playbooks │  │    Modules       │  │   │
│  │  │(hosts)   │  │ (YAML)    │  │ (apt, yum, copy) │  │   │
│  │  └──────────┘  └───────────┘  └──────────────────┘  │   │
│  │  ┌──────────┐  ┌───────────┐  ┌──────────────────┐  │   │
│  │  │  Roles   │  │ Variables │  │    Templates     │  │   │
│  │  │          │  │ & Facts   │  │   (Jinja2)       │  │   │
│  │  └──────────┘  └───────────┘  └──────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────┘
                               │ SSH (port 22) — No agent needed!
       ┌───────────────────────┼───────────────────────┐
       │                       │                       │
┌──────▼──────┐         ┌──────▼──────┐         ┌──────▼──────┐
│  Managed    │         │  Managed    │         │  Managed    │
│   Node 1    │         │   Node 2    │         │   Node 3    │
│ web-01      │         │ web-02      │         │  db-01      │
│             │         │             │         │             │
│ Only needs: │         │ Only needs: │         │ Only needs: │
│ • SSH open  │         │ • SSH open  │         │ • SSH open  │
│ • Python    │         │ • Python    │         │ • Python    │
└─────────────┘         └─────────────┘         └─────────────┘
```

**How Ansible Works:**
1. Control node reads inventory (which hosts to target)
2. Reads playbook (what tasks to run)
3. Connects via SSH to each host
4. Transfers Python modules as temp files
5. Executes modules on remote host
6. Collects results and reports
7. Cleans up temp files

---

## 3. Installation and Setup

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install -y ansible

# Python pip (any OS)
pip3 install ansible
pip3 install ansible boto3 botocore   # For AWS modules

# From source (latest version)
pip3 install ansible-core

# Version check
ansible --version

# Configuration file (in order of precedence):
# 1. ANSIBLE_CONFIG environment variable
# 2. ./ansible.cfg (current directory)
# 3. ~/.ansible.cfg (user home)
# 4. /etc/ansible/ansible.cfg (global)

# Minimal ansible.cfg
cat > ansible.cfg << 'EOF'
[defaults]
inventory = ./inventory
remote_user = ubuntu
private_key_file = ~/.ssh/aws-key.pem
host_key_checking = False
stdout_callback = yaml
timeout = 30
forks = 10

[privilege_escalation]
become = True
become_method = sudo
become_user = root
EOF
```

---

## 4. Inventory

```ini
# Static inventory: inventory/hosts
# Simple list
[webservers]
web-01 ansible_host=10.0.1.10
web-02 ansible_host=10.0.1.11
web-03 ansible_host=10.0.1.12

[databases]
db-01 ansible_host=10.0.2.10
db-02 ansible_host=10.0.2.11

[cacheservers]
cache-01 ansible_host=10.0.3.10

# Group of groups
[production:children]
webservers
databases
cacheservers

# Variables for a group
[webservers:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/web-key.pem
http_port=80
nginx_worker_processes=4

# Variables for specific host
web-01 ansible_host=10.0.1.10 nginx_worker_processes=8

# Connection types
localhost ansible_connection=local
windows-server ansible_host=10.0.4.10 ansible_connection=winrm
```

```yaml
# YAML inventory (more flexible)
# inventory/hosts.yml
all:
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/key.pem
  children:
    production:
      children:
        webservers:
          hosts:
            web-01:
              ansible_host: 10.0.1.10
              weight: 100
            web-02:
              ansible_host: 10.0.1.11
              weight: 100
          vars:
            nginx_port: 80
            app_env: production
        databases:
          hosts:
            db-01:
              ansible_host: 10.0.2.10
              db_role: primary
            db-02:
              ansible_host: 10.0.2.11
              db_role: replica
    staging:
      hosts:
        staging-01:
          ansible_host: 10.0.5.10
```

```bash
# Dynamic inventory — query AWS, GCP, etc. in real-time
# AWS EC2 dynamic inventory
pip install boto3
# Create aws_ec2.yml plugin file:
cat > inventory/aws_ec2.yml << 'EOF'
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  instance-state-name: running
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: placement.region
    prefix: region
hostnames:
  - private-ip-address
compose:
  ansible_host: private_ip_address
EOF

# Test dynamic inventory
ansible-inventory -i inventory/aws_ec2.yml --list
ansible-inventory -i inventory/aws_ec2.yml --graph

# Run commands
ansible all -m ping                        # Test connectivity
ansible webservers -m ping                 # Target group
ansible web-01 -m ping                     # Target single host
ansible all -m command -a "uptime"         # Run command
ansible all -a "df -h"                     # Shorthand for command module
ansible all -m shell -a "echo $HOSTNAME"   # Shell (supports pipes, vars)
ansible webservers -m service -a "name=nginx state=restarted"  # Manage service
ansible all -b -m apt -a "name=vim state=present"  # Install package
```

---

## 5. Playbooks

```yaml
# playbooks/deploy-app.yml
---
- name: Deploy Application to Web Servers
  hosts: webservers                   # Target from inventory
  become: true                        # Run as sudo
  serial: "30%"                       # Rolling update — 30% at a time
  
  vars:
    app_name: myapp
    app_version: "2.1.0"
    app_dir: "/opt/{{ app_name }}"
    app_user: appuser
  
  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: true
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"
    
    - name: Remove from load balancer
      local_action:
        module: uri
        url: "http://lb-api/remove/{{ inventory_hostname }}"
        method: POST
  
  tasks:
    - name: Create app user
      user:
        name: "{{ app_user }}"
        system: true
        shell: /bin/false
        home: "{{ app_dir }}"
    
    - name: Create app directory
      file:
        path: "{{ app_dir }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0750'
    
    - name: Download application artifact
      get_url:
        url: "https://artifacts.company.com/{{ app_name }}/{{ app_version }}/app.tar.gz"
        dest: "/tmp/{{ app_name }}.tar.gz"
        checksum: "sha256:{{ app_checksum }}"
    
    - name: Extract application
      unarchive:
        src: "/tmp/{{ app_name }}.tar.gz"
        dest: "{{ app_dir }}"
        remote_src: true
        owner: "{{ app_user }}"
    
    - name: Deploy configuration
      template:
        src: app.conf.j2
        dest: "{{ app_dir }}/config/app.conf"
        owner: "{{ app_user }}"
        mode: '0640'
      notify: Restart application     # Notify handler
    
    - name: Ensure application is started and enabled
      systemd:
        name: "{{ app_name }}"
        state: started
        enabled: true
        daemon_reload: true
    
    - name: Wait for application to be healthy
      uri:
        url: "http://localhost:8080/health"
        return_content: true
      register: health_check
      until: health_check.status == 200
      retries: 12
      delay: 5
  
  post_tasks:
    - name: Add back to load balancer
      local_action:
        module: uri
        url: "http://lb-api/add/{{ inventory_hostname }}"
        method: POST
  
  handlers:
    - name: Restart application
      systemd:
        name: "{{ app_name }}"
        state: restarted
```

---

## 6. Variables and Facts

```yaml
# Variable sources (in order of precedence, highest first):
# 1. --extra-vars on command line
# 2. task vars
# 3. include_vars
# 4. role vars (vars/)
# 5. block vars
# 6. play vars
# 7. host_vars file
# 8. group_vars/all
# 9. role defaults (defaults/)
# 10. inventory variables

# group_vars/webservers.yml
nginx_port: 80
nginx_worker_processes: "{{ ansible_processor_vcpus * 2 }}"
max_upload_size: "50m"

# group_vars/production.yml
log_level: "warn"
ssl_enabled: true

# host_vars/web-01.yml
nginx_worker_processes: 8   # Override for this specific host

# Accessing variables in playbook
- name: Configure nginx
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  vars:
    timeout: 30
  when: ssl_enabled | bool

# Register — capture command output
- name: Get current version
  command: "{{ app_dir }}/bin/app --version"
  register: current_version
  changed_when: false    # This never "changes" anything

- name: Show version
  debug:
    msg: "Current version: {{ current_version.stdout }}"

- name: Only deploy if version changed
  # Do deployment
  when: current_version.stdout != app_version

# Ansible Facts (auto-collected info about managed hosts)
- name: Gather facts
  setup:                     # Explicit fact gathering

- name: Show OS info
  debug:
    msg: "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"

# Common facts:
# ansible_hostname              → web-01
# ansible_fqdn                  → web-01.example.com
# ansible_os_family             → Debian, RedHat
# ansible_distribution          → Ubuntu, CentOS
# ansible_distribution_version  → "22.04"
# ansible_processor_vcpus       → 4
# ansible_memtotal_mb           → 8192
# ansible_default_ipv4.address  → 10.0.1.10
# ansible_eth0.ipv4.address     → 10.0.1.10
# ansible_all_ipv4_addresses    → list of IPs
# ansible_mounts                → list of mount points

# Custom facts (files in /etc/ansible/facts.d/*.fact)
# Gets collected as ansible_local.custom_fact_name
```

---

## 7. Tasks and Modules

### Essential Modules

```yaml
# File operations
- name: Create directory
  file:
    path: /opt/myapp
    state: directory
    owner: appuser
    mode: '0755'

- name: Copy file
  copy:
    src: files/app.conf
    dest: /etc/app/app.conf
    owner: root
    mode: '0644'
    backup: true        # Backup before overwriting

- name: Create symlink
  file:
    src: /opt/app/current/app.jar
    dest: /opt/app/app.jar
    state: link

- name: Delete file
  file:
    path: /tmp/old-file.txt
    state: absent

# Package management
- name: Install packages (apt)
  apt:
    name:
      - nginx
      - python3-pip
      - git
    state: present
    update_cache: true

- name: Install specific version
  apt:
    name: "nginx=1.18.0-0ubuntu1"
    state: present

- name: Install packages (yum/dnf)
  yum:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - python3-pip

# Service management
- name: Start and enable nginx
  service:
    name: nginx
    state: started
    enabled: true

- name: Restart with systemd
  systemd:
    name: nginx
    state: restarted
    daemon_reload: true    # Needed after editing unit files

# User management
- name: Create user
  user:
    name: deploy
    groups: "docker,sudo"
    append: true
    shell: /bin/bash
    home: /home/deploy
    create_home: true
    state: present

- name: Add SSH key
  authorized_key:
    user: deploy
    key: "{{ lookup('file', 'files/deploy.pub') }}"
    state: present

# Command execution
- name: Run command
  command: /opt/app/init.sh
  args:
    creates: /opt/app/.initialized   # Don't run if file exists (idempotent)

- name: Run shell command
  shell: |
    cd /opt/app
    ./gradlew build 2>&1 | tee /var/log/build.log
  register: build_result
  failed_when: "'BUILD FAILED' in build_result.stdout"

# Cron jobs
- name: Setup backup cron
  cron:
    name: "Daily DB backup"
    minute: "0"
    hour: "2"
    job: "/opt/scripts/backup.sh >> /var/log/backup.log 2>&1"
    user: backup

# Firewall
- name: Allow HTTP/HTTPS (UFW)
  ufw:
    rule: allow
    port: "{{ item }}"
    proto: tcp
  loop:
    - '80'
    - '443'

# Wait for conditions
- name: Wait for port 8080
  wait_for:
    port: 8080
    host: localhost
    timeout: 120

- name: Wait for URL
  uri:
    url: http://localhost:8080/health
    return_content: true
  register: result
  until: result.status == 200
  retries: 24
  delay: 5

# Debug
- name: Print variable
  debug:
    var: ansible_default_ipv4.address

- name: Print message
  debug:
    msg: "Deploying version {{ app_version }} to {{ inventory_hostname }}"
```

---

## 8. Handlers

Handlers run only when notified, and only once at the end of play:

```yaml
tasks:
  - name: Update nginx.conf
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify:
      - Validate nginx config
      - Reload nginx

  - name: Update SSL certificate
    copy:
      src: certs/server.crt
      dest: /etc/nginx/ssl/server.crt
    notify: Reload nginx    # Won't run if already notified

handlers:
  - name: Validate nginx config
    command: nginx -t
    
  - name: Reload nginx
    service:
      name: nginx
      state: reloaded

  # Listen to multiple notifications
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
    listen: "restart web services"
```

---

## 9. Roles

Roles are the standard way to organize complex Ansible content:

```
roles/
└── nginx/
    ├── tasks/
    │   ├── main.yml        # Main task list
    │   ├── install.yml     # Install tasks
    │   └── configure.yml   # Config tasks
    ├── handlers/
    │   └── main.yml        # Handlers
    ├── templates/
    │   └── nginx.conf.j2   # Jinja2 templates
    ├── files/
    │   └── mime.types      # Static files
    ├── vars/
    │   └── main.yml        # Variables (high precedence)
    ├── defaults/
    │   └── main.yml        # Default variables (low precedence)
    ├── meta/
    │   └── main.yml        # Role metadata and dependencies
    └── README.md
```

```yaml
# roles/nginx/defaults/main.yml
nginx_port: 80
nginx_worker_processes: "auto"
nginx_max_upload_size: "50m"
nginx_keepalive_timeout: 65
ssl_enabled: false

# roles/nginx/tasks/main.yml
---
- import_tasks: install.yml
- import_tasks: configure.yml

# roles/nginx/tasks/install.yml
---
- name: Install nginx
  package:
    name: nginx
    state: present

- name: Ensure nginx started and enabled
  service:
    name: nginx
    state: started
    enabled: true

# roles/nginx/tasks/configure.yml
---
- name: Deploy nginx configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    validate: "/usr/sbin/nginx -t -c %s"
  notify: Reload nginx

# roles/nginx/handlers/main.yml
---
- name: Reload nginx
  service:
    name: nginx
    state: reloaded

- name: Restart nginx
  service:
    name: nginx
    state: restarted
```

```yaml
# Playbook using roles
- name: Configure web servers
  hosts: webservers
  become: true
  
  roles:
    - role: nginx
      vars:
        nginx_port: 443
        ssl_enabled: true
    
    - role: app
    
    - role: monitoring

  # Or with role dependencies (meta/main.yml):
  # dependencies:
  #   - { role: common, tags: always }
  #   - { role: ssl-certs, when: ssl_enabled }
```

```bash
# Create role skeleton
ansible-galaxy role init nginx
```

---

## 10. Templates (Jinja2)

```jinja2
{# templates/nginx.conf.j2 #}
# Managed by Ansible — DO NOT EDIT MANUALLY
# Generated for {{ inventory_hostname }} on {{ ansible_date_time.date }}

user www-data;
worker_processes {{ nginx_worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections | default(1024) }};
}

http {
    sendfile on;
    keepalive_timeout {{ nginx_keepalive_timeout }};
    client_max_body_size {{ nginx_max_upload_size }};
    
    # Logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
    
    # Server blocks for each vhost
    {% for vhost in nginx_vhosts %}
    server {
        listen {{ nginx_port }};
        server_name {{ vhost.name }};
        root {{ vhost.docroot }};
        
        {% if ssl_enabled %}
        listen 443 ssl;
        ssl_certificate /etc/nginx/ssl/{{ vhost.name }}.crt;
        ssl_certificate_key /etc/nginx/ssl/{{ vhost.name }}.key;
        {% endif %}
        
        {% if vhost.proxy_pass is defined %}
        location / {
            proxy_pass {{ vhost.proxy_pass }};
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        {% endif %}
    }
    {% endfor %}
    
    # Include upstream blocks
    {% for upstream in nginx_upstreams | default([]) %}
    upstream {{ upstream.name }} {
        {% for server in upstream.servers %}
        server {{ server }} weight={{ upstream.weight | default(1) }};
        {% endfor %}
    }
    {% endfor %}
}
```

---

## 11. Ansible Vault

Encrypt sensitive data:

```bash
# Encrypt a file
ansible-vault encrypt group_vars/production/secrets.yml

# Decrypt (view)
ansible-vault view group_vars/production/secrets.yml

# Edit encrypted file
ansible-vault edit group_vars/production/secrets.yml

# Encrypt a string (embed in YAML)
ansible-vault encrypt_string 'supersecret' --name 'db_password'
# Output:
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   ...

# Run playbook with vault password
ansible-playbook deploy.yml --ask-vault-pass
ansible-playbook deploy.yml --vault-password-file ~/.vault_pass
# Or set environment variable:
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass

# Rotate vault password
ansible-vault rekey secrets.yml
```

```yaml
# group_vars/production/secrets.yml (encrypted)
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  66386439653236383837636163663831306563353366346462383634383737...

# Use just like a normal variable
- name: Configure database
  template:
    src: db.conf.j2
    dest: /etc/app/db.conf
  vars:
    password: "{{ db_password }}"
```

---

## 12. Ansible Galaxy

```bash
# Install roles from Galaxy
ansible-galaxy role install geerlingguy.nginx
ansible-galaxy role install geerlingguy.mysql
ansible-galaxy role install geerlingguy.docker

# Install from requirements file
cat > requirements.yml << 'EOF'
roles:
  - name: geerlingguy.nginx
    version: "3.1.4"
  - name: geerlingguy.mysql
  - src: git+https://github.com/company/ansible-role-app.git
    name: app
    version: main

collections:
  - name: amazon.aws
    version: ">=6.0.0"
  - name: community.general
  - name: community.docker
EOF

ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml

# List installed
ansible-galaxy role list
ansible-galaxy collection list

# Init new role
ansible-galaxy role init my-new-role
```

---

## 13. Ansible with AWS

```yaml
# install amazon.aws collection first
# ansible-galaxy collection install amazon.aws

# playbooks/aws-provisioning.yml
---
- name: Provision AWS Infrastructure
  hosts: localhost
  connection: local
  gather_facts: false
  
  vars:
    region: us-east-1
    vpc_cidr: "10.0.0.0/16"
    instance_type: t3.micro
    
  tasks:
    - name: Create VPC
      amazon.aws.ec2_vpc_net:
        name: ansible-vpc
        cidr_block: "{{ vpc_cidr }}"
        region: "{{ region }}"
        tags:
          Environment: production
          ManagedBy: ansible
      register: vpc
    
    - name: Create subnet
      amazon.aws.ec2_vpc_subnet:
        vpc_id: "{{ vpc.vpc.id }}"
        cidr: "10.0.1.0/24"
        az: "{{ region }}a"
        region: "{{ region }}"
        tags:
          Name: public-subnet-1
      register: subnet
    
    - name: Create security group
      amazon.aws.ec2_security_group:
        name: web-sg
        description: Web server security group
        vpc_id: "{{ vpc.vpc.id }}"
        region: "{{ region }}"
        rules:
          - proto: tcp
            ports: [80, 443]
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            ports: [22]
            cidr_ip: "{{ my_ip }}/32"
      register: sg
    
    - name: Launch EC2 instances
      amazon.aws.ec2_instance:
        name: "web-{{ item }}"
        key_name: my-keypair
        instance_type: "{{ instance_type }}"
        image_id: ami-0c55b159cbfafe1f0
        wait: true
        security_group: "{{ sg.group_id }}"
        vpc_subnet_id: "{{ subnet.subnet.id }}"
        region: "{{ region }}"
        tags:
          Role: webserver
          Environment: production
      loop: [1, 2, 3]
      register: ec2_instances
    
    - name: Add new instances to inventory
      add_host:
        hostname: "{{ item.public_ip_address }}"
        groupname: new_webservers
      loop: "{{ ec2_instances.results | map(attribute='instances') | flatten }}"

- name: Configure new instances
  hosts: new_webservers
  become: true
  roles:
    - nginx
    - app
```

---

## 14. Real-World Playbooks

### Complete Server Bootstrap Playbook

```yaml
# playbooks/bootstrap-server.yml
---
- name: Bootstrap Ubuntu Server
  hosts: "{{ target_hosts | default('all') }}"
  become: true
  gather_facts: true
  
  vars:
    system_packages:
      - vim
      - curl
      - wget
      - git
      - htop
      - jq
      - unzip
      - net-tools
      - python3-pip
    
    deploy_users:
      - { name: deploy, groups: "sudo,docker", key: "files/keys/deploy.pub" }
    
    ntp_servers:
      - "0.pool.ntp.org"
      - "1.pool.ntp.org"
  
  tasks:
    - name: Update all packages
      apt:
        upgrade: dist
        update_cache: true
      when: ansible_os_family == "Debian"
    
    - name: Install essential packages
      apt:
        name: "{{ system_packages }}"
        state: present
    
    - name: Set timezone
      timezone:
        name: UTC
    
    - name: Configure NTP
      template:
        src: templates/timesyncd.conf.j2
        dest: /etc/systemd/timesyncd.conf
      notify: Restart timesyncd
    
    - name: Harden SSH configuration
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
        validate: "/usr/sbin/sshd -t -f %s"
      loop:
        - { regexp: "^PermitRootLogin", line: "PermitRootLogin no" }
        - { regexp: "^PasswordAuthentication", line: "PasswordAuthentication no" }
        - { regexp: "^MaxAuthTries", line: "MaxAuthTries 3" }
        - { regexp: "^X11Forwarding", line: "X11Forwarding no" }
      notify: Restart sshd
    
    - name: Configure UFW firewall
      ufw:
        state: enabled
        policy: deny
        direction: incoming
    
    - name: Allow SSH
      ufw:
        rule: allow
        port: "22"
        proto: tcp
    
    - name: Create deploy users
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        append: true
        shell: /bin/bash
        create_home: true
      loop: "{{ deploy_users }}"
    
    - name: Add SSH keys for deploy users
      authorized_key:
        user: "{{ item.name }}"
        key: "{{ lookup('file', item.key) }}"
      loop: "{{ deploy_users }}"
    
    - name: Install Docker
      include_role:
        name: geerlingguy.docker
      vars:
        docker_users:
          - "{{ item.name }}"
      loop: "{{ deploy_users }}"
    
    - name: Set kernel parameters for containers
      sysctl:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        sysctl_set: true
        reload: true
      loop:
        - { name: "net.core.somaxconn", value: "1024" }
        - { name: "vm.overcommit_memory", value: "1" }
        - { name: "vm.swappiness", value: "10" }
    
    - name: Setup log rotation
      copy:
        src: files/logrotate-app.conf
        dest: /etc/logrotate.d/app
        mode: '0644'
    
    - name: Install CloudWatch agent
      include_tasks: tasks/cloudwatch-agent.yml
      when: cloud_provider == "aws"
  
  handlers:
    - name: Restart sshd
      service:
        name: sshd
        state: restarted
    
    - name: Restart timesyncd
      service:
        name: systemd-timesyncd
        state: restarted
```

```bash
# Running playbooks
ansible-playbook playbooks/bootstrap-server.yml \
    -i inventory/hosts.yml \
    --limit webservers \
    --extra-vars "target_hosts=webservers" \
    --tags "users,security" \
    --check                        # Dry run
    --diff                         # Show config differences

# Useful flags
ansible-playbook playbook.yml \
    -v                             # Verbose
    -vv                            # More verbose
    -vvv                           # Connection debugging
    --list-hosts                   # Show which hosts would be targeted
    --list-tasks                   # List all tasks
    --list-tags                    # List all tags
    --start-at-task "Install nginx"  # Start from specific task
    --step                         # Confirm each task
    --forks 20                     # Parallel execution
    --timeout 60                   # SSH timeout
```

---

## 15. Common Interview Questions

**Q1: What is the difference between Ansible and other config management tools like Chef/Puppet?**
> Ansible is agentless — connects via SSH, only needs Python on managed nodes. Push-based (control node sends tasks). YAML-based, easy to learn. Chef/Puppet are agent-based — agents pull configs from master, Ruby-based, steeper learning curve. Ansible is better for ad-hoc tasks and simpler environments; Chef/Puppet for complex enterprise with drift detection.

**Q2: What is idempotency in Ansible?**
> Running a playbook multiple times produces the same result. Example: `state: present` installs a package if missing, does nothing if already installed. `changed=0` in output means no changes were needed. Most Ansible modules are idempotent. `command` and `shell` modules are NOT — use `creates:` or `when:` to make them idempotent.

**Q3: What is the difference between `include_tasks` and `import_tasks`?**
> `import_tasks`: static — loaded and processed at parse time. Supports `tags` and `when` applied at import level. `include_tasks`: dynamic — loaded at runtime. Supports looping. Use `include_tasks` with loops or dynamic file names. Use `import_tasks` for static task files where you want tags/when to apply.

**Q4: How does Ansible handle errors?**
> By default, stops on the host that failed (other hosts continue). `ignore_errors: true` — continue despite errors. `failed_when` — custom failure condition. `block/rescue/always` — try/catch/finally. `max_fail_percentage` on play level. `any_errors_fatal: true` — stop all hosts on any failure.

**Q5: What is an Ansible Role and what are its advantages?**
> Role is a structured way to organize tasks, handlers, templates, variables, and files. Advantages: reusable across projects, shareable via Galaxy, testable independently, promotes consistent structure, separation of concerns. Use roles for: nginx setup, user management, Docker installation, application deployment.

**Q6: How do you run Ansible playbooks in CI/CD?**
> In Jenkins/GitHub Actions: (1) Install Ansible in CI environment. (2) Configure SSH key as CI secret. (3) Run playbook with `--inventory`, `--extra-vars`. (4) Use `--check` for dry-run on PR. (5) Apply on merge to main. Use dynamic inventory for cloud environments. Store vault password as CI secret.

**Q7: What is Ansible Galaxy?**
> Community hub for sharing Ansible roles and collections. `ansible-galaxy role install geerlingguy.nginx` downloads community role. Use `requirements.yml` to version-pin dependencies. Collections extend Ansible with new modules, roles, plugins (like `amazon.aws` for AWS modules).

**Q8: What is the difference between `vars` and `defaults` in a role?**
> `vars/`: High precedence — harder to override from outside the role. Use for role-internal constants. `defaults/`: Lowest precedence — easily overridden by inventory vars, group_vars, command line. Use for defaults you expect users to override. If you want users to customize, put in defaults. If it must not change, put in vars.

**Q9: How do you test Ansible playbooks?**
> `--check` (dry run — shows what would change). `--diff` (shows file content differences). **Molecule**: full testing framework — creates VMs/containers, runs playbook, runs tests, destroys. `ansible-lint`: YAML/best-practice linting. **Testinfra/InSpec**: test resulting server state. `ansible-playbook --syntax-check`: syntax validation.

**Q10: How does Ansible handle secrets?**
> Ansible Vault: encrypt files or individual strings. Store vault password in CI as secret env var or file. `ansible-vault encrypt_string 'value' --name 'var_name'`. For cloud deployments: use AWS Secrets Manager, HashiCorp Vault, or pull-at-runtime. Never commit unencrypted secrets to git.

---

*Next: [Ansible Assignments](02-ansible-assignments.md)*
