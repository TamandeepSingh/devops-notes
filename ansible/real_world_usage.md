# Ansible in Production: Real-World Usage, CI/CD Integration, and Operations at Scale

## 1. Core Concept

Production Ansible requires:

1. **Reliability**: Idempotent, automated, recovery mechanisms
2. **Traceability**: Audit logs, version control, change tracking
3. **Safety**: Testing, approval workflows, rollback capability
4. **Scale**: Hundreds/thousands of servers, high concurrency
5. **Integration**: GitOps, CI/CD pipelines, cloud platforms

### Production Architecture

```
Source Control (Git)
  ├─ Playbooks
  ├─ Roles
  ├─ Inventory
  └─ Scripts

CI/CD Pipeline (GitHub Actions, GitLab CI, Jenkins)
  ├─ Syntax validation
  ├─ Linting
  ├─ Testing
  ├─ Approval gates
  └─ Trigger deployment

Ansible Tower/AWX (Enterprise)
  ├─ RBAC (Role-Based Access Control)
  ├─ Credential management
  ├─ Workflow orchestration
  ├─ Audit logging
  └─ REST API

Target Infrastructure
  ├─ Production servers
  ├─ Staging servers
  └─ Development servers
```

---

## 2. How Production Ansible Differs From Dev

### Development (Simple)

```bash
# Dev environment:
$ ansible-playbook playbook.yml

# Run immediately
# Limited error handling
# No approval required
# Manual testing only
```

### Production (Robust)

```bash
# Production workflow:

# 1. Validate (syntax, lint, tests)
$ ansible-lint playbook.yml
$ ansible-playbook --syntax-check playbook.yml
$ molecule test roles/myrole

# 2. Dry-run (preview changes)
$ ansible-playbook -i inventory/prod.ini playbook.yml --check

# 3. Test in staging
$ ansible-playbook -i inventory/staging.ini playbook.yml

# 4. Approval gate (human review)
$ echo "Approved for production" # Via Tower/GitLab/Slack

# 5. Deploy to production
$ ansible-playbook -i inventory/prod.ini playbook.yml \
  --tags=production \
  --extra-vars "deployment_id={{ deployment_id }}"

# 6. Verify
$ ansible-playbook -i inventory/prod.ini \
  playbook_verify.yml

# 7. Rollback (if needed)
$ ansible-playbook -i inventory/prod.ini \
  playbook_rollback.yml
```

---

## 3. When to Use Ansible vs Terraform (Production)

### Ansible Strong Suits

✅ **Configuration management** (packages, services, configs)
```yaml
---
- hosts: webservers
  roles:
    - nginx
    - application
    - monitoring
```

✅ **Application deployment & updates**
```yaml
---
- hosts: app_servers
  serial: 1
  tasks:
    - name: Deploy new version
      git:
        repo: https://github.com/app
        version: "{{ app_version }}"
```

✅ **Orchestration & callbacks**
```yaml
---
- hosts: all
  tasks:
    - name: Health check
      uri:
        url: http://localhost/health
      retries: 3
      delay: 10
```

### Terraform Strong Suits

✅ **Infrastructure provisioning** (cloud resources)
```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.medium"
  count         = 3
}
```

✅ **State management** (track what exists)
```hcl
# terraform.tfstate tracks all resources
# Easy to audit, modify, destroy
```

### Recommended Production Pattern

```
Terraform (IaC):
├─ Provision cloud infrastructure (VMs, networks, LBs)
├─ Outputs: IP addresses, hostnames
└─ Managed by: Infrastructure team

Ansible (CM):
├─ Configure provisioned infrastructure
├─ Deploy applications
├─ Manage ongoing changes
└─ Managed by: DevOps/SRE team

Both version controlled in Git:
├─ Git push → CI/CD
├─ CI/CD → Terraform validate/apply
├─ CI/CD → Ansible playbook
└─ Result: Tested, reviewed, deployed infrastructure!
```

---

## 4. Real-World Production Scenarios

### Scenario 1: GitOps-Based Configuration Management

