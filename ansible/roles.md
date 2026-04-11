# Ansible Roles: Modular, Reusable Infrastructure Code

## 1. Core Concept

Roles are **modular units of infrastructure code** that:

1. **Organize tasks**: Group related functionality together
2. **Enable reuse**: Same role across multiple projects
3. **Manage scope**: Variables, files, templates isolated
4. **Simplify playbooks**: Roles abstract complexity
5. **Enforce structure**: Standardized directory layout

### Role Structure

```
roles/
├── common/                  # Role name
│   ├── tasks/
│   │   ├── main.yml        # Entry point (required)
│   │   ├── install.yml     # Other tasks (included)
│   │   └── config.yml
│   │
│   ├── handlers/
│   │   └── main.yml        # Event handlers (optional)
│   │
│   ├── templates/
│   │   ├── config.conf.j2  # Jinja2 templates
│   │   └── app.conf.j2
│   │
│   ├── files/
│   │   └── app.jar         # Static files
│   │
│   ├── defaults/
│   │   └── main.yml        # Default variables (lowest priority)
│   │
│   ├── vars/
│   │   └── main.yml        # Role variables (higher priority)
│   │
│   ├── meta/
│   │   └── main.yml        # Role metadata, dependencies
│   │
│   └── README.md           # Role documentation
```

### Simple Role Example

```yaml
# roles/nginx/tasks/main.yml
---
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes

- name: Enable nginx
  service:
    name: nginx
    state: started
    enabled: yes

# roles/nginx/handlers/main.yml
---
- name: reload nginx
  service:
    name: nginx
    state: reloaded

# roles/nginx/defaults/main.yml
---
nginx_version: latest
nginx_port: 80

# Usage in playbook:
---
- name: Deploy nginx
  hosts: webservers
  roles:
    - nginx
  
# Playbook automatically:
# 1. Gathers facts
# 2. Sets default variables from defaults/main.yml
# 3. Runs tasks/main.yml
# 4. Registers handlers/main.yml for use
```

---

## 2. How Ansible Roles Work Internally

### Role Loading and Execution

```
When playbook references role:
- hosts: webservers
  roles:
    - nginx

Ansible:
1. Search for role in:
   ├─ ./roles/                    (current directory)
   ├─ ~/.ansible/roles/           (user home)
   ├─ /etc/ansible/roles/         (system)
   └─ Paths in ansible.cfg

2. Load role structure:
   └─ roles/nginx/
      ├─ tasks/main.yml          (mandatory)
      ├─ handlers/main.yml       (if exists)
      ├─ defaults/main.yml       (if exists)
      ├─ vars/main.yml           (if exists)
      ├─ templates/              (if exists)
      ├─ files/                  (if exists)
      └─ meta/main.yml           (if exists)

3. Set variable precedence:
   1. defaults/main.yml  (lowest)
   2. vars/main.yml      (higher)
   3. Playbook vars
   4. CLI vars (-e)      (highest)

4. Execute tasks/main.yml:
   - Register handlers from handlers/main.yml
   - Execute tasks in order
   - Handlers available for notifications

5. Result:
   └─ All role tasks executed on target host
```

### Variable Scope in Roles

```yaml
# roles/app/defaults/main.yml (lowest priority)
---
app_port: 8080
app_user: appuser
log_level: info

# roles/app/vars/main.yml (overrides defaults)
---
app_version: 1.2.3
app_user: appuser-prod  # Overrides default!

# playbook.yml (overrides defaults + vars)
---
- name: Deploy
  hosts: webservers
  vars:
    log_level: debug    # Overrides role default
  roles:
    - app

# CLI override (highest priority)
$ ansible-playbook -e app_port=9000 -e log_level=trace

# Final values:
app_port: 9000         ← From CLI (-e)
app_version: 1.2.3     ← From role vars
app_user: appuser-prod ← From role vars (overrides default)
log_level: trace       ← From CLI (overrides playbook var)
```

### Role Dependencies

