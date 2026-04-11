# Ansible Playbooks: Writing, Structuring, and Best Practices

## 1. Core Concept

Playbooks are **YAML files that define desired state** for infrastructure:

1. **Plays**: List of plays (each targets a host group)
2. **Tasks**: Atomic actions (install package, copy file, restart service)
3. **Handlers**: Conditional tasks (triggered by notifications)
4. **Variables**: Dynamic values (host-specific, group-specific, playbook-level)
5. **Control Flow**: Conditionals, loops, error handling

### Simple Playbook Structure

```yaml
---
# playbook.yml
- name: Configure webservers        # Play name
  hosts: webservers                 # Target host group
  gather_facts: yes                 # Collect system facts
  vars:                             # Playbook-level variables
    nginx_version: 1.20.0
    
  pre_tasks:                        # Tasks before main tasks
    - name: Update package cache
      apt: update_cache=yes
  
  tasks:                            # Main tasks
    - name: Install nginx
      apt:
        name: "nginx={{ nginx_version }}"
        state: present
    
    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes
  
  post_tasks:                       # Tasks after main tasks
    - name: Verify nginx running
      uri:
        url: http://localhost
        status_code: 200
  
  handlers:                         # Conditional tasks
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

### Playbook Execution

```bash
# Run playbook
$ ansible-playbook playbook.yml

# Output:
PLAY [Configure webservers]

TASK [Gathering Facts]
ok: [web1]
ok: [web2]

TASK [Update package cache]
changed: [web1]
changed: [web2]

TASK [Install nginx]
changed: [web1]
changed: [web2]

TASK [Start nginx]
ok: [web1]
ok: [web2]

TASK [Verify nginx running]
ok: [web1]
ok: [web2]

PLAY RECAP
web1: ok=5 changed=2 unreachable=0 failed=0
web2: ok=5 changed=2 unreachable=0 failed=0
```

---

## 2. How Ansible Playbooks Work Internally

### Parsing and Execution Pipeline

```
Step 1: Parse YAML
├─ Read playbook.yml
├─ Parse YAML syntax
└─ Extract plays

Step 2: Load context
├─ Read inventory
├─ Load variables (defaults, group_vars, host_vars)
├─ Match play hosts to inventory
└─ Identify target hosts

Step 3: For each play
├─ resolve_hosts: Get list of target hosts
├─ gather_facts: Run fact gathering (if enabled)
├─ execute_tasks:
│  ├─ For each task (in order)
│  │  ├─ Evaluate conditionals (when, changed_when)
│  │  ├─ Expand variables {{ var }}
│  │  ├─ For each target host (default: parallel)
│  │  │  ├─ Format module + parameters
│  │  │  ├─ SSH to host
│  │  │  ├─ Execute module
│  │  │  ├─ Capture result
│  │  │  ├─ Evaluate result (changed, failed, rc)
│  │  │  └─ Register variables (if registered)
│  │  └─ Process notifications (handlers)
│  └─ Execute handlers (if notified)
└─ Output results
```

### Variable Resolution (Precedence)

```
Ansible resolves {{ variable }} from (in order):
1. Command line (-e, --extra-vars)    ← Highest priority
2. Current play vars section
3. Role defaults (defaults/main.yml)
4. Role vars (vars/main.yml)
5. Host facts (from gather_facts)
6. Host variables (group_vars/all, group_vars/GROUP, host_vars/HOST)
7. Ansible defaults                   ← Lowest priority

Example:
inventory/group_vars/webservers.yml:
  nginx_version: 1.18.0

playbook.yml:
  vars:
    nginx_version: 1.20.0     ← This wins

$ ansible-playbook -e nginx_version=1.22.0  ← This wins over all!
```

### Task Parallelization

```yaml
# Default: Ansible runs tasks parallel across hosts
tasks:
  - name: Install package     # Execute on all hosts in parallel
    apt:
      name: nginx
      state: present
  # Once ALL hosts complete this task, move to next task