```
Repository Structure:
├── playbooks/
│   ├── deploy.yml
│   ├── update.yml
│   ├── rollback.yml
│   └── verify.yml
│
├── roles/
│   ├── base/
│   ├── nginx/
│   ├── app/
│   └── monitoring/
│
├── inventory/
│   ├── production.yml
│   ├── staging.yml
│   └── development.yml
│
├── group_vars/
│   └── prod/
│       ├── all.yml
│       └── webservers.yml
│
└── .github/workflows/
    ├── validate.yml
    ├── deploy-staging.yml
    └── deploy-prod.yml

GitHub Actions Workflow:
1. Developer commits changes
   └─ Triggers: validate.yml

2. Validation pipeline
   ├─ ansible-lint playbooks/
   ├─ ansible-playbook --syntax-check
   ├─ molecule test roles/
   └─ pytest tests/

3. All checks pass
   └─ Pull request approved by reviewers

4. Merge to main
   └─ Triggers: deploy-staging.yml

5. Deploy to staging
   └─ ansible-playbook -i inventory/staging.yml playbooks/deploy.yml

6. Manual approval + verification
   └─ Team checks metrics, logs, health

7. Deploy to production (manual trigger)
   └─ ansible-playbook -i inventory/production.yml playbooks/deploy.yml

8. Verify production
   └─ ansible-playbook -i inventory/production.yml playbooks/verify.yml

Result: Zero-trust deployment!
- All changes git-driven
- All changes have approval trail
- All changes tested
- Rollback: simple git revert
```

### Scenario 2: Blue-Green Deployment

```yaml
---
- name: Blue-Green Deployment
  hosts: localhost
  gather_facts: no
  vars:
    current_version: v1.0.0
    new_version: v1.1.0

  pre_tasks:
    - name: Get current active color
      shell: cat /var/lib/deployment/active_color
      register: active_color
      ignore_errors: yes

  tasks:
    # Determine which color is inactive
    - name: Determine inactive color
      set_fact:
        inactive_color: "{{ 'green' if (active_color.stdout | trim) == 'blue' else 'blue' }}"
    
    - name: Deploy to inactive color
      debug:
        msg: "Deploying {{ new_version }} to {{ inactive_color }}"
    
    - name: Configure {{ inactive_color }} servers
      hosts: "{{ inactive_color }}_servers"
      roles:
        - role: application
          vars:
            app_version: "{{ hostvars['localhost']['new_version'] }}"
      register: deployment_result
  
  post_tasks:
    - name: Health check on {{ inactive_color }}
      uri:
        url: "http://{{ item }}:8080/health"
        status_code: 200
      retries: 5
      delay: 10
      loop: "{{ groups[inactive_color + '_servers'] }}"
      register: health_check
    
    - name: Switch traffic to {{ inactive_color }}
      block:
        - name: Update load balancer
          shell: |
            curl -X PUT http://lb.example.com/target-group \
              -d "target={{ inactive_color }}"
        
        - name: Mark as active
          shell: echo "{{ inactive_color }}" > /var/lib/deployment/active_color
      when: health_check is succeeded
      rescue:
        - name: Rollback failed
          debug:
            msg: "Deployment failed, keeping {{ active_color.stdout }} active"

{{ inventory_hostname }}: {{ health_status }}
        # Result: Instant rollback to previous version if health failed
```

### Scenario 3: Multi-Region Deployment with Distributed Management

```yaml
---
- name: Multi-Region Deployment Orchestration
  hosts: localhost
  gather_facts: no

  vars:
    regions:
      - name: us-west-2
        priority: 1
        max_fail_percentage: 0
      - name: us-east-1
        priority: 2
        max_fail_percentage: 0
      - name: eu-west-1
        priority: 3
        max_fail_percentage: 10  # Allow 10% failures in EU

  pre_tasks:
    - name: Wait for health across regions
      uri:
        url: "https://{{ item }}.api.example.com/health"
        status_code: 200
      retries: 3
      delay: 5
      loop: "{{ regions | map(attribute='name') | list }}"
      register: regional_health
    
    - name: Verify all regions healthy
      assert:
        that:
          - regional_health is succeeded
        fail_msg: "One or more regions unhealthy"

  tasks:
    - name: Deploy to each region sequentially
      include_tasks: deploy_region.yml
      loop: "{{ regions }}"
      vars:
        loop_var: region_item

# deploy_region.yml
---
- name: Deploy to {{ region_item.name }}
  hosts: "region_{{ region_item.name }}"
  max_fail_percentage: "{{ region_item.max_fail_percentage | default(0) }}"
  serial: "{{ (groups[f'region_{region_item.name}'] | length * 0.2) | int }}"
  
  tasks:
    - name: Deploy application
      include_role:
        name: application
      vars:
        app_version: "{{ deployment_version }}"
    
    - name: Verify deployment
      uri:
        url: "http://localhost:8080/health"
        status_code: 200
      retries: 10
      delay: 5

# Result: Serial regional deployment with safety checks
# If one region fails: Stops and alerts
# If percentage threshold exceeded: Stops and rolls back
```

