# Ansible Fundamentals: Deep Dive into Configuration Management

## 1. Core Concept

Ansible is an **agentless configuration management and orchestration platform** that:

1. **Manages Infrastructure**: SSH into servers, execute commands (no agents)
2. **Desired State**: Apply idempotent tasks (safe to run repeatedly)
3. **Orchestrates**: Run tasks in sequence, with dependencies
4. **Multiplatform**: Linux, Windows, cloud, containers, network appliances

### Three Core Concepts

**Inventory**:
- List of hosts/groups to manage
- Can be static (INI/YAML file) or dynamic (scripts)
- Each host has variables (IP, OS, role)

```yaml
# inventory.ini
[webservers]
web1.example.com
web2.example.com

[databases]
db1.example.com

[all:vars]
ansible_user=admin
```

**Playbook**:
- YAML file describing tasks to execute
- List of plays (each play targets a host group)
- Each play contains tasks (imperative actions)

```yaml
# playbook.yml
---
- name: Configure webservers
  hosts: webservers
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
    - name: Start nginx
      service:
        name: nginx
        state: started
```

**Module**:
- Single unit of work (install package, start service, copy file)
- Idempotent (safe to run repeatedly)
- Has parameters and return values

```bash
# Module: apt (Debian package manager)
- name: Install nginx
  apt:
    name: nginx        # Parameter: package name
    state: present     # Parameter: desired state
    update_cache: yes  # Parameter: update package cache

# Return value: changed (yes/no)
# If nginx already installed: changed=no
# If nginx just installed: changed=yes (idempotent!)
```

### Ansible Workflow

```
1. User runs
   $ ansible-playbook playbook.yml

2. Ansible reads playbook.yml

3. For each play:
   ├─ Select hosts from inventory
   ├─ Establish SSH connection
   ├─ Push Python module to host (if needed)
   ├─ Execute module
   ├─ Collect return value
   ├─ Evaluate result (changed? failed?)
   └─ Move to next task

4. Output results
   PLAY [Configure webservers]
   TASK [Install nginx]
   changed: [web1]
   changed: [web2]
   
   PLAY RECAP
   web1: ok=1 changed=1 unreachable=0 failed=0
   web2: ok=1 changed=1 unreachable=0 failed=0
```

**Idempotency** (core feature):
```yaml
# Running playbook multiple times is SAFE
# First run: Install nginx
# Second run: nginx already installed, no changes (idempotent!)
# Third run: Still no changes

- name: Install nginx
  apt:
    name: nginx
    state: present

# Running 3 times produces:
# Run 1: changed=yes (installed)
# Run 2: changed=no (already there)
# Run 3: changed=no (already there)
```

---

## 2. How Ansible Works Internally

### Architecture (Agentless)

```
Control Machine
├─ Where playbooks live
├─ SSH client
└─ Python installed

Target Host
├─ SSH server (openssh)
├─ Python installed (2.7+ or 3.6+)
└─ NO agent software needed!

Communication:
Control Machine
  └─ SSH connection (secure)
     └─ Execute command
        └─ Push Python module
           └─ Run module
              └─ Return JSON
```

### Task Execution Flow (Detailed)

```
Step 1: Ansible reads playbook
- name: Install nginx
  apt:
    name: nginx
    state: present

Step 2: Parse task parameters
Module: apt
Parameters: {name: "nginx", state: "present"}

Step 3: Generate module code (Python)
# Inline Python script
import urllib.request
import subprocess

package_name = "nginx"
desired_state = "present"

if desired_state == "present":
    result = subprocess.run(["apt-get", "install", "-y", package_name])
    
# Returns: {"changed": true/false, "failed": false, "state": "present"}

Step 4: Transfer to target host via SSH
$ scp /tmp/ansible_module.py remote:/tmp/

Step 5: Execute on remote host
$ ssh remote "/usr/bin/python /tmp/ansible_module.py"

Step 6: Capture return JSON
{"changed": true, "failed": false, "rc": 0}

Step 7: Process on control machine
If "failed" == true:
  └─ Task failed, stop playbook
If "changed" == true:
  └─ Task changed system state
If "changed" == false:
  └─ System already in desired state (idempotent!)

Step 8: Execute handlers (if needed)
If nginx changed:
  └─ Run handler "restart nginx"
```

### Idempotency Mechanism