---
- name: Configure servers
  hosts: webservers
  
  # Control parallelization:
  serial: 1           # One host at a time (rolling deployment)
  forks: 10           # Up to 10 hosts in parallel (default: 5)
  
  tasks:
    - name: Task 1
      ...
    - name: Task 2
      ...

# With serial: 1
# web1: Task1 → Task2 (complete)
# web2: Task1 → Task2 (complete)
# web3: Task1 → Task2 (complete)

# With default (forks: 5)
# web1,web2,web3,web4,web5: Task1 (parallel)
# web6,web7,web8,web9,web10: Task1 (2nd batch)
# web1,web2,web3,web4,web5: Task2 (parallel)
```

### Conditional Task Execution

```yaml
# when: condition (boolean expression)
tasks:
  - name: Install nginx
    apt:
      name: nginx
      state: present
    when: ansible_os_family == "Debian"  # Only on Debian/Ubuntu

  - name: Install nginx (RedHat)
    yum:
      name: nginx
      state: present
    when: ansible_os_family == "RedHat"  # Only on RedHat/CentOS

  - name: Debug if variable
    debug:
      msg: "Variable is {{ var }}"
    when: var is defined  # Check if variable defined

  - name: Enable feature
    shell: feature enable
    when:
      - environment == "production"        # AND
      - enable_feature is defined          # AND
      - enable_feature | bool              # Convert to boolean

# Conditional functions:
is defined           # Variable exists
is undefined         # Variable doesn't exist
is true              # Boolean true
is false             # Boolean false
is changed           # Task reported changed
is failed            # Task failed
is succeeded         # Task succeeded
is skipped           # Task was skipped
in                   # Check if in list
```

### Loops (Iteration)

```yaml
# Simple loop (over list)
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - postgresql
    - redis

# Loop over dictionary
- name: Create users
  user:
    name: "{{ item.name }}"
    uid: "{{ item.uid }}"
  loop:
    - { name: 'alice', uid: 1001 }
    - { name: 'bob', uid: 1002 }
    - { name: 'charlie', uid: 1003 }

# Loop with condition
- name: Install packages (only large packages)
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - postgresql
    - redis
  when: item != 'redis'  # Skip redis

# Loop with index
- name: Print index
  debug:
    msg: "Item {{ ansible_loop.index0 }}: {{ item }}"
  loop:
    - first
    - second
    - third
  # Output:
  # Item 0: first
  # Item 1: second
  # Item 2: third

# Nested loop
- name: Create users in groups
  debug:
    msg: "User {{ user }} in group {{ group }}"
  loop: "{{ groups }}"
  vars:
    groups:
      - { group: 'admins', users: ['alice', 'bob'] }
      - { group: 'users', users: ['charlie', 'dave'] }
```

---

## 3. When to Use Ansible vs Terraform (Playbook-Specific)

### Playbook Use Cases (Ansible Strength)

✅ **Configuration management**
```yaml
- name: Configure servers
  tasks:
    - name: Update config files
    - name: Restart services
    - name: Verify functionality
```

✅ **Application deployment**
```yaml
- name: Deploy application
  tasks:
    - name: Git clone
    - name: Install dependencies
    - name: Start service
```

✅ **Multi-step orchestration**
```yaml
- name: Blue-green deployment
  tasks:
    - name: Drain connections
    - name: Deploy new version
    - name: Smoke tests
    - name: Switch traffic
```

### Terraform Use Cases (Avoid Playbooks For)

❌ Playbooks should NOT:
```hcl
# Don't use Ansible to create cloud resources
# Use Terraform instead:

resource "aws_instance" "web" {
  count           = 3
  ami             = "ami-12345"
  instance_type   = "t3.medium"
}

# Then use Ansible to configure:
# Generate inventory from Terraform output
# Run playbook to install/configure
```

### Combined Workflow (Recommended)

```
Terraform:
├─ Create EC2 instances
├─ Create security groups
├─ Create load balancer
└─ Output: Instance IPs in inventory.ini

