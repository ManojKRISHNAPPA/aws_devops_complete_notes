# Ansible — Assignments

## Assignment 1: First Ansible Steps

**Setup and test connectivity:**

```bash
# 1. Install Ansible
pip3 install ansible

# 2. Create project structure
mkdir ansible-lab && cd ansible-lab
mkdir -p inventory group_vars host_vars roles

# 3. Create inventory
cat > inventory/hosts.yml << 'EOF'
all:
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
  children:
    webservers:
      hosts:
        web-01:
          ansible_host: 127.0.0.1
          ansible_connection: local  # For local testing
    databases:
      hosts:
        db-01:
          ansible_host: 127.0.0.1
          ansible_connection: local
EOF

# 4. Create ansible.cfg
cat > ansible.cfg << 'EOF'
[defaults]
inventory = ./inventory
host_key_checking = False
stdout_callback = yaml
roles_path = ./roles
EOF

# 5. Test connectivity
ansible all -m ping

# 6. Gather facts
ansible web-01 -m setup | head -50
ansible web-01 -m setup -a "filter=ansible_distribution*"

# 7. Run ad-hoc commands
ansible all -m command -a "uptime"
ansible all -m shell -a "df -h | grep /"
ansible webservers -m apt -a "name=curl state=present" -b
```

---

## Assignment 2: Write Your First Playbook

**Task: Configure a complete web server:**

```yaml
# playbooks/configure-webserver.yml
# This playbook should:
# 1. Update apt cache
# 2. Install nginx, curl, git, python3-pip
# 3. Create group 'webteam'
# 4. Create user 'webadmin' in group webteam
# 5. Deploy nginx configuration from template
# 6. Create document root /var/www/html
# 7. Deploy a sample index.html
# 8. Enable and start nginx
# 9. Open port 80 in ufw
# 10. Verify nginx returns 200 OK with curl

# Template: templates/nginx.conf.j2
# should include:
# - server_name from inventory var
# - worker_processes from fact ansible_processor_vcpus
# - access log to /var/log/nginx/{{ inventory_hostname }}-access.log
```

**Run and verify:**
```bash
# Syntax check
ansible-playbook playbooks/configure-webserver.yml --syntax-check

# Dry run
ansible-playbook playbooks/configure-webserver.yml --check --diff

# Run
ansible-playbook playbooks/configure-webserver.yml -v

# Verify idempotency - run again, should show 0 changes
ansible-playbook playbooks/configure-webserver.yml

# Check result
curl http://localhost/
```

---

## Assignment 3: Variables, Facts, and Conditionals

**Task: Multi-OS package installation playbook:**

```yaml
# playbooks/install-devtools.yml
# Install development tools - works on both Ubuntu AND CentOS/RHEL
#
# Tools to install (different package names on different OS):
# Ubuntu: build-essential, python3-pip, python3-venv
# CentOS: gcc, python3-pip, python3-venv
# Both: git, curl, wget, vim, htop, jq, tree
#
# After installation:
# - Install Python packages via pip: boto3, requests, flask
# - Configure git global settings (from variables)
# - Create ~/.bashrc aliases for all users in 'developers' group
#
# Variables (in group_vars/all.yml):
# git_user_name: "DevOps Team"
# git_user_email: "devops@company.com"
# developer_aliases:
#   - { name: "ll", command: "ls -lah" }
#   - { name: "k", command: "kubectl" }
#   - { name: "tf", command: "terraform" }
```

**Register and use results:**
```yaml
# After running, create a verification report
- name: Gather installed versions
  shell: "{{ item.cmd }}"
  loop:
    - { name: git, cmd: "git --version" }
    - { name: python3, cmd: "python3 --version" }
    - { name: pip, cmd: "pip3 --version" }
  register: version_results

- name: Print version report
  debug:
    msg: "{{ item.item.name }}: {{ item.stdout }}"
  loop: "{{ version_results.results }}"
```

---

## Assignment 4: Create an Ansible Role

**Build a complete Docker installation role:**

```bash
# Create role skeleton
ansible-galaxy role init roles/docker

# roles/docker/
# ├── defaults/main.yml   → docker_version, docker_users, compose_version
# ├── tasks/main.yml      → import install.yml, import configure.yml
# ├── tasks/install.yml   → add repo, install docker-ce
# ├── tasks/configure.yml → daemon.json, add users to docker group
# ├── handlers/main.yml   → restart docker
# ├── templates/daemon.json.j2  → docker daemon config
# └── vars/main.yml       → OS-specific variables
```

```yaml
# roles/docker/tasks/install.yml
# Should:
# 1. Check if Docker is already installed
# 2. Add Docker's official GPG key
# 3. Add Docker repository
# 4. Install docker-ce, docker-ce-cli, containerd.io, docker-compose-plugin
# 5. Start and enable docker service

# roles/docker/tasks/configure.yml
# Should:
# 1. Deploy daemon.json from template with:
#    - log-driver: json-file
#    - log-opts: max-size 10m, max-file 3
#    - default-address-pools for custom subnet
# 2. Add users to docker group (from docker_users list)
# 3. Notify handler to restart Docker

# roles/docker/defaults/main.yml
docker_version: "latest"
docker_users: []   # List of users to add to docker group
docker_log_max_size: "10m"
docker_log_max_file: "3"
```

**Use the role:**
```yaml
# playbooks/setup-docker-host.yml
- name: Configure Docker hosts
  hosts: all
  become: true
  roles:
    - role: docker
      vars:
        docker_users:
          - ubuntu
          - deploy
```

