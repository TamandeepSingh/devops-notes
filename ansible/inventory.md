# Ansible Inventory: Host Management and Management

## 1. Core Concept

Inventory is the **source of truth** for managed servers:

1. **Lists hosts**: Servers to manage via Ansible
2. **Organizes groups**: Group hosts by role/environment/region
3. **Stores variables**: Host-specific and group-specific configuration
4. **Supports dynamic**: Generate host list at runtime (AWS, Azure, Kubernetes, etc)

### Inventory Formats

Three formats supported:

**INI Format**:
```ini
# inventory.ini or hosts.ini
[webservers]
web1.example.com
web2.example.com

[databases]
db.example.com

[all:vars]
ansible_user=admin
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

**YAML Format**:
```yaml
# inventory.yml or hosts.yml
all:
  children:
    webservers:
      hosts:
        web1.example.com:
          nginx_port: 80
        web2.example.com:
          nginx_port: 80
    databases:
      hosts:
        db.example.com:
          db_port: 5432
  vars:
    ansible_user: admin
```

**TOML Format**:
```toml
# inventory.toml
[webservers]
web1.example.com nginx_port=80
web2.example.com nginx_port=80

[databases]
db.example.com db_port=5432

[all.vars]
ansible_user="admin"
```

---

## 2. How Ansible Inventory Works Internally

### Inventory Loading Pipeline

```
Step 1: Accept inventory source
├─ -i /path/to/inventory      (file)
├─ -i /path/to/script.py       (dynamic script)
└─ -i host1.com,host2.com,     (comma-separated list)

Step 2: Parse inventory
├─ Detect format (INI, YAML, dynamic)
├─ Load configuration
└─ Build in-memory inventory

Step 3: Organize into groups
├─ explicit groups ([webservers], [databases])
├─ implicit groups (all, ungrouped)
├─ group_vars/variables
└─ host_vars/variables

Step 4: Merge variables (precedence)
├─ Ansible defaults (lowest)
├─ Group variables (group_vars/all.yml, group_vars/GROUPNAME.yml)
├─ Host variables (host_vars/HOSTNAME.yml)
├─ Extra vars (--extra-vars)
└─ CLI overrides (highest)

Step 5: Build connection map
├─ Host → ansible_host (IP to connect to)
├─ Host → ansible_user (SSH user)
├─ Host → ansible_port (SSH port)
└─ Other connection settings

Step 6: Ready for playbook execution
```

### Groups and Host Patterns

```ini
# inventory.ini
[webservers]
web1.example.com
web2.example.com
web3.example.com

[databases]
db1.example.com
db2.example.com

[caches]
cache1.example.com

[prod:children]  # Meta-group: contains other groups
webservers
databases
caches

[dev]
dev-web.example.com
dev-db.example.com

# Implicit groups:
# [all] - all hosts
# [ungrouped] - hosts not in any group

# Selection patterns:
ansible all              # All hosts
ansible webservers      # webservers group
ansible prod            # prod group (includes children)
ansible 'prod:!databases'  # prod except databases
ansible '*example.com'  # Hosts matching pattern
ansible localhost       # Local machine
```

### Variable Resolution

```yaml
# Directory structure
.
├── inventory.yml
├── group_vars/
│   ├── all.yml                # All hosts
│   ├── webservers.yml         # Webservers group
│   └── prod.yml               # Production group
└── host_vars/
    ├── web1.example.com.yml   # Specific host
    └── db1.example.com.yml

# inventory.yml
all:
  vars:
    default_user: admin
  children:
    webservers:
      vars:
        nginx_port: 80
      hosts:
        web1.example.com:
          app_env: production   # Host-specific var
        web2.example.com:
          app_env: staging

# group_vars/all.yml
---
log_level: info
timeout: 30

# group_vars/webservers.yml
---
nginx_version: 1.20.0
worker_processes: 4

# host_vars/web1.example.com.yml
---
app_env: production
cpu_limit: 4