Ansible:
├─ Read inventory from Terraform
├─ Install packages on instances
├─ Configure applications
└─ Start services

Result: Infrastructure + Configuration managed separately
```

---

## 4. Real-World Usage

### Scenario 1: Multi-Play Deployment

```yaml
---
# deploy.yml: Deploy entire application stack
- name: Update package cache
  hosts: all
  tasks:
    - name: apt update
      apt:
        update_cache: yes

- name: Configure databases
  hosts: databases
  pre_tasks:
    - name: Backup database
      shell: pg_dump -Fc mydb > /backups/mydb.dump
  
  roles:
    - postgresql
  
  post_tasks:
    - name: Verify database
      shell: psql -c "SELECT 1"

- name: Deploy application
  hosts: app_servers
  serial: 1  # One server at a time
  
  tasks:
    - name: Git clone
      git:
        repo: https://github.com/myapp/repo.git
        dest: /opt/myapp
        version: "{{ app_version }}"
    
    - name: Install dependencies
      pip:
        requirements: /opt/myapp/requirements.txt
        virtualenv: /opt/myapp/venv
    
    - name: Run migrations
      command: /opt/myapp/venv/bin/python manage.py migrate
    
    - name: Start application
      systemd:
        name: myapp
        state: restarted
        daemon_reload: yes
    
    - name: Health check
      uri:
        url: http://localhost:8000/health
        status_code: 200
      retries: 5
      delay: 10

- name: Configure reverse proxy
  hosts: loadbalancers
  
  tasks:
    - name: Update upstream servers
      template:
        src: nginx_upstream.j2
        dest: /etc/nginx/conf.d/upstream.conf
      notify: reload nginx
  
  handlers:
    - name: reload nginx
      service:
        name: nginx
        state: reloaded

# Execution order:
# 1. all: Update cache
# 2. databases: Backup + PostgreSQL role
# 3. app_servers: Deploy (one by one)
# 4. loadbalancers: Update config
# Total: Coordinated, sequential deployment
```

### Scenario 2: Variable Overrides

```bash
# inventory/prod.yml
[webservers]
web[1:3].prod.example.com

[webservers:vars]
nginx_version: 1.20.0
app_version: 1.2.3

# playbook.yml
---
- name: Deploy
  hosts: webservers
  vars:
    environment: production  # Playbook-level variable
  
  tasks:
    - name: Install nginx
      apt:
        name: "nginx={{ nginx_version }}"  # From inventory
        state: present
    
    - name: Deploy app
      git:
        repo: https://github.com/app
        version: "{{ app_version }}"  # From inventory

# Run with CLI overrides
$ ansible-playbook playbook.yml \
  -i inventory/prod.yml \
  -e nginx_version=1.22.0 \
  -e environment=staging

# Precedence:
# CLI (-e) wins over inventory vars
# Final: nginx_version=1.22.0 (from CLI)
```

### Scenario 3: Conditional Deployment Paths

```yaml
---
- name: Deployment with feature flags
  hosts: all
  
  vars:
    deploy_type: "{{ deploy_type | default('standard') }}"
    enable_monitoring: yes
    enable_caching: "{{ enable_caching | default(false) }}"
  
  tasks:
    # Standard deployment (always)
    - name: Standard tasks
      include_tasks: tasks/standard.yml
    
    # Conditional: Enable monitoring
    - name: Setup monitoring
      include_tasks: tasks/monitoring.yml
      when: enable_monitoring
    
    # Conditional: Enable caching
    - name: Setup cache layer
      include_tasks: tasks/caching.yml
      when: enable_caching | bool
    
    # Conditional: Blue-green deployment
    - name: Blue-green deployment
      include_tasks: tasks/blue_green.yml
      when: deploy_type == "blue_green"
    
    # Conditional: Canary deployment
    - name: Canary deployment
      include_tasks: tasks/canary.yml
      when: deploy_type == "canary"