```yaml
# Example: apt module (Debian package manager)

# What user writes:
- name: Install nginx
  apt:
    name: nginx
    state: present

# What apt module does internally:
1. Check if nginx installed
   $ dpkg -l | grep nginx
   
2. If NOT installed:
   ├─ Run: apt-get install -y nginx
   └─ Return: "changed": true
   
3. If ALREADY installed:
   ├─ Do nothing
   └─ Return: "changed": false

# Result: Safe to run multiple times
# - First run: Installs (changed=yes)
# - Subsequent runs: No-op (changed=no)
```

### Fact Gathering (Context Collection)

```yaml
# Before running tasks, Ansible gathers system facts

- name: Configure system
  hosts: all
  gather_facts: yes    # Enabled by default
  tasks:
    - debug: msg={{ ansible_os_family }}  # Linux
    - debug: msg={{ ansible_distribution }}  # Ubuntu
    - debug: msg={{ ansible_distribution_version }}  # 22.04

# Ansible collects 50+ facts:
ansible_os_family: Linux
ansible_distribution: Ubuntu
ansible_distribution_version: 22.04
ansible_processor_cores: 8
ansible_memtotal_mb: 16000
ansible_default_ipv4.address: 192.168.1.100
ansible_user_id: root
ansible_hostname: server1
ansible_eth0.ipv4.address: 10.0.0.5
# ... many more

# These facts available in playbook:
- name: Install package (Ubuntu)
  apt:
    name: nginx
  when: ansible_os_family == "Debian"

- name: Install package (RedHat)
  yum:
    name: nginx
  when: ansible_os_family == "RedHat"
```

### Register and Variables

```yaml
# Capture command output into variable
- name: Check nginx status
  command: systemctl status nginx
  register: nginx_status    # Capture output

- name: Print result
  debug: msg="{{ nginx_status.stdout }}"
  # Output: nginx is running

# Example output captured:
nginx_status:
  stdout: "● nginx.service - A free, open-source..."
  stderr: ""
  rc: 0
  changed: false
  
# Use in conditions:
- name: Restart nginx if not running
  service:
    name: nginx
    state: restarted
  when: nginx_status.rc != 0  # If status failed

# Register with loop:
- name: Check multiple services
  command: systemctl status "{{ item }}"
  register: service_status
  loop:
    - nginx
    - mysql
    - postgresql

# service_status.results is list:
service_status.results[0].stdout  # nginx status
service_status.results[1].stdout  # mysql status
```

---

## 3. When to Use Ansible vs Terraform

### Fundamental Difference

| Aspect | Ansible | Terraform |
|--------|---------|-----------|
| **Focus** | Configuration mgmt | Infrastructure provisioning |
| **What it does** | Apply configs to existing servers | Create/modify cloud resources |
| **Agents** | Agentless (SSH) | Agentless (API calls) |
| **State** | Loose (playbook history) | Strict (state file) |
| **Idempotency** | Built-in | Via state tracking |
| **Learning curve** | Easy (YAML) | Moderate (HCL) |

### Use Ansible When:

✅ Configuring already-running servers (package install, service config)
✅ Orchestrating application deployment
✅ Managing configuration files and templates
✅ Running commands/scripts across many servers
✅ Simple, sequential task execution on servers

```yaml
# Perfect for Ansible:
- name: Deploy application
  hosts: webservers
  tasks:
    - name: Git clone application
      git:
        repo: https://github.com/myapp/repo.git
        dest: /opt/myapp
        version: v1.2.3
    
    - name: Install dependencies
      pip:
        requirements: /opt/myapp/requirements.txt
    
    - name: Start application
      systemd:
        name: myapp
        state: started
        enabled: yes
```

### Use Terraform When:

✅ Creating cloud infrastructure (VMs, networks, storage)
✅ Managing infrastructure as code with state tracking
✅ Multi-cloud deployments
✅ Complex resource dependencies
✅ Infrastructure versioning and rollback

```hcl
# Perfect for Terraform
resource "aws_instance" "webserver" {
  count           = 3
  ami             = data.aws_ami.ubuntu.id
  instance_type   = "t3.medium"
  security_groups = [aws_security_group.web.id]
  
  tags = {
    Name = "web-${count.index + 1}"
  }
}

resource "aws_lb" "web" {
  name = "web-lb"
  # ... configuration
}
```

### Combined Workflow (Terraform + Ansible)