# Final variables for web1:
ansible_user: admin      (from inventory/all)
log_level: info           (from group_vars/all.yml)
nginx_version: 1.20.0     (from group_vars/webservers.yml)
app_env: production       (from host_vars/web1.example.com.yml)
cpu_limit: 4              (from host_vars/web1.example.com.yml)
```

### Dynamic Inventory

```python
#!/usr/bin/env python3
# inventory.py (dynamic inventory script)
import json
import boto3

ec2 = boto3.client('ec2', region_name='us-east-1')

response = ec2.describe_instances(
    Filters=[
        {'Name': 'instance-state-name', 'Values': ['running']},
        {'Name': 'tag:ManagedBy', 'Values': ['Ansible']}
    ]
)

inventory = {
    'webservers': {
        'hosts': [],
        'vars': {'ansible_user': 'admin'}
    },
    'databases': {
        'hosts': [],
        'vars': {'ansible_user': 'admin'}
    },
    '_meta': {
        'hostvars': {}
    }
}

for reservation in response['Reservations']:
    for instance in reservation['Instances']:
        host_ip = instance['PrivateIpAddress']
        host_vars = {
            'ansible_host': host_ip,
            'instance_id': instance['InstanceId'],
            'instance_type': instance['InstanceType']
        }
        
        # Categorize by tag
        for tag in instance['Tags']:
            if tag['Key'] == 'Role':
                if tag['Value'] == 'webserver':
                    inventory['webservers']['hosts'].append(host_ip)
                elif tag['Value'] == 'database':
                    inventory['databases']['hosts'].append(host_ip)
        
        inventory['_meta']['hostvars'][host_ip] = host_vars

print(json.dumps(inventory))

# Usage:
$ ansible-playbook -i inventory.py playbook.yml
# Automatically discovers running EC2 instances!
```

---

## 3. Static vs Dynamic Inventory

### Static Inventory (File-Based)

```ini
# Good for: Small, stable environments
# inventory/prod.ini
[webservers]
web1.prod.example.com
web2.prod.example.com

[databases]
db1.prod.example.com

# Run playbook:
$ ansible-playbook -i inventory/prod.ini playbook.yml

Advantages:
✅ Simple, no runtime overhead
✅ Version controlled (Git)
✅ Fast (no API calls)

Disadvantages:
❌ Manual updates (add/remove servers)
❌ Out-of-sync with cloud (if servers terminated)
```

### Dynamic Inventory (Runtime)

```python
#!/usr/bin/env python3
# inventory_aws.py
# Pulls running instances from AWS at runtime

# Run playbook:
$ ansible-playbook -i inventory_aws.py playbook.yml

# Ansible calls: ./inventory_aws.py --list
# Returns: JSON with current running instances

Advantages:
✅ Always up-to-date (discovers current resources)
✅ Scales automatically (add servers, inventory grows)
✅ No manual updates needed

Disadvantages:
❌ Runtime overhead (API calls)
❌ Requires cloud SDK/credentials
❌ More complex to set up
```

### Hybrid Approach

```ini
# static.ini (controlled servers)
[webservers_static]
web1.example.com

# dynamic via script
$ ansible-playbook -i static.ini -i inventory_aws.py playbook.yml

# Combines both:
- Static: Servers you manage directly
- Dynamic: Cloud instances discovered automatically
```

---

## 4. Real-World Usage

### Scenario 1: Multi-Environment Inventory

```
inventory/
├── production/
│   ├── inventory.yml
│   ├── group_vars/
│   │   ├── all.yml
│   │   ├── webservers.yml
│   │   └── databases.yml
│   └── host_vars/
│       ├── web1.prod.example.com.yml
│       └── db1.prod.example.com.yml
│
├── staging/
│   ├── inventory.yml
│   ├── group_vars/
│   │   ├── all.yml
│   │   └── webservers.yml
│   └── host_vars/
│
└── development/
    ├── inventory.yml
    └── group_vars/