# Run with different options:
$ ansible-playbook deploy.yml  # Standard
$ ansible-playbook deploy.yml -e enable_monitoring=yes enable_caching=yes
$ ansible-playbook deploy.yml -e deploy_type=blue_green
$ ansible-playbook deploy.yml -e deploy_type=canary
```

---

## 5. Common Mistakes

### Mistake 1: Not Using `register` + `when` for Idempotency

```yaml
# ❌ WRONG (runs every time)
- name: Run migration
  command: python manage.py migrate

# ✅ CORRECT
- name: Check if migrated
  stat:
    path: /var/lib/app/.migrated
  register: migrated

- name: Run migration
  command: python manage.py migrate
  when: not migrated.stat.exists

- name: Mark as migrated
  file:
    path: /var/lib/app/.migrated
    state: touch
  when: not migrated.stat.exists
```

### Mistake 2: Using `shell` When `command` Sufficient

```yaml
# ❌ WRONG (shell has more overhead)
- shell: /usr/bin/apt-get install -y nginx

# ✅ CORRECT
- command: /usr/bin/apt-get install -y nginx

# Shell needed only for:
- shell: find /tmp -name "*.log" -delete  # Piping/redirection
- shell: echo "{{ var }}" > /tmp/file     # Output redirection
```

### Mistake 3: Forgetting `loop_control`

```yaml
# ❌ WRONG (unclear output)
- name: Create users
  user:
    name: "{{ item }}"
  loop:
    - alice
    - bob
    - charlie
  # Output shows "item" not actual value

# ✅ CORRECT
- name: Create users
  user:
    name: "{{ item }}"
  loop:
    - alice
    - bob
    - charlie
  loop_control:
    label: "{{ item }}"  # Shows actual item name
```

### Mistake 4: Not Handling Errors in Loops

```yaml
# ❌ WRONG (first failure stops loop)
- name: Restart services
  service:
    name: "{{ item }}"
    state: restarted
  loop:
    - nginx
    - mysql
    - postgresql
  # If mysql fails, postgresql not restarted!

# ✅ CORRECT
- name: Restart services
  service:
    name: "{{ item }}"
    state: restarted
  loop:
    - nginx
    - mysql
    - postgresql
  ignore_errors: yes  # Continue on failure
  register: restart_results

- name: Report failures
  debug:
    msg: "{{ restart_results.results | selectattr('failed', 'equalto', true) }}"
  when: restart_results is failed
```

### Mistake 5: Variables in Loops Not Expanding

```yaml
# ❌ WRONG
- name: Install packages
  apt:
    name: "{{ item }}"
  vars:
    packages:
      - nginx
      - postgresql
  loop:
    - "{{ packages }}"  # Expands to: packages (string!)

# ✅ CORRECT
- name: Install packages
  apt:
    name: "{{ item }}"
  loop: "{{ packages }}"  # Loop over expanded list
  vars:
    packages:
      - nginx
      - postgresql
```

---

## 6. Debugging Playbooks

### Scenario 1: Task Failing, Need Details

```bash
# Run with verbose output
$ ansible-playbook playbook.yml -v    # -v: 1 level verbose
$ ansible-playbook playbook.yml -vv   # -vv: 2 levels
$ ansible-playbook playbook.yml -vvv  # -vvv: 3 levels (most detailed)

# Output shows:
TASK [Install nginx]
<web1> EXEC /bin/sh -c 'apt-get install -y nginx'
Stderr: Unable to locate package nginx
rc: 100
Failed!

# Debug: Package cache not updated
# Fix: Add update_cache
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes
```

### Scenario 2: Variable Showing {{ }} Not Substituted

```yaml
# Problem: Template shows {{ var }} literally
- name: Install package
  apt:
    name: "nginx={{ nginx_version }}"

- name: Debug variable
  debug:
    msg: "Version is {{ nginx_version }}"
  # Output: Version is {{ nginx_version }}
  # Variable not defined!