### Scenario 4: Automated Remediation (Self-Healing)

```yaml
---
- name: Continuous Monitoring & Self-Healing
  hosts: all
  gather_facts: no
  tasks:
    - block:
        - name: Check disk usage
          shell: df -h / | awk 'NR==2 {print $5}' | sed 's/%//'
          register: disk_usage
        
        - name: Alert if disk usage high
          debug:
            msg: "ALERT: Disk usage {{ disk_usage.stdout }}%"
          when: disk_usage.stdout | int > 80
        
        - name: Clean old logs (remediation)
          shell: find /var/log -name "*.log" -mtime +30 -delete
          when: disk_usage.stdout | int > 85
          register: cleanup_result
        
        - name: Verify cleanup
          debug:
            msg: "Cleaned up old logs"
          when: cleanup_result is changed
      
      # Similar blocks for:
      # - Memory pressure
      # - Failed services
      # - Configuration drift
      # - SSL certificate expiration
    
    - name: Report metrics
      shell: |
        echo "disk_usage={{ disk_usage.stdout }}" >> /var/log/health.log
      when: disk_usage is defined

# Run this playbook via cron (every 5 minutes)
# $ crontab -e
# */5 * * * * ansible-playbook -i inventory/prod.ini health-check.yml

# Benefits:
# - Automatic remediation (no human intervention)
# - Consistent actions (always the same fix)
# - Audit trail (logs all actions)
# - Escalation (alert if remediation fails)
```

---

## 5. Common Production Mistakes

### Mistake 1: No Dry-Run Before Production

```bash
# ❌ WRONG
$ ansible-playbook -i inventory/prod.ini playbook.yml
# Applied directly!

# ✅ CORRECT
# Always preview changes first:
$ ansible-playbook -i inventory/prod.ini playbook.yml --check
# Shows what WOULD change, doesn't apply

# Then apply:
$ ansible-playbook -i inventory/prod.ini playbook.yml
```

### Mistake 2: Not Implementing Forks Limit

```bash
# ❌ WRONG
$ ansible-playbook -i inventory/prod.yml playbook.yml
# Defaults to: forks=5 (parallel execution)
# 1000 servers = 200 batches = slow

# ✅ CORRECT
$ ansible-playbook -i inventory/prod.yml playbook.yml \
  -f 50
# Run 50 hosts in parallel (faster)

# Or in ansible.cfg:
[defaults]
forks = 50

# Balance between speed and system load!
```

### Mistake 3: Not Using serial for Rolling Deployment

```yaml
# ❌ WRONG (all at once)
- hosts: webservers
  tasks:
    - name: Deploy
      ...
# All 100 web servers deploy simultaneously
# If bad → All 100 broken!

# ✅ CORRECT (rolling)
- hosts: webservers
  serial: 10          # 10 at a time
  max_fail_percentage: 5  # Stop if 5%+ fail
  tasks:
    - name: Deploy
      ...
# Batch 1: 10 servers deploy
# Verify → continue
# Batch 2: 10 servers deploy
# Gradual rollout, safer!
```

### Mistake 4: No Idempotency Checks