# inventory/production/inventory.yml
---
all:
  children:
    webservers:
      hosts:
        web1.prod.example.com:
        web2.prod.example.com:
    databases:
      hosts:
        db1.prod.example.com:

# inventory/production/group_vars/webservers.yml
---
nginx_version: 1.20.0
app_version: 1.2.3
replicas: 5
environment: production

# inventory/staging/group_vars/webservers.yml
---
nginx_version: 1.19.0
app_version: 1.3.0-rc1
replicas: 2
environment: staging

# Usage:
$ ansible-playbook -i inventory/production playbook.yml
$ ansible-playbook -i inventory/staging playbook.yml

# Same playbook, different environments!
```

### Scenario 2: Group Variables with Inheritance

```yaml
# inventory.yml
---
all:
  vars:
    ansible_user: admin
    dns_servers:
      - 8.8.8.8
      - 1.1.1.1
  children:
    prod:
      children:
        webservers:
          hosts:
            web[1:3].prod.example.com:
        databases:
          hosts:
            db1.prod.example.com:
    
    staging:
      children:
        webservers:
          hosts:
            web.staging.example.com:

# group_vars/all.yml
---
environment: general
log_level: info

# group_vars/prod.yml (overrides all)
---
environment: production
log_level: warning
syslog_enabled: yes

# group_vars/prod/webservers.yml
---
nginx_workers: 8
enable_ssl: yes

# Final variables for web1.prod.example.com:
ansible_user: admin            (from all)
dns_servers: [8.8.8.8, ...]   (from all)
environment: production         (from prod)
log_level: warning              (from prod)
syslog_enabled: yes             (from prod)
nginx_workers: 8                (from prod/webservers)
enable_ssl: yes                 (from prod/webservers)
```

### Scenario 3: AWS Dynamic Inventory

```python
#!/usr/bin/env python3
# aws_ec2.py (AWS EC2 dynamic inventory)
import json
import boto3

def get_inventory():
    ec2 = boto3.resource('ec2', region_name='us-west-2')
    
    inventory = {
        'webservers': {'hosts': [], 'vars': {}},
        'databases': {'hosts': [], 'vars': {}},
        'caches': {'hosts': [], 'vars': {}},
        '_meta': {'hostvars': {}}
    }
    
    # Find all running instances tagged with Ansible=true
    instances = ec2.instances.filter(
        Filters=[
            {'Name': 'instance-state-name', 'Values': ['running']},
            {'Name': 'tag:Ansible', 'Values': ['true']}
        ]
    )
    
    for instance in instances:
        private_ip = instance.private_ip_address
        
        # Get tags
        tags = {tag['Key']: tag['Value'] for tag in instance.tags or []}
        
        # Categorize by role tag
        role = tags.get('Role', 'ungrouped')
        if role == 'webserver':
            inventory['webservers']['hosts'].append(private_ip)
        elif role == 'database':
            inventory['databases']['hosts'].append(private_ip)
        elif role == 'cache':
            inventory['caches']['hosts'].append(private_ip)
        
        # Store host variables
        host_vars = {
            'ansible_host': private_ip,
            'public_ip': instance.public_ip_address,
            'instance_id': instance.id,
            'instance_type': instance.instance_type,
            'availability_zone': instance.placement['AvailabilityZone'],
            'security_groups': [sg['GroupName'] for sg in instance.security_groups],
            'tags': tags
        }
        
        inventory['_meta']['hostvars'][private_ip] = host_vars
    
    return inventory

if __name__ == '__main__':
    print(json.dumps(get_inventory(), indent=2))

# Usage:
$ ansible-playbook -i aws_ec2.py playbook.yml

# Inventory auto-populated from AWS EC2 instances!
# New instances automatically included
# Terminated instances automatically removed
```

---

## 5. Common Mistakes

### Mistake 1: Not Organizing Group Variables

```ini
# ❌ WRONG - Everything in one file
inventory.ini:
[webservers]
web1 nginx_version=1.20.0 app_port=8080 worker_threads=4 enable_ssl=yes