# Debug steps:
1. Check inventory has variable:
$ grep nginx_version inventory.ini
[webservers:vars]
nginx_version=1.20.0

2. Check playbook vars:
$ grep -A 5 "vars:" playbook.yml
vars:
  nginx_version: 1.18.0

3. Print all variables:
- name: Show all
  debug:
    var: hostvars[inventory_hostname]

4. Check variable scope (inside vs outside role):
# Variables available in role must be set in:
# - role defaults/main.yml (lowest priority)
# - role vars/main.yml (higher priority)
# - Playbook vars (even higher)
# - CLI (-e flag) (highest)
```

### Scenario 3: Loop Not Iterating

```yaml
# Problem: Loop body not appearing
- name: Create users
  user:
    name: "{{ item }}"
  loop: "{{ users }}"
  # Output: Nothing! Loop didn't run

# Debug: Variable not defined
- name: Debug users
  debug:
    var: users
  # Output: users is undefined

# Fix: Define variable
---
- name: Create users
  hosts: all
  vars:
    users:
      - alice
      - bob
  tasks:
    - name: Create users
      user:
        name: "{{ item }}"
      loop: "{{ users }}"  # Now defined!
```

### Scenario 4: Handler Not Running

```yaml
# Problem: Handler not executing
- name: Update config
  template:
    src: config.j2
    dest: /etc/app/config.conf
  notify: restart app
  # Handler not running!

handlers:
  - name: restart app
    service:
      name: myapp
      state: restarted

# Debug:
1. Check notification name matches handler name exactly
   notify: restart app       # Must match exactly!
   name: restart app         # ← Must match

2. Check handler is in same play:
   handlers: MUST be in same play, not different file

3. Check if task actually changed system:
   handlers only run if task reports changed=yes
   
# Fix: Ensure task reports change:
- name: Update config
  template:
    src: config.j2
    dest: /etc/app/config.conf
  notify: restart app
  # If template changed file → reports changed=yes
  # → Handler runs
```

---

## 7. Interview Questions (15+ Scenario-Based)

### Level 1: Basic Playbooks

**Q1**: What's the difference between `pre_tasks`, `tasks`, and `post_tasks`?

**Answer**:
```yaml
---
- name: Deployment
  hosts: webservers
  
  pre_tasks:      # Run first (before fact gathering in some cases)
    - name: Notify start
      slack:
        msg: "Deployment starting"
  
  tasks:          # Main tasks
    - name: Deploy app
      git:
        repo: https://github.com/app
        dest: /opt/app
  
  post_tasks:     # Run last (after all tasks)
    - name: Smoke tests
      uri:
        url: http://localhost/health
        status_code: 200

# Execution order:
# 1. Gather facts
# 2. pre_tasks (notify)
# 3. tasks (deploy)
# 4. handlers (if notified)
# 5. post_tasks (smoke tests)
```

---

**Q2**: How would you implement a rolling deployment?

**Answer**:
```yaml
---
- name: Rolling deployment
  hosts: webservers
  serial: 2          # 2 servers at a time
  max_fail_percentage: 0  # No failures allowed
  
  tasks:
    - name: Drain connections
      shell: signal_drain.sh
    
    - name: Deploy
      git:
        repo: https://github.com/app
        version: v{{ version }}
        dest: /opt/app
    
    - name: Healt check
      uri:
        url: http://localhost/health
      retries: 5
      delay: 10