---

## Assignment 5: Ansible Vault for Secrets

```bash
# 1. Create vault password file
echo "myVaultPassword123!" > ~/.vault_pass
chmod 600 ~/.vault_pass

# Configure ansible.cfg to use it
echo "vault_password_file = ~/.vault_pass" >> ansible.cfg

# 2. Create encrypted secrets file
ansible-vault create group_vars/all/secrets.yml

# Add these secrets (in the editor):
# db_root_password: "SuperSecret123!"
# api_key: "sk-abc123xyz789"
# slack_webhook_url: "https://hooks.slack.com/services/..."

# 3. View the encrypted file
cat group_vars/all/secrets.yml  # Shows encrypted content
ansible-vault view group_vars/all/secrets.yml  # Shows decrypted

# 4. Use secrets in a playbook
cat > playbooks/test-vault.yml << 'EOF'
---
- name: Test Vault Secrets
  hosts: localhost
  connection: local
  tasks:
    - name: Show DB password length (not value!)
      debug:
        msg: "DB password has {{ db_root_password | length }} characters"
    
    - name: Configure app with secrets
      template:
        src: templates/app.conf.j2
        dest: /tmp/app.conf
EOF

# 5. Run the playbook
ansible-playbook playbooks/test-vault.yml

# 6. Rotate the vault password
ansible-vault rekey group_vars/all/secrets.yml
```

---

## Assignment 6: AWS Dynamic Inventory

**Manage EC2 instances with dynamic inventory:**

```bash
# 1. Install required collection
ansible-galaxy collection install amazon.aws

# 2. Create AWS dynamic inventory config
cat > inventory/aws_ec2.yml << 'EOF'
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  instance-state-name: running
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: tags.Environment
    prefix: env
compose:
  ansible_host: private_ip_address
  ansible_user: "'ubuntu'"
EOF

# 3. Test the inventory
ansible-inventory -i inventory/aws_ec2.yml --list
ansible-inventory -i inventory/aws_ec2.yml --graph

# 4. Tag your EC2 instances with:
# Role: webserver
# Environment: production

# 5. Run playbook against all webservers
ansible-playbook -i inventory/aws_ec2.yml \
    playbooks/configure-webserver.yml \
    --limit "role_webserver"
```

---

## Interview Assignment: Complete Deployment Automation

**Scenario:** Your team needs to deploy a Node.js application across 10 servers with zero downtime.

**Write a complete Ansible solution:**

```
ansible-project/
├── ansible.cfg
├── inventory/
│   ├── production/
│   │   └── hosts.yml
│   └── staging/
│       └── hosts.yml
├── group_vars/
│   ├── all/
│   │   ├── vars.yml
│   │   └── secrets.yml (vault encrypted)
│   ├── production.yml
│   └── staging.yml
├── host_vars/
│   └── web-01.yml (if any host-specific overrides)
├── roles/
│   ├── common/      (base config, users, packages)
│   ├── nodejs/      (install Node.js, npm)
│   └── myapp/       (deploy the application)
└── playbooks/
    ├── site.yml          (run everything)
    ├── bootstrap.yml     (first-time setup)
    └── deploy.yml        (just deploy the app)
```

**The deploy.yml playbook must:**
1. Take application version as extra var: `--extra-vars "version=2.1.0"`
2. Deploy to servers in batches of 2 (`serial: 2`)
3. Before each batch: remove from load balancer
4. Deploy new version
5. Run health check
6. If health check fails: rollback to previous version
7. After successful deploy: add back to load balancer
8. Send Slack notification with deployment summary

---

## Cheat Sheet

```bash
# Ad-hoc commands
ansible all -m ping
ansible all -m setup                              # Gather facts
ansible all -m setup -a "filter=ansible_*"       # Filter facts
ansible all -b -m service -a "name=nginx state=restarted"
ansible all -b -m apt -a "name=nginx state=latest update_cache=yes"
ansible all -m copy -a "src=file.conf dest=/etc/app/ mode=0644"
ansible all -m shell -a "uptime"
ansible all -m command -a "df -h"
ansible web-01 -m user -a "name=deploy state=present groups=docker"

# Playbook options
ansible-playbook site.yml
ansible-playbook site.yml --check            # Dry run
ansible-playbook site.yml --diff             # Show file diffs
ansible-playbook site.yml -v/-vv/-vvv        # Verbosity
ansible-playbook site.yml --limit web-01     # Specific host
ansible-playbook site.yml --limit webservers # Specific group
ansible-playbook site.yml --tags "config"    # Run tagged tasks only
ansible-playbook site.yml --skip-tags "users" # Skip tagged tasks
ansible-playbook site.yml --start-at-task "Deploy app"
ansible-playbook site.yml --list-tasks       # List all tasks
ansible-playbook site.yml --list-hosts       # List targeted hosts
ansible-playbook site.yml -e "version=2.1.0" # Extra vars

# Vault
ansible-vault create secrets.yml
ansible-vault edit secrets.yml
ansible-vault view secrets.yml
ansible-vault encrypt_string 'my_password' --name 'db_pass'
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass

# Galaxy
ansible-galaxy role install geerlingguy.docker
ansible-galaxy role list
ansible-galaxy collection install amazon.aws
ansible-galaxy role init myrole
ansible-galaxy install -r requirements.yml
```