```yaml
# roles/app/meta/main.yml
---
dependencies:
  - role: geerlingguy.java    # External role from Galaxy
    java_packages:
      - openjdk-11-jdk
  
  - role: common              # Local role
    tags: common
  
  - role: monitoring          # Optional dependency
    when: enable_monitoring is defined
    tags: monitoring

# When playbook includes app role:
---
- name: Deploy
  hosts: webservers
  roles:
    - app

# Ansible automatically:
# 1. Resolves dependencies
# 2. Downloads external roles (from Galaxy)
# 3. Executes dependencies first:
#    - geerlingguy.java (Java runtime)
#    - common (General setup)
#    - monitoring (if condition met)
# 4. Then executes: app role
```

### Task Inclusion in Roles

```yaml
# roles/database/tasks/main.yml (entry point)
---
- name: Install database
  include_tasks: install.yml

- name: Configure database
  include_tasks: config.yml

- name: Setup replication
  include_tasks: replication.yml
  when: replication_enabled

# roles/database/tasks/install.yml
---
- name: Install PostgreSQL packages
  apt:
    name: postgresql
    state: present

# roles/database/tasks/config.yml
---
- name: Configure PostgreSQL
  template:
    src: postgresql.conf.j2
    dest: /etc/postgresql/postgresql.conf
    owner: postgres
    group: postgres

# Result:
# Task 1: Install (from include_tasks)
# Task 2: Configure (from include_tasks)
# Task 3: Replication (conditional include_tasks)
# Organized, readable, modular!
```

---

## 3. When to Use Roles vs Playbooks

### Use Roles When:

✅ **Reusable functionality** (used across projects)
```yaml
# roles/nginx/          - Configure nginx (reusable)
# roles/postgresql/     - Setup postgres (reusable)
# roles/monitoring/     - Prometheus setup (reusable)
roles:
  - nginx
  - postgresql
  - monitoring
```

✅ **Complex functionality** (many tasks, handlers, templates)
```yaml
# roles/kubernetes/tasks/main.yml has 50+ tasks
# Better organized than one big playbook
roles:
  - kubernetes
```

✅ **Standardizing best practices** (enforce structure)
```yaml
# roles/security/
# └─ Always enable SELinux
# └─ Always configure firewall
# └─ Always audit everything
# Ensures consistency
```

### Use Playbooks When:

✅ **One-off tasks** (specific to one project)
```yaml
---
- name: Deploy version 1.2.3
  hosts: webservers
  tasks:
    - name: Update database schema
      shell: python manage.py migrate
# Specific to this app, not reusable
```

✅ **Simple orchestration** (few tasks, sequential)
```yaml
---
- name: Restart services
  hosts: all
  tasks:
    - service: name=nginx state=restarted
    - service: name=mysql state=restarted
# Simple, direct playbook
```

---

## 4. Real-World Usage

### Scenario 1: Reusable Role for Database

```
roles/postgresql/
├── tasks/
│   ├── main.yml       # Entry point
│   ├── install.yml    # Install packages
│   ├── config.yml     # Configure postgres
│   └── replication.yml
│
├── handlers/
│   └── main.yml       # Restart handlers
│
├── templates/
│   ├── postgresql.conf.j2
│   ├── pg_hba.conf.j2
│   └── recovery.conf.j2
│
├── files/
│   └── backup.sh
│
├── defaults/
│   └── main.yml
│      postgres_version: 13
│      postgres_port: 5432
│      postgres_data_dir: /var/lib/postgresql
│      backup_enabled: yes
│      replication_enabled: no
│
└── meta/
    └── main.yml

# Usage across projects:

# project1/playbook.yml
- name: Deploy project 1
  hosts: databases
  roles:
    - postgresql
  vars:
    postgres_version: 12
    replication_enabled: yes

# project2/playbook.yml
- name: Deploy project 2
  hosts: databases
  roles:
    - postgresql
  vars:
    postgres_version: 14
    backup_enabled: yes
    replication_enabled: no

# Same role, different configs!
# DRY principle: Write once, use everywhere
```

### Scenario 2: Role Dependencies and Stack Deployment