# With 5 servers and serial: 2
# Batch 1: [web1, web2] deploy
# Batch 2: [web3, web4] deploy
# Batch 3: [web5] deploy
# Total time: 1/3 of parallel deployment
# Better than all-at-once
```

---

### Level 2: Advanced Playbooks

**Q3**: Design a playbook for database schema migration with rollback capability.

**Answer**:
```yaml
---
- name: Database schema migration
  hosts: database
  vars:
    version: "{{ app_version | default('1.0.0') }}"
  
  pre_tasks:
    - name: Backup database
      shell: |
        pg_dump -Fc mydb > /backups/mydb_{{ version }}.dump
      register: backup

  tasks:
    - name: Check migration status
      stat:
        path: "/var/lib/db/.migrated_{{ version }}"
      register: migrated
    
    - name: Run migration
      shell: |
        python manage.py migrate --version {{ version }}
      register: migration_result
      when: not migrated.stat.exists
      ignore_errors: yes
    
    - name: Verify migration
      command: python manage.py migrate --check
      when: migration_result is succeeded
    
    - name: Mark migration complete
      file:
        path: "/var/lib/db/.migrated_{{ version }}"
        state: touch
      when: migration_result is succeeded
  
  post_tasks:
    - name: Rollback if failed
      shell: |
        pg_restore -d mydb /backups/mydb_{{ version }}.dump
      when: migration_result is failed

# Advantages:
- Automatic backup before migration
- Idempotent (marked after success)
- Auto-rollback on failure
- Version tracking
```

---

## 8. Advanced Insights

### Pattern 1: Playbook Composition

```yaml
# main.yml (orchestration playbook)
---
- import_playbook: infrastructure.yml      # Create resources
- import_playbook: configuration.yml        # Configure resources
- import_playbook: validation.yml           # Verify setup
- import_playbook: monitoring.yml           # Setup monitoring

# infrastructure.yml
---
- name: Setup infrastructure
  hosts: localhost
  tasks:
    - name: Provision servers (via Terraform)
      ...

# configuration.yml
---
- name: Configure servers
  hosts: all
  tasks:
    - name: Install packages
      ...

# Run everything:
$ ansible-playbook main.yml
# Executes all sub-playbooks in order!
```

### Pattern 2: Error Recovery

```yaml
---
- name: Deployment with recovery
  hosts: webservers
  
  tasks:
    - block:
        - name: Deploy
          git:
            repo: https://github.com/app
            dest: /opt/app
            version: "{{ version }}"
        
        - name: Health check
          uri:
            url: http://localhost/health
            status_code: 200
      
      rescue:
        - name: Log error
          copy:
            content: "Deployment failed at {{ now() }}"
            dest: /var/log/deployment_error.log
        
        - name: Revert to previous
          shell: git -C /opt/app checkout HEAD~1
        
        - name: Notify team
          mail:
            host: smtp.example.com
            to: ops@example.com
            subject: "Deployment failed, reverted"
      
      always:
        - name: Report status
          debug:
            msg: "Deployment {{ 'succeeded' if migration_result is succeeded else 'failed' }}"

# Execution:
# - Try: Deploy + health check
# - If fails, Rescue: Log + revert + notify
# - Always: Report status
# Similar to try/catch/finally!
```

### Pattern 3: Fact Caching

```yaml
---
- name: Setup with fact caching
  hosts: all
  
  # Enable fact caching (Redis backend)
  vars:
    ansible_facts_cache: /tmp/ansible_facts
    ansible_facts_cache_plugin: jsonfile
    ansible_facts_cache_timeout: 86400  # 24 hours
  
  tasks:
    - name: Gather facts (cached for future runs)
      setup:
    
    - name: Use cached facts
      debug:
        msg: "{{ ansible_os_family }}"  # From cache!

# Benefits:
- First run: Gather facts (slow)
- Subsequent runs: Use cached facts (fast!)
- Huge speedup for large inventories
```

---

## Conclusion

Playbooks are the **primary interface** for infrastructure automation:

1. **Sequential execution**: Task runs after task
2. **Parallel by default**: Across multiple hosts
3. **Error handling**: Blocks, rescue, always
4. **Flexible variables**: Overridable at multiple levels
5. **Handlers**: Triggered by notifications

Master playbook design and you can:
- ✅ Automate complex multi-step deployments
- ✅ Handle errors gracefully with recovery
- ✅ Scale to thousands of servers
- ✅ Maintain infrastructure as code

**Key principle**: Playbooks are declarative (what you want) but executed imperative (how to get there). Structure them for clarity, readability, and reusability!