```
1. Terraform provisions infrastructure
   $ terraform apply
   └─ Creates 3 EC2 instances in AWS
   └─ Creates security groups, load balancer
   └─ Outputs: instance IPs

2. Generate Ansible inventory from Terraform
   $ terraform output instances > inventory.ini
   
3. Ansible configures instances
   $ ansible-playbook -i inventory.ini playbook.yml
   └─ Install nginx on each instance
   └─ Configure application
   └─ Start services

Result: Infrastructure + Configuration in code!
```

### Decision Matrix

```
Question 1: Need to create cloud resources?
├─ YES → Use Terraform first
└─ NO → Use Ansible

Question 2: Already have existing servers?
├─ YES → Use Ansible to configure
└─ NO → Use Terraform to create, then Ansible

Question 3: Need strict state tracking?
├─ YES → Terraform
└─ NO → Ansible sufficient

Question 4: Simple one-time task automation?
├─ YES → Ansible
└─ NO → Might need both

Result: Terraform for infrastructure, Ansible for configuration
```

---

## 4. Real-World Usage

### Scenario 1: Deploy Java Application Stack

```bash
# Hosts to configure
inventory:
  [app_servers]
  app1.example.com
  app2.example.com
  
  [databases]
  db.example.com
  
  [all:vars]
  app_version: 1.2.3

# Playbook: deploy.yml
---
- name: Configure database
  hosts: databases
  roles:
    - postgresql
    - backup
  
- name: Deploy application
  hosts: app_servers
  roles:
    - java-runtime
    - application
    - monitoring
  handlers:
    - name: Restart application
      systemd:
        name: myapp
        state: restarted

# Usage:
$ ansible-playbook -i inventory/prod.ini deploy.yml

# Result:
PLAY [Configure database]
TASK [postgresql : Install PostgreSQL] changed
TASK [postgresql : Start PostgreSQL] ok
TASK [backup : Configure backups] changed

PLAY [Deploy application]
TASK [java-runtime : Install OpenJDK] ok
TASK [application : Copy application JAR] changed
TASK [application : Update config] changed
RUNNING HANDLER [Restart application]
HANDLER [Restart application] changed

PLAY RECAP
app1: ok=12 changed=5
app2: ok=12 changed=4
db: ok=8 changed=2
```

### Scenario 2: Rolling Update (0-Downtime Deployment)

```yaml
# Deploy new version with rolling updates
---
- name: Deploy with rolling updates
  hosts: webservers
  serial: 1      # One server at a time
  tasks:
    - name: Remove from load balancer
      shell: |
        curl -X PUT http://lb.internal:8080/api/nodes/{{ inventory_hostname }}/disable
      delegate_to: localhost
    
    - name: Wait for connections to drain
      pause:
        seconds: 30
    
    - name: Stop application
      systemd:
        name: myapp
        state: stopped
    
    - name: Deploy new version
      git:
        repo: https://github.com/myapp/repo.git
        dest: /opt/myapp
        version: v{{ app_version }}
    
    - name: Start application
      systemd:
        name: myapp
        state: started
    
    - name: Health check
      uri:
        url: "http://{{ inventory_hostname }}:8080/health"
        status_code: 200
      retries: 5
      delay: 10
    
    - name: Add back to load balancer
      shell: |
        curl -X PUT http://lb.internal:8080/api/nodes/{{ inventory_hostname }}/enable
      delegate_to: localhost

# Result (with serial: 1):
# web1: Update → Restore
# web2: Update → Restore
# web3: Update → Restore
# Total downtime: 0 (load balancer balances traffic)
```

### Scenario 3: Database Migration

```yaml
---
- name: Backup database
  hosts: db_primary
  tasks:
    - name: Create backup directory
      file:
        path: /backups
        state: directory
    
    - name: Backup production database
      shell: |
        pg_dump -U postgres mydb | gzip > /backups/mydb-{{ ansible_date_time.iso8601 }}.sql.gz
      register: backup_result
    
    - name: Display backup size
      debug:
        msg: "Backup size: {{ backup_result.stdout }}"

- name: Migrate to new database
  hosts: app_servers
  tasks:
    - name: Copy database dump
      copy:
        src: "{{ hostvars['db_primary'].backup_result.stdout }}"
        dest: /tmp/mydb.sql.gz
      
    - name: Import into new database
      shell: |
        gunzip -c /tmp/mydb.sql.gz | psql -U postgres -h db_new.example.com mydb
    
    - name: Update application connection string
      template:
        src: database.conf.j2
        dest: /opt/myapp/config/database.conf
    
    - name: Restart application
      systemd:
        name: myapp
        state: restarted

# Variables:
hostvars['db_primary'].backup_result  # Access facts from different host
```