```yaml
# ❌ WRONG (non-idempotent)
- name: Update app
  command: cp /tmp/app.jar /opt/app/app.jar
  # Running twice = copies twice
  # But file same = no issue

- name: Run migration
  command: python manage.py migrate
  # Running twice = tries to migrate twice!
  # May fail or corrupt data!

# ✅ CORRECT (idempotent)
- name: Check if migrated
  stat:
    path: /var/lib/app/.migrated_{{ app_version }}
  register: migrated

- name: Run migration
  command: python manage.py migrate
  register: migration_result
  when: not migrated.stat.exists

- name: Mark migration done
  file:
    path: /var/lib/app/.migrated_{{ app_version }}
    state: touch
  when: migration_result is succeeded

# Always safe to run multiple times!
```

### Mistake 5: Not Logging Execution

```bash
# ❌ WRONG (no history)
$ ansible-playbook playbook.yml
# Output goes to console only
# No record for auditing/debugging

# ✅ CORRECT (log everything)
$ ansible-playbook playbook.yml \
  -i inventory/prod.ini \
  -vvv \
  2>&1 | tee /var/log/ansible/deployment-{{ date }}.log

# Or in ansible.cfg:
[defaults]
log_path = /var/log/ansible.log

# Combined with:
- CloudWatch/ELK integration
- Slack notifications
- Change tracking in Git
```

---

## 6. Debugging Production Issues

### Scenario 1: Playbook Failed Halfway

```bash
# Problem: Deployment crashed after task 5 of 20

# Debug: Use --start-at-task to resume
$ ansible-playbook -i inventory/prod.ini playbook.yml \
  --start-at-task "5. Task name"

# Alternative: Run specific task
$ ansible-playbook -i inventory/prod.ini playbook.yml \
  --tags "specific_task_tag"

# Or specific hosts
$ ansible-playbook -i inventory/prod.ini playbook.yml \
  -l "specific_host"

# Step-by-step:
$ ansible-playbook -i inventory/prod.ini playbook.yml \
  --step
# (y/n/c)? for each task
```

### Scenario 2: Host Unreachable

```bash
# Problem: Cannot SSH to host

# Debug Step 1: Test SSH manually
$ ssh -v admin@web1.example.com
# Shows connection details

# Step 2: Check inventory
$ ansible-inventory -i inventory/prod.ini --host web1.example.com
# Shows host variables and connection info

# Step 3: Test connectivity via Ansible
$ ansible web1.example.com -m ping
# UNREACHABLE: Unable to connect

# Step 4: Check ansible.cfg
$ cat ansible.cfg | grep -i host_key_checking
host_key_checking = False  # Skip SSH key verification

# Step 5: Check SSH key
$ ls -la ~/.ssh/id_rsa
# SSH key exists and readable?

# Fix: Update inventory with correct IP/hostname
ansible_host: 192.168.1.100
ansible_user: admin
ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

### Scenario 3: Task Output Shows Errors But Continued

```bash
# Problem: Task reported errors but playbook didn't fail

# Check task definition:
- name: Some task
  command: failing_command
  ignore_errors: yes  # ← This!
  # Intentionally ignores errors

# Or check error handling:
- block:
    - command: possible_failure
  rescue:
    - debug: msg="Error caught and handled"
  # Errors caught by rescue block

# To make it fail:
- name: Some task
  command: failing_command
  # Remove ignore_errors

# Or check output:
- name: Some task
  command: some_command
  register: result
  failed_when: result.rc != 0 and result.rc != 2
  # Custom failure condition
```

---

## 7. Interview Questions (Production-Focused)

### **Q1**: Design a safe production deployment process using Ansible.

**Answer**:
```yaml
---
# Step 1: Validation (automated)
- hosts: localhost
  tasks:
    - name: Lint playbooks
      shell: ansible-lint playbooks/
    
    - name: Syntax check
      shell: ansible-playbook --syntax-check playbooks/deploy.yml
    
    - name: Test roles
      shell: cd roles/myapp && molecule test

# Step 2: Approval gate (human)
# - Pull request review
# - Approval from 2+ maintainers
# - Changelog updated

# Step 3: Deploy to staging
- hosts: staging_servers
  tasks:
    - include_role: myrole
    
    - name: Run smoke tests
      uri:
        url: http://localhost/health
        status_code: 200

# Step 4: Wait for approval
# - Manual trigger or time-based