```
roles/
├── base/              # Base OS configuration
│   ├── tasks/main.yml
│   ├── handlers/main.yml
│   └── defaults/main.yml
│
├── docker/            # Docker runtime
│   ├── meta/main.yml
│   │  dependencies:
│   │    - role: base  # Docker requires base setup!
│   ├── tasks/main.yml
│   └── defaults/main.yml
│
├── kubernetes/        # Kubernetes cluster
│   ├── meta/main.yml
│   │  dependencies:
│   │    - role: docker  # K8s requires Docker
│   ├── tasks/main.yml
│   └── defaults/main.yml
│
└── monitoring/        # Prometheus + Grafana
    ├── meta/main.yml
    │  dependencies:
    │    - role: docker  # Monitoring uses containers
    ├── tasks/main.yml
    └── defaults/main.yml

# Usage in playbook:
---
- name: Deploy full stack
  hosts: servers
  roles:
    - monitoring    # Request monitoring role
  
# Ansible automatically resolves:
# Dependencies for monitoring:
#   → Requires docker
#     → Requires base
# Final execution order:
# 1. base (OS setup)
# 2. docker (container runtime)
# 3. monitoring (Prometheus)
# No manual ordering needed!
```

### Scenario 3: Conditional Role Application

```yaml
---
- name: Deploy with conditional roles
  hosts: all
  
  roles:
    # Always apply
    - role: base
      tags: always
    
    # Conditional: Only on database servers
    - role: postgresql
      when: inventory_hostname in groups['databases']
      tags: database
    
    # Conditional: Only in production
    - role: monitoring
      when: environment == "production"
      tags: monitoring
    
    # Conditional: Only if enabled
    - role: logging
      when: enable_logging is defined and enable_logging
      tags: logging
    
    # With role variables
    - role: nginx
      vars:
        nginx_version: 1.20.0
        enable_ssl: yes
      tags: webserver

# Usage:
$ ansible-playbook deploy.yml              # All roles
$ ansible-playbook deploy.yml -t database  # Only database role
$ ansible-playbook deploy.yml -t webserver # Only nginx role
$ ansible-playbook deploy.yml -t always    # Always applies even with tags
```

---

## 5. Common Mistakes

### Mistake 1: Mixing Defaults and Vars

```yaml
# ❌ WRONG
roles/myapp/defaults/main.yml
---
app_name: myapp
app_port: 8080
app_version: latest

roles/myapp/vars/main.yml
---
app_port: 8080  # Same variable in two places!
worker_threads: 4

# Confusing! Which takes precedence? Maintenance nightmare

# ✅ CORRECT
# defaults/main.yml: ALL defaults (can be overridden)
---
app_name: myapp
app_port: 8080
app_version: latest
worker_threads: 4

# vars/main.yml: Role requirements (should NOT be overridden)
---
app_user: "{{ app_name }}-user"      # Derived from app_name
app_group: "{{ app_name }}-group"    # Derived from app_name
app_home: "/opt/{{ app_name }}"      # Derived from app_name
```

### Mistake 2: Hard-coded Values in Tasks

```yaml
# ❌ WRONG
roles/app/tasks/main.yml
- name: Create app directory
  file:
    path: /opt/myapp        # Hard-coded!
    state: directory
- name: Copy app
  copy:
    src: app.jar
    dest: /opt/myapp/app.jar

# Edit playbook to change path

# ✅ CORRECT
roles/app/defaults/main.yml
---
app_home: /opt/myapp
app_user: appuser

roles/app/tasks/main.yml
- name: Create app directory
  file:
    path: "{{ app_home }}"  # Variable!
    state: directory
- name: Copy app
  copy:
    src: app.jar
    dest: "{{ app_home }}/app.jar"

# Override in playbook:
roles:
  - role: app
    vars:
      app_home: /applications/myapp  # Different path!
```

### Mistake 3: Not Using Tags for Flexibility

```yaml
# ❌ WRONG
---
- name: Deploy
  hosts: all
  roles:
    - common
    - nginx
    - app
    - monitoring

# To skip monitoring: No easy way! Modify playbook

# ✅ CORRECT
---
- name: Deploy
  hosts: all
  roles:
    - role: common
      tags: always
    - role: nginx
      tags: webserver
    - role: app
      tags: application
    - role: monitoring
      tags: monitoring

# Usage:
$ ansible-playbook deploy.yml                    # All roles
$ ansible-playbook deploy.yml --tags application # Only app role
$ ansible-playbook deploy.yml --skip-tags monitoring # Skip monitoring
$ ansible-playbook deploy.yml -t webserver,application
```

### Mistake 4: Complex Role, No Task Organization