---

## 5. Common Mistakes

### Mistake 1: Non-Idempotent Tasks

```yaml
# ❌ WRONG (non-idempotent)
- name: Run migration script
  command: /opt/myapp/migrate.sh
  # Running twice = runs migration twice!
  # If migration modifies data, second run corrupts it!

# ✅ CORRECT (idempotent)
- name: Check if migration needed
  stat:
    path: /opt/myapp/.migration_v1.2.3_done
  register: migration_done

- name: Run migration
  command: /opt/myapp/migrate.sh
  when: not migration_done.stat.exists

- name: Mark migration complete
  file:
    path: /opt/myapp/.migration_v1.2.3_done
    state: touch
  when: not migration_done.stat.exists
```

### Mistake 2: not Using `changed_when`

```yaml
# ❌ WRONG
- name: Check if service running
  command: systemctl is-active nginx
  # Ansible marks as "changed" even though checking!
  # Reports changed=yes when service is running
  # Looks like task made changes

# ✅ CORRECT
- name: Check if service running
  command: systemctl is-active nginx
  register: service_status
  changed_when: false   # Explicitly: this doesn't change system
  # Reports ok (no changes)

# Or for conditional:
- name: Run script
  script: /tmp/script.sh
  register: result
  changed_when: result.stdout | length > 0  # Only if script output
```

### Mistake 3: Hardcoded Values (No Flexibility)

```yaml
# ❌ WRONG
- name: Install package
  apt:
    name: nginx-1.18.0
    state: present

# Hardcoded version, can't change without editing playbook
# Not reusable across environments

# ✅ CORRECT
- name: Install package
  apt:
    name: "nginx-{{ nginx_version }}"
    state: present

# Can pass version at runtime:
$ ansible-playbook playbook.yml -e nginx_version=1.20.0

# Or in inventory:
[webservers]
web1.example.com nginx_version=1.20.0
web2.example.com nginx_version=1.19.0
```

### Mistake 4: Using shell/command Without Checking

```yaml
# ❌ WRONG
- name: Download file
  shell: wget https://example.com/app.tar.gz
  # Runs every time! Downloads every time!
  # Not idempotent

# ✅ CORRECT (check first, then download)
- name: Check if app exists
  stat:
    path: /opt/app.tar.gz
  register: app_exists

- name: Download app
  shell: wget https://example.com/app.tar.gz -O /opt/app.tar.gz
  when: not app_exists.stat.exists
```

### Mistake 5: Not Handling Errors Gracefully

```yaml
# ❌ WRONG
- name: Restart service
  service:
    name: nginx
    state: restarted
  # If service doesn't exist → playbook fails
  # Stops execution

# ✅ CORRECT
- name: Restart service
  service:
    name: nginx
    state: restarted
  ignore_errors: yes   # Continue even if fails
  # OR
  failed_when: false

# Or check first:
- name: Check if service exists
  stat:
    path: /etc/systemd/system/nginx.service
  register: service_exists

- name: Restart service (if exists)
  service:
    name: nginx
    state: restarted
  when: service_exists.stat.exists
```

---

## 6. Debugging Playbooks

### Scenario 1: Task Failed, Need to Investigate

```bash
# Run playbook with verbose output
$ ansible-playbook playbook.yml -vvv

# Output shows:
TASK [Install nginx]
<web1> EXEC /bin/sh -c 'apt-get install -y nginx'
Stderr: Unable to locate package nginx
Failed!

# Debug steps:
1. SSH to target host
$ ssh web1.example.com

2. Try command manually
$ apt-get update    # Update package cache!
$ apt-get install -y nginx

# Fix: Add update_cache
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes   # ← Add this!

3. Re-run playbook
$ ansible-playbook playbook.yml
# Success!
```

### Scenario 2: Variable Not Resolving