# Messy, hard to maintain

# ✅ CORRECT - Organized structure
inventory/
├── inventory.yml
├── group_vars/
│   └── webservers.yml  # All webserver variables
└── host_vars/
    └── web1/           # web1-specific variables

# group_vars/webservers.yml
---
nginx_version: 1.20.0
worker_threads: 4
enable_ssl: yes

# host_vars/web1/main.yml
---
app_port: 8080
```

### Mistake 2: Hardcoding IPs in Inventory

```ini
# ❌ WRONG
[webservers]
192.168.1.10
192.168.1.11

# If IP changes → inventory breaks

# ✅ CORRECT
[webservers]
web1.example.com
web2.example.com

# Or with dynamic:
# inventory_aws.py (discovers IPs)
```

### Mistake 3: Not Using ansible_host for Non-SSH Access

```ini
# ❌ WRONG
[databases]
db1.internal.network

# Ansible tries to SSH to db1.internal.network
# But that's not resolvable from control machine!

# ✅ CORRECT
[databases]
db1 ansible_host=10.0.1.20  # IP for SSH connection

# Or in inventory.yml:
databases:
  hosts:
    db1:
      ansible_host: 10.0.1.20   # SSH connects here
      db_hostname: db1.internal  # App uses this name
```

### Mistake 4: Variable Conflicts Across Groups

```yaml
# ❌ WRONG
group_vars/all.yml:
log_level: info

group_vars/prod.yml:
log_level: info              # Same variable!

group_vars/webservers.yml:
log_level: debug             # Different value!

# Confusing precedence! Which takes effect?

# ✅ CORRECT
group_vars/all.yml:
default_log_level: info

group_vars/prod.yml:
environment: production

group_vars/webservers.yml:
app_log_level: debug         # Different name, clear intent!
```

### Mistake 5: Not Using Aliases for Complex Hostnames

```ini
# ❌ WRONG
[webservers]
very-long-internal-hostname-web1.us-west-2.internal.example.com ansible_host=10.0.1.1
very-long-internal-hostname-web2.us-west-2.internal.example.com ansible_host=10.0.1.2

# Verbose, hard to read

# ✅ CORRECT
[webservers]
web1 ansible_host=10.0.1.1
web2 ansible_host=10.0.1.2

# Or in playbook:
- hosts: web1
  tasks:
    - debug: msg="Working on {{ inventory_hostname }}"  # Alias
    - debug: msg="Connecting to {{ ansible_host }}"    # Real connection
```

---

## 6. Debugging Inventory Issues

### Scenario 1: Host Not Found in Inventory

```bash
# Problem: ansible webservers -m ping
# Error: "no hosts matched"

# Debug Step 1: List all hosts
$ ansible-inventory -i inventory.yml --list

# Check if host group exists
$ ansible-inventory -i inventory.yml --host web1

# Step 2: Check inventory syntax
$ ansible-inventory -i inventory.yml --list | jq
# Should output valid JSON

# Step 3: Check group name
$ grep webservers inventory.yml
# Is it exactly spelled "webservers"?

# Step 4: Check host definitions
$ grep web inventory.yml
# Are hosts defined under [webservers] group?

# Fix: Verify inventory format and group names match
```

### Scenario 2: Variables Not Applying

```bash
# Problem: ansible web1 -m debug -a "msg={{ log_level }}"
# Shows: Variable not defined

# Debug Step 1: List all variables for host
$ ansible-inventory -i inventory.yml --host web1

# Should show all variables merged

# Step 2: Check group_vars file
$ ls -la group_vars/
# File for group must exist

# Step 3: Verify group_vars format
$ ansible-inventory -i inventory.yml --list | jq._meta.hostvars | grep log_level
# Should show variable

# Step 4: Check variable precedence
# Group_vars applied in order:
# 1. group_vars/all.yml (all hosts)
# 2. group_vars/GROUPNAME.yml (specific group)
# 3. host_vars/HOSTNAME.yml (host-specific)