# Step 5: Deploy to production
- hosts: prod_servers
  serial: "{{ (groups['prod_servers'] | length * 0.1) | int }}"
  max_fail_percentage: 0
  
  pre_tasks:
    - name: Take backup
      shell: backup_script.sh
  
  tasks:
    - include_role: myrole
  
  post_tasks:
    - name: Verify deployment
      uri:
        url: http://localhost/health
        status_code: 200

# Step 6: Monitoring & alerting
# - Track error rates
# - Compare vs baseline
# - Alert if degradation detected
# - Auto-rollback if critical

# Result: Safe, audited, recoverable!
```

---

## 8. Advanced Insights & Patterns

### Pattern 1: Ansible with Container Orchestration

```yaml
---
# Deploy to Kubernetes using Ansible

- name: Deploy to Kubernetes
  hosts: localhost
  gather_facts: no
  
  vars:
    k8s_namespace: production
    image_tag: "{{ deployment_version }}"
  
  tasks:
    - name: Build container image
      shell: |
        docker build -t myapp:{{ image_tag }} .
        docker push {{ registry }}/myapp:{{ image_tag }}
    
    - name: Update deployment manifest
      template:
        src: deployment.yaml.j2
        dest: /tmp/deployment.yaml
      vars:
        app_image: "{{ registry }}/myapp:{{ image_tag }}"
    
    - name: Apply to Kubernetes
      kubernetes.core.k8s:
        kubeconfig: ~/.kube/config
        namespace: "{{ k8s_namespace }}"
        state: present
        src: /tmp/deployment.yaml
    
    - name: Wait for rollout
      kubernetes.core.k8s_info:
        kind: Deployment
        name: myapp
        namespace: "{{ k8s_namespace }}"
        wait: yes
        wait_condition:
          type: Available
          status: True
      register: rollout_status
    
    - name: Verify deployment
      uri:
        url: "http://myapp.{{ k8s_namespace }}.svc.cluster.local/health"
        status_code: 200
      retries: 10
      delay: 5

# Benefits:
# - Single tool for container deployment
# - Integrated with infrastructure
# - Consistent playbooks across platforms
```

### Pattern 2: Disaster Recovery Automation

```yaml
---
- name: Automated Disaster Recovery
  hosts: backup_server
  
  tasks:
    # Monitor primary site
    - name: Health check primary site
      uri:
        url: https://primary.example.com/health
        status_code: 200
      register: primary_health
      ignore_errors: yes
    
    # If primary down, failover to backup
    - block:
        - name: Alert: Primary site down!
          slack:
            token: "{{ slack_token }}"
            msg: "PRIMARY SITE DOWN - Initiating failover"
        
        - name: Update DNS to point to backup
          shell: |
            aws route53 change-resource-record-sets \
              --hosted-zone-id Z123 \
              --change-batch file:///tmp/failover.json
        
        - name: Activate backup services
          service:
            name: "{{ item }}"
            state: started
          loop:
            - nginx
            - mysql
            - redis
        
        - name: Restore from latest backup
          shell: |
            mysql < /backups/latest.sql
        
        - name: Verify backup site
          uri:
            url: https://backup.example.com/health
            status_code: 200
          retries: 10
          delay: 5
        
        - name: Notify team
          mail:
            host: smtp.example.com
            to: ops@example.com
            subject: "FAILOVER COMPLETE: Backup site active"
      
      when: primary_health is failed

# Automated failover: Minutes to recover!
```

---

## Conclusion

Production Ansible at scale requires:

1. **Infrastructure as Code**: Everything in Git
2. **Automation**: CI/CD pipelines, approval gates
3. **Safety**: Dry-runs, testing, serial deployments
4. **Reliability**: Idempotency, error handling, recovery
5. **Observability**: Logging, metrics, alerting
6. **Scalability**: Hundreds to thousands of servers

Master production Ansible and you can:
- ✅ Deploy infrastructure changes safely
- ✅ Recover from incidents automatically
- ✅ Scale to enterprise environments
- ✅ Maintain audit trails for compliance
- ✅ Integrate with orchestration platforms

**Key principle**: Production Ansible is not just automation - it's disaster prevention, recovery automation, and operational excellence combined!