```yaml
# Problem: Template shows {{ variable_name }} literally
- name: Copy config
  template:
    src: config.j2
    dest: /etc/app/config.conf
  # Variable not substituting in template!

# Debug: Check if variable exists
- name: Debug variable
  debug:
    msg: "Database host is {{ db_host }}"
  # Output: Database host is {{ db_host }}
  # Variable not defined!

# Debug steps:
1. Check variable sources (in order):
   ├─ playbook vars section
   ├─ role defaults/main.yml
   ├─ role vars/main.yml
   ├─ Host variables (from inventory)
   ├─ Group variables (from inventory)

2. List all variables
$ ansible-playbook playbook.yml -e @vars.yml \
  -v | grep "db_host"

# Check inventory has variable:
[databases]
db1.example.com db_host=db1.prod.internal
# ← Must define here!

3. Verify in playbook:
- name: Show all vars
  debug:
    var: hostvars
  # Shows all variables for current host
```

### Scenario 3: Idempotency Broken (Task Always Reports changed)

```bash
# Problem: Running playbook twice shows changed=yes both times
$ ansible-playbook playbook.yml
TASK [Some task] changed

$ ansible-playbook playbook.yml
TASK [Some task] changed  # Should be ok=1!

# Debug: Check task for side effects
# Look for:
- shell: command    # May not be idempotent
- command: cmd      # May not be idempotent
- copy: file        # Might overwrite every time

# Example:
- name: Create marker file
  shell: touch /tmp/marker_{{ ansible_date_time.iso8601_basic }}
  # Creates new file each time! Not idempotent

# Fix: Make task idempotent
- name: Create marker file
  file:
    path: /tmp/marker
    state: touch
  # Same marker name each time
  
# Or explicitly mark:
- name: Some task
  shell: some_command
  changed_when: result.stdout != ""
  # Only report changed if output
```

### Scenario 4: Fact Not Available

```yaml
# Problem: ansible_eth0 fact doesn't exist
- name: Show interface
  debug:
    msg: "IP is {{ ansible_eth0.ipv4.address }}"
  # Error: undefined variable

# Debug steps:
1. Check available facts
$ ansible all -m setup | grep -i eth

# Output shows:
"ansible_eth0": not found
"ansible_ens3": {        # Different interface name!
    "ipv4": {...}
}

# Fix: Use dynamic variable lookup
- name: Show interface
  debug:
    msg: "IP is {{ hostvars[inventory_hostname].ansible_default_ipv4.address }}"

# Or use first interface:
- debug:
    msg: "IP is {{ ansible_interfaces[0] }}"
```

---

## 7. Interview Questions (15+ Scenario-Based)

### Level 1: Fundamentals

**Q1**: What's the difference between `command` and `shell` modules?

**Answer**:
- **command**: Executes command directly (no shell processing)
  ```yaml
  - command: echo "Hello" > /tmp/file.txt
    # Error! > is not processed
  ```

- **shell**: Runs through /bin/bash (processes redirects, pipes)
  ```yaml
  - shell: echo "Hello" > /tmp/file.txt
    # Works! > is processed by bash
  ```

Best practice:
```yaml
# Prefer `command` (more predictable)
- command: /usr/local/bin/apt-get install -y nginx

# Use `shell` only when needed:
- shell: find /tmp -name "*.log" | wc -l
# ← Pipe requires shell
```

---

**Q2**: Explain idempotency with a concrete example.

**Answer**:
```yaml
# Running 3 times should produce same result (idempotent)

- name: Install nginx
  apt:
    name: nginx
    state: present

# First run:
$ ansible-playbook playbook.yml
TASK [Install nginx]
changed: [web1]   # ← nginx installed

# Second run (idempotent):
$ ansible-playbook playbook.yml
TASK [Install nginx]
ok: [web1]        # ← already exists, no change!

# Third run (idempotent):
$ ansible-playbook playbook.yml
TASK [Install nginx]
ok: [web1]        # ← still no change

# Why idempotent?
# apt module checks: dpkg -l | grep nginx
# If installed → reports changed=false
# If not installed → installs and reports changed=true
# Running multiple times safe!
```

---

### Level 2: Advanced Usage

**Q3**: Design a playbook to deploy a new application version with zero downtime.