# Fix: Ensure variable defined in correct group_vars file
```

### Scenario 3: Dynamic Inventory Returns Empty

```bash
# Problem: ansible-inventory -i inventory.py --list
# Returns: empty inventory

# Debug Step 1: Run inventory script directly
$ python3 inventory.py --list
# Should output JSON with hosts

# Step 2: Make script executable
$ chmod +x inventory.py

# Step 3: Check if script has errors
$ python3 inventory.py --list 2>&1 | head
# Any error output?

# Step 4: Check credentials
# Script needs AWS credentials / API keys to work
$ aws ec2 describe-instances
# Can you access AWS manually?

# Step 5: Check filters
# Script filters instances by tag/status
# Verify instances match filters:
$ aws ec2 describe-instances \
  --filters Name=instance-state-name,Values=running
# Any running instances?

# Fix: Debug script, check credentials, verify filters
```

---

## 7. Interview Questions (Inventory-Specific)

### **Q1**: How would you structure inventory for multi-region deployment?

**Answer**:
```yaml
# inventory.yml for multi-region
all:
  children:
    region_us_west:
      children:
        us_west_webservers:
          hosts:
            web1.us-west-2.example.com:
        us_west_databases:
          hosts:
            db1.us-west-2.example.com:
    
    region_us_east:
      children:
        us_east_webservers:
          hosts:
            web1.us-east-1.example.com:
        us_east_databases:
          hosts:
            db1.us-east-1.example.com:

# group_vars/region_us_west.yml
---
region: us-west-2
dns_servers:
  - 8.8.8.8

# group_vars/region_us_east.yml
---
region: us-east-1
dns_servers:
  - 1.1.1.1

# Usage:
$ ansible-playbook -i inventory.yml -l region_us_west playbook.yml
$ ansible-playbook -i inventory.yml -l region_us_east playbook.yml
```

---

### **Q2**: When would you use dynamic vs static inventory?

**Answer**:
```
Static (File):
✅ Use when: Stable servers, development, simple setups
❌ Avoid when: Cloud-managed, frequently changing

$ ansible-playbook -i inventory.ini playbook.yml

Dynamic (Script):
✅ Use when: Cloud infrastructure, auto-scaling, large deployments
❌ Avoid when: Simple setups, offline environments

$ ansible-playbook -i inventory.py playbook.yml

Hybrid:
✅ Combine both: Static + Dynamic from different sources
$ ansible-playbook -i static.ini -i inventory.py playbook.yml
```

---

## 8. Advanced Inventory Patterns

### Pattern 1: Composition Plugins

```yaml
# ansible.cfg
[inventory]
plugins = aws_ec2, constructed

# inventory/aws_ec2.yml (dynamic AWS inventory)
plugin: aws_ec2
aws_profile: production
regions:
  - us-west-2
filters:
  tag:Environment: production
  instance-state-name: running
keyed_groups:
  - key: placement.region
    parent_group: regions
  - key: tags.Role
    parent_group: roles

# Result: Hosts grouped by Region + Role automatically!
```

### Pattern 2: Ansible Tower/AWX Integration

```yaml
# inventory managed in Tower API
# /api/v2/inventories/

# Playbook runs from Tower:
# Tower pulls inventory dynamically
# Adds RBAC (role-based access control)
# Tracks execution history
# Credential management
```

---

## Conclusion

Inventory is foundational:

1. **Organization**: Groups related hosts
2. **Flexibility**: Static and dynamic sources
3. **Scalability**: From 10 to 100,000 servers
4. **Maintainability**: Organized group/host variables

Master inventory and you can:
- ✅ Manage complex, multi-environment infrastructure
- ✅ Scale from static to cloud-managed
- ✅ Organize variables logically
- ✅ Integrate with cloud platforms

**Key principle**: Inventory is your infrastructure's blueprint - organize it well!