```yaml
# ❌ WRONG
roles/kubernetes/tasks/main.yml (1000+ lines!)
- task 1
- task 2
- task 3
...
- task 100

# Impossible to navigate or maintain

# ✅ CORRECT
roles/kubernetes/tasks/main.yml
---
- name: Install kubelet
  include_tasks: kubelet.yml

- name: Configure networking
  include_tasks: networking.yml

- name: Setup RBAC
  include_tasks: rbac.yml

- name: Configure monitoring
  include_tasks: monitoring.yml
  when: enable_monitoring

# roles/kubernetes/tasks/kubelet.yml (100 lines)
# roles/kubernetes/tasks/networking.yml (100 lines)
# Organized, maintainable!
```

### Mistake 5: Not Using Role Prefix for Galaxy Roles

```yaml
# ❌ WRONG (confusing)
---
roles:
  - geerlingguy.java     # From Galaxy
  - nginx                # Local role
  - geerlingguy.docker   # From Galaxy

# Hard to tell which are local vs external

# ✅ CORRECT (with example)
# Install external roles first:
$ ansible-galaxy install geerlingguy.java

# Then in playbook:
---
roles:
  - common                    # Local: common justification clear
  - geerlingguy.java          # External: Galaxy prefix obvious
  - nginx                     # Local
  - geerlingguy.docker        # External
  - geerlingguy.kubernetes    # External
  
# Or organize in requirements.yml:
---
- name: geerlingguy.java
  src: geerlingguy.java
  version: 2.0.0

- name: geerlingguy.docker
  src: geerlingguy.docker
  version: 5.0.0

# Install all:
$ ansible-galaxy install -r requirements.yml
```

---

## 6. Debugging Roles

### Scenario 1: Role Task Failing

```bash
# Run with verbose output
$ ansible-playbook playbook.yml -vvv

# Output shows:
TASK [rolename : Task name] FAILED - ...

# Debug steps:
1. Identify which task failed
2. Check role's tasks/main.yml for include_tasks
3. Check included file for the failing task
4. Add debug statement before task:
   - debug: msg="About to run X"
   
5. Re-run to see where failure occurs

$ ansible-playbook playbook.yml -vvv | grep FAILED
```

### Scenario 2: Role Variables Not Set

```bash
# Problem: Variable shows {{ var }} not substituted

# Step 1: Check variable defined in role
$ grep var roles/myrole/defaults/main.yml
var: default_value

# Step 2: Check if variable overridden
$ grep -r "var:" .

# Step 3: Print all role variables
- name: Show role vars
  debug:
    var: hostvars[inventory_hostname]
  # Should show var with value

# Step 4: Check variable precedence
defaults/main.yml → vars/main.yml → playbook vars → CLI vars
# Which one takes precedence?

# Fix: Add explicit debugging
roles/myrole/tasks/main.yml:
- debug: msg="var is {{ var }}"  # Before using var
- template:
    src: config.j2
    dest: /etc/config.conf       # Uses var
```

### Scenario 3: Role Not Executing

```bash
# Problem: Role tasks not running

# Step 1: Check role path
$ ansible-playbook playbook.yml -vvv | grep -i "role search"
Role search path: ./roles

# Step 2: Verify role exists
$ ls -la roles/myrole/
# Role directory exists?

# Step 3: Verify tasks/main.yml exists
$ ls -la roles/myrole/tasks/main.yml
# Must exist! (other dirs optional)

# Step 4: Check playbook references role
$ grep myrole playbook.yml
roles:
  - myrole

# Step 5: Check if role conditionally skipped
$ grep 'when:' roles/myrole/tasks/main.yml
when: some_condition
# If condition false, role skipped

# Fix: Ensure role path, tasks/main.yml, reference in playbook
```

---

## 7. Interview Questions (15+ Scenario-Based)

### Level 1: Role Basics

**Q1**: What's the purpose of `defaults/main.yml` vs `vars/main.yml`?

**Answer**:
```yaml
# defaults/main.yml (lowest priority)
# User CAN override these values
nginx_version: 1.18.0          # Default version
nginx_port: 80                 # Can be overridden
enable_ssl: no

# vars/main.yml (higher priority than defaults)
# Role-specific, usually NOT overridden
app_user: "{{ app_name }}-user"  # Computed from other vars
app_group: "{{ app_name }}-group"
data_dir: "/var/lib/{{ app_name }}"

# Usage:
# Override defaults:
roles:
  - role: nginx
    vars:
      nginx_version: 1.20.0  # Override default

# Don't override vars:
# vars/main.yml determines computed values
```