**Answer**:
```yaml
---
- name: Zero-downtime deployment
  hosts: webservers
  serial: 1              # One host at a time
  max_fail_percentage: 0 # No failures allowed
  
  pre_tasks:
    - name: Remove from load balancer
      shell: |
        curl -X POST http://lb.internal/remove \
          -d "host={{ inventory_hostname }}"
      delegate_to: localhost
    
    - name: Wait for active requests to complete
      pause:
        seconds: 30

  tasks:
    - name: Stop application
      systemd:
        name: myapp
        state: stopped
    
    - name: Deploy new version
      git:
        repo: https://github.com/myapp/repo.git
        dest: /opt/myapp
        version: v{{ new_version }}
    
    - name: Start application
      systemd:
        name: myapp
        state: started
    
    - name: Health check
      uri:
        url: "http://{{ inventory_hostname }}:8080/health"
        status_code: 200
      retries: 10
      delay: 5
      failed_when: false

  post_tasks:
    - name: Add back to load balancer
      shell: |
        curl -X POST http://lb.internal/add \
          -d "host={{ inventory_hostname }}"
      delegate_to: localhost

# Result: Each server updated one at a time
# - Load balancer routes traffic to others
# - Users never see downtime
# - If one host fails, deployment stops (max_fail_percentage: 0)
```

---

## 8. Advanced Insights

### Pattern 1: Dynamic Inventory Scaling

```python
# inventory.py (custom script)
#!/usr/bin/env python3
import json
import boto3

ec2 = boto3.client('ec2', region_name='us-east-1')
response = ec2.describe_instances(Filters=[
    {'Name': 'tag:Environment', 'Values': ['production']},
    {'Name': 'instance-state-name', 'Values': ['running']}
])

inventory = {'webservers': {'hosts': []}}
for reservation in response['Reservations']:
    for instance in reservation['Instances']:
        for tag in instance['Tags']:
            if tag['Key'] == 'Role' and tag['Value'] == 'webserver':
                inventory['webservers']['hosts'].append(instance['PrivateIpAddress'])

print(json.dumps(inventory))

# Usage:
$ ansible-playbook -i inventory.py playbook.yml
# Automatically discovers all production webserver instances!
```

### Pattern 2: Handler Ordering

```yaml
---
- name: Deployment with handlers
  hosts: webservers
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
      notify: restart nginx      # Queue handler
    
    - name: Update config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: reload nginx       # Queue handler
    
    - name: Copy SSL certificate
      copy:
        src: /certs/site.crt
        dest: /etc/nginx/ssl/
      notify: reload nginx       # Queue handler (already queued, runs once!)
  
  handlers:
    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
      listen: reload nginx
    
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
      listen: restart nginx

# Handler execution:
# - All tasks execute first
# - Then handlers run (in defined order)
# - Duplicates consolidated (reload nginx runs once, not 3x!)
# - restart nginx called → runs
# - reload nginx called → runs after restart
```

### Pattern 3: Conditional Execution & Block Error Handling

```yaml
---
- name: Advanced error handling
  hosts: all
  tasks:
    - block:  # Group tasks together
        - name: Database operations
          block:
            - name: Backup database
              shell: pg_dump mydb > /tmp/backup.sql
            
            - name: Migrate schema
              command: app migrate
            
            - name: Verify migration
              command: app verify
          
          rescue:  # Run if inner block fails
            - name: Restore from backup
              shell: psql mydb < /tmp/backup.sql
            
            - name: Notify team
              mail:
                host: smtp.example.com
                to: ops@example.com
                subject: "Migration failed"
          
          always:  # Run regardless (success/failure)
            - name: Clean up temporary files
              file:
                path: /tmp/backup.sql
                state: absent
      
      rescue:  # Run if outer block fails
        - name: Log failure
          shell: echo "Deployment failed" >> /var/log/deployment.log
      
      always:  # Run always
        - name: Send final status
          debug:
            msg: "Deployment complete"

# Execution flow:
# - Try: Run Backup → Migrate → Verify
# - If any fails:
#   - Rescue: Restore from backup + notify
# - Always:
#   - Clean up temp files + send status
# - Similar to try/catch/finally in programming!
```

---

## Conclusion

Ansible fundamentals rest on:

1. **Agentless**: SSH-based (no software on target)
2. **Idempotent**: Safe to run repeatedly
3. **Task-based**: Sequential execution of modules
4. **Inventory-driven**: Group servers, apply plays to groups

Master these and you can:
- ✅ Configure thousands of servers consistently
- ✅ Deploy applications with zero downtime
- ✅ Orchestrate complex infrastructure changes
- ✅ Build reusable, maintainable automation

**Key principle**: Ansible is configuration management for infrastructure that already exists. Use Terraform to create infrastructure, then Ansible to configure it!