---

**Q2**: How do role dependencies work?

**Answer**:
```yaml
# roles/myapp/meta/main.yml
---
dependencies:
  - role: java          # Local role
  - role: docker        # Local role
  - geerlingguy.nginx   # External role

# When playbook includes myapp:
---
- hosts: servers
  roles:
    - myapp

# Ansible executes:
# 1. java (dependency)
# 2. docker (dependency)
# 3. geerlingguy.nginx (external dependency)
# 4. myapp (main role)

# Dependencies run even if referenced role skipped!
# Use 'when' in meta to make conditional:
dependencies:
  - role: monitoring
    when: enable_monitoring is defined
```

---

### Level 2: Advanced Roles

**Q3**: Design a reusable role for application deployment with version management.

**Answer**:
```yaml
# roles/app_deploy/defaults/main.yml
---
app_name: myapp
app_version: 1.0.0              # Can be overridden
app_owner: appuser
app_group: appgroup
app_home: "/opt/{{ app_name }}"
app_repo: https://github.com/... # Git repo
enable_monitoring: no
enable_logging: no

# roles/app_deploy/tasks/main.yml
---
- name: Create app user
  user:
    name: "{{ app_owner }}"
    home: "{{ app_home }}"

- name: Clone application from git
  git:
    repo: "{{ app_repo }}"
    dest: "{{ app_home }}"
    version: "v{{ app_version }}"
  register: app_cloned

- name: Install dependencies
  shell: "cd {{ app_home }} && npm install"
  when: app_cloned.changed

- name: Setup monitoring
  include_tasks: monitoring.yml
  when: enable_monitoring

- name: Start application
  systemd:
    name: "{{ app_name }}"
    state: started
    enabled: yes

# Usage across environments:
# Playbook for production:
- hosts: prod-servers
  roles:
    - role: app_deploy
      vars:
        app_version: 1.2.3 (production version)
        enable_monitoring: yes
        enable_logging: yes

# Playbook for staging:
- hosts: staging-servers
  roles:
    - role: app_deploy
      vars:
        app_version: 1.3.0-rc1 (release candidate)
        enable_monitoring: no

# Same role, different implementations!
```

---

## 8. Advanced Insights

### Pattern 1: Role Orchestration with Groups

```yaml
---
- name: Deploy full platform
  hosts: localhost
  gather_facts: no
  
  tasks:
    # Play 1: Setup infrastructure
    - name: Configure infrastructure
      include_role:
        name: infrastructure
      vars:
        env: "{{ environment }}"
    
    # Play 2: Setup services
    - name: Configure services
      include_role:
        name: "{{ item }}"
      loop:
        - database
        - cache
        - message_queue
    
    # Play 3: Deploy applications
    - name: Deploy applications
      include_role:
        name: "{{ item }}"
      loop:
        - api_backend
        - web_frontend
        - worker_service

# Orchestrate roles programmatically!
```

### Pattern 2: Testing Roles with Molecule

```bash
# Install Molecule (role testing framework)
$ pip install molecule

# Initialize role for testing
$ molecule init role app_deploy

# Run tests
$ molecule test
├─ Lint role
├─ Create test container
├─ Setup role
├─ Run role
├─ Verify role (check idempotency, assertions)
└─ Cleanup

# test/default/converge.yml (how to test role)
---
- name: Converge
  hosts: all
  roles:
    - role: app_deploy

# test/default/verify.yml (what to check)
---
- name: Verify
  hosts: all
  tasks:
    - name: Verify app is running
      command: systemctl is-active myapp
      register: result
      assert:
        that:
          - result.rc == 0

# Ensures role works correctly!
```

---

## Conclusion

Roles are the **building blocks** of enterprise Ansible:

1. **Modular**: Organize by functionality (database, monitoring, web)
2. **Reusable**: Share across projects
3. **Maintainable**: Clear structure, easy to locate code
4. **Scalable**: Handle complexity through composition
5. **Testable**: Independently test each role

Master roles and you can:
- ✅ Build reusable infrastructure code
- ✅ Share roles across teams
- ✅ Scale from 10s to 1000s of servers
- ✅ Maintain consistency across environments

**Key principle**: Write roles for reuse, playbooks for orchestration!
