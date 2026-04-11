# Deployment Strategies: Blue-Green, Canary, Rolling, & Shadow Deployments

## 1. Core Concept

Deployment strategy defines **how** code moves from "ready to deploy" to "running in production":

1. **Blue-Green**: Two full environments, switch traffic
2. **Canary**: Gradual rollout to small percentage
3. **Rolling**: Replace servers one at a time
4. **Shadow**: Run new version alongside old (no traffic)
5. **Feature Flags**: Gradual feature enablement
6. **Immutable Infrastructure**: Never modify, replace

### Why Strategy Matters

```
Wrong strategy = disaster during deployment:
├─ Downtime (users can't access app)
├─ Data loss (incomplete transactions)
├─ Performance degradation
├─ No quick recovery
└─ Team stress!

Right strategy = safe deployment:
├─ Zero downtime
├─ Instant rollback if issues
├─ User-facing gradual rollout
├─ Metrics-based decisions
└─ Team confidence
```

---

## 2. Pipeline Architecture for Deployments

### Pre-Deployment Validation

```
Deployment triggers
    ↓
Sanity checks:
├─ Image exists in registry?
├─ Database migrations safe?
├─ Secrets available?
├─ Target infrastructure ready?
└─ Rollback plan exists?
    ↓
Health check current version
    ↓
Begin deployment strategy
```

### Generic Deployment Flow

```yaml
---
- name: Execute deployment strategy
  hosts: localhost
  
  vars:
    deployment_strategy: "blue-green"  # or canary, rolling, etc
    deployment_timeout: 5m
    health_check_retries: 30
    success_criteria:
      error_rate_threshold: 5  # %
      latency_p95_threshold: 500  # ms
      error_logs_increase: 50  # %
  
  pre_tasks:
    - name: Pre-deployment safety checks
      block:
        - name: Verify new image exists
          shell: docker inspect {{ new_image }}
        
        - name: Validate database state
          shell: |
            mysqlcheck -u{{ db_user }} -p{{ db_pass }} {{ db_name }}
        
        - name: Health check current version
          uri:
            url: https://prod.example.com/health
            status_code: 200
          register: pre_health
        
        - name: Capture baseline metrics
          shell: |
            curl -s https://metrics.example.com/baseline > /tmp/baseline.json
  
  tasks:
    - block:
        - name: Deploy using {{ deployment_strategy }}
          include_tasks: "deploy_{{ deployment_strategy }}.yml"
        
        - name: Health checks post-deployment
          uri:
            url: https://prod.example.com/health
            status_code: 200
          retries: "{{ health_check_retries }}"
          delay: 10
      
      rescue:
        - name: Deployment failed!
          debug:
            msg: "Triggering rollback..."
        
        - name: Rollback to previous version
          include_tasks: rollback.yml
        
        - name: Alert team
          slack:
            token: "{{ slack_token }}"
            msg: "🚨 Deployment FAILED and rolled back"
        
        - name: Fail the deployment
          fail:
            msg: "Deployment failed"
```

---

## 3. Real-World Deployment Strategies

### Strategy 1: Blue-Green Deployment (Instant Switch)

```
Architecture:
┌──────────────────────────────────────┐
│ Load Balancer                        │
│ ├─ Route 100% to BLUE                │
│ └─ Route 0% to GREEN                 │
├──────────────┬───────────────────────┤
│ BLUE         │ GREEN                 │
│ (v1.0)       │ (v1.1 - deploying)    │
│ ✅ Active    │ ⏳ Staging            │
└──────────────┴───────────────────────┘

Deployment Flow:
1. Deploy v1.1 to GREEN
2. Run tests on GREEN
3. Switch LB: 100% → GREEN
4. Keep BLUE for instant rollback
5. After stability (hours), cleanup BLUE

Pros:
✅ Instant switch (no downtime)
✅ Instant rollback (revert LB setting)
✅ Complete environment isolation
✅ Run tests before switching

Cons:
❌ Requires 2x resources (expensive!)
❌ Database schema changes tricky
❌ Config management complexity
```

#### Blue-Green Implementation

```yaml
---
- name: Blue-Green Deployment
  hosts: localhost
  gather_facts: no
  
  vars:
    current_time: "{{ ansible_date_time.iso8601_basic_short }}"
  
  tasks:
    # Determine which is active
    - name: Get current active environment
      shell: |
        aws elbv2 describe-target-groups \
          --names myapp-active \
          --query 'TargetGroups[0].TargetType' \
          -o text
      register: current_active
    
    - name: Determine inactive environment
      set_fact:
        active_env: "{{ current_active.stdout }}"
        inactive_env: "{{ 'green' if current_active.stdout == 'blue' else 'blue' }}"
    
    # Deploy to inactive environment
    - name: "Deploy to {{ inactive_env }}"
      block:
        - name: Update deployment image
          kubernetes.core.k8s:
            state: present
            definition:
              apiVersion: apps/v1
              kind: Deployment
              metadata:
                name: "myapp-{{ inactive_env }}"
                namespace: production
              spec:
                replicas: 3
                template:
                  spec:
                    containers:
                      - name: app
                        image: "{{ new_image }}"
        
        - name: Wait for deployment ready
          kubernetes.core.k8s_info:
            kind: Deployment
            name: "myapp-{{ inactive_env }}"
            namespace: production
            wait: yes
            wait_condition:
              type: Available
              status: True
            wait_sleep: 10
            wait_timeout: 300
    
    # Health checks on new environment
    - name: Run health checks on {{ inactive_env }}
      block:
        - name: Get service endpoint
          kubernetes.core.k8s_info:
            kind: Service
            name: "myapp-{{ inactive_env }}"
            namespace: production
          register: service_info
        
        - name: Health check
          uri:
            url: "http://{{ service_info.resources[0].status.loadBalancer.ingress[0].ip }}/health"
            status_code: 200
          retries: 30
          delay: 5
          register: health_result
      
      rescue:
        - name: Rollback deployment
          kubernetes.core.k8s:
            kind: Deployment
            name: "myapp-{{ inactive_env }}"
            namespace: production
            state: absent
        
        - fail:
            msg: "Health checks failed, rolled back {{ inactive_env }}"
    
    # Run smoke tests
    - name: Smoke tests
      shell: |
        BASE_URL=http://{{ service_info.resources[0].status.loadBalancer.ingress[0].ip }}
        ./scripts/smoke-tests.sh "$BASE_URL"
      register: smoke_tests
    
    # Switch traffic
    - name: "Switch traffic from {{ active_env }} to {{ inactive_env }}"
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Service
          metadata:
            name: myapp
            namespace: production
          spec:
            selector:
              app: myapp
              version: "{{ inactive_env }}"  # ← Switch happens here!
            ports:
              - port: 80
                targetPort: 8080
      register: traffic_switch
    
    # Monitor new environment
    - name: "Monitor {{ inactive_env }} for 5 mins"
      block:
        - name: Check error rate
          shell: |
            curl -s https://metrics.example.com/error_rate?pod={{ inactive_env }} \
              | jq '.current_rate'
          register: error_rate
          until: error_rate.stdout | float < 1.0
          retries: 30
          delay: 10
        
        - name: Check latency
          shell: |
            curl -s https://metrics.example.com/latency_p95?pod={{ inactive_env }} \
              | jq '.current_p95'
          register: latency
          until: latency.stdout | float < 500
          retries: 30
          delay: 10
      
      rescue:
        - name: Revert traffic!
          kubernetes.core.k8s:
            state: present
            definition:
              apiVersion: v1
              kind: Service
              metadata:
                name: myapp
                namespace: production
              spec:
                selector:
                  version: "{{ active_env }}"  # ← Switch back!
        
        - fail:
            msg: "Monitoring detected issues, reverted to {{ active_env }}"
    
    # Cleanup old environment
    - name: "Mark {{ active_env }} for cleanup"
      debug:
        msg: "{{ active_env }} can be cleaned up after stabilization period"
```

### Strategy 2: Canary Deployment (Gradual Rollout)

```
Architecture:
┌──────────────────────────────────────┐
│ Load Balancer                        │
│ ├─ Route 95% to STABLE (v1.0)        │
│ └─ Route 5% to CANARY (v1.1)         │
├──────────────┬───────────────────────┤
│ STABLE (95%) │ CANARY (5%)           │
│ (v1.0)       │ (v1.1)                │
│ ✅ 19/20     │ ⏳ 1/20 servers       │
└──────────────┴───────────────────────┘

Timeline:
T+0min: Deploy to 1 server (5%)
        Monitor 5 minutes
        ├─ Error rate ok?
        ├─ Latency ok?
        └─ Logs show issues?

T+5min: If good, expand to 25% (5 servers)
T+10min: If good, expand to 50% (10 servers)
T+15min: If good, expand to 100% (all servers)

If problems detected at any stage:
├─ Revert canary servers instantly
├─ Keep at previous percentage
└─ Alert team

Pros:
✅ Gradual risk exposure
✅ Real traffic (vs synthetic tests)
✅ Only partial blast radius if bad
✅ Easy to revert (just shrink canary)

Cons:
❌ Slower (manual monitoring needed)
❌ Complexity (multiple versions running)
❌ Database schema changes tricky
```

#### Canary Implementation

```yaml
---
- name: Canary Deployment
  hosts: localhost
  gather_facts: no
  
  vars:
    canary_stages:
      - percentage: 5
        duration: 5
        error_threshold: 2
        latency_threshold: 600
      - percentage: 25
        duration: 5
        error_threshold: 2
        latency_threshold: 550
      - percentage: 50
        duration: 5
        error_threshold: 1.5
        latency_threshold: 500
      - percentage: 100
        duration: 0
  
  tasks:
    - name: Start canary deployment
      block:
        - name: Deploy new version to canary servers
          kubernetes.core.k8s:
            state: present
            definition:
              apiVersion: apps/v1
              kind: Deployment
              metadata:
                name: myapp-canary
                namespace: production
              spec:
                replicas: 0  # Start with 0
                template:
                  spec:
                    containers:
                      - name: app
                        image: "{{ new_image }}"
        
        - name: Canary rollout progression
          block:
            - name: "Stage {{ item.percentage }}%"
              block:
                # Calculate replicas needed for this percentage
                - name: Get total replicas
                  kubernetes.core.k8s_info:
                    kind: Deployment
                    name: myapp-stable
                    namespace: production
                  register: stable_deployment
                
                - name: Calculate canary replicas
                  set_fact:
                    total_replicas: "{{ stable_deployment.resources[0].spec.replicas }}"
                    canary_replicas: "{{ ((stable_deployment.resources[0].spec.replicas * item.percentage / 100) | int) }}"
                
                - name: Update canary replicas to {{ canary_replicas }}
                  kubernetes.core.k8s:
                    state: present
                    definition:
                      apiVersion: apps/v1
                      kind: Deployment
                      metadata:
                        name: myapp-canary
                        namespace: production
                      spec:
                        replicas: "{{ canary_replicas }}"
                
                - name: Wait for canary pods ready
                  kubernetes.core.k8s_info:
                    kind: Deployment
                    name: myapp-canary
                    namespace: production
                    wait: yes
                    wait_condition:
                      type: Available
                  wait_sleep: 10
                  wait_timeout: 300
                
                # Update traffic routing (canary ingress controller)
                - name: Update traffic split to {{ item.percentage }}%
                  kubernetes.core.k8s:
                    state: present
                    definition:
                      apiVersion: flagger.app/v1beta1
                      kind: Canary
                      metadata:
                        name: myapp
                        namespace: production
                      spec:
                        targetRef:
                          name: myapp
                        progressDeadlineSeconds: 60
                        service:
                          port: 80
                          targetPort: 8080
                        analysis:
                          interval: 1m
                          threshold: 5
                          maxWeight: "{{ item.percentage }}"
                          stepWeight: "{{ item.percentage }}"
                          metrics:
                            - name: error_rate
                              thresholdRange:
                                max: "{{ item.error_threshold }}"
                            - name: latency
                              thresholdRange:
                                max: "{{ item.latency_threshold }}"
                
                # Wait for duration
                - name: "Monitor for {{ item.duration }} minutes"
                  shell: sleep {{ item.duration * 60 }}
                
                # Collect metrics
                - name: Collect metrics
                  shell: |
                    curl -s https://metrics.example.com/metrics \
                      --data-urlencode "pod=canary" \
                      > /tmp/metrics.json
                  register: metrics
                
                - name: Analyze metrics
                  set_fact:
                    error_rate: "{{ (metrics.stdout | from_json).error_rate }}"
                    latency: "{{ (metrics.stdout | from_json).latency_p95 }}"
                
                # Check thresholds
                - name: Validate metrics
                  assert:
                    that:
                      - error_rate | float < item.error_threshold
                      - latency | float < item.latency_threshold
                    fail_msg: "Metrics exceed threshold - stopping rollout!"
              
              rescue:
                - name: "Rollback: Keep at previous percentage"
                  debug:
                    msg: "Metrics failed, reverting canary replicas"
                
                - name: Rollback canary deployment
                  shell: |
                    kubectl delete deployment myapp-canary -n production
                
                - fail:
                    msg: "Canary deployment stopped at {{ (item | first).percentage | default('0') }}%"
            
            loop: "{{ canary_stages }}"
```

### Strategy 3: Rolling Deployment (Replace Gradually)

```
Architecture:
Servers: [S1, S2, S3, S4, S5]

T+0min: [v1.0, v1.0, v1.0, v1.0, v1.0]
        Deploy S1 → v1.1
        Load balancer removes S1
        Deploy S1 → v1.1

T+2min: [v1.1, v1.0, v1.0, v1.0, v1.0]
        Deploy S2 → v1.1
        Load balancer removes S2
        Deploy S2 → v1.1

T+4min: [v1.1, v1.1, v1.0, v1.0, v1.0]
        Continue...

T+10min: [v1.1, v1.1, v1.1, v1.1, v1.1]
         Complete!

Pros:
✅ Progressive upgrade
✅ No extra resources needed
✅ Easy to rollback (switch LB back)
✅ Natural load balancing

Cons:
❌ Downtime for each replica
❌ Slower than blue-green
❌ Can't keep old version running
❌ Schema changes cause issues
```

#### Rolling Implementation

```yaml
---
- name: Rolling Deployment
  hosts: localhost
  
  tasks:
    - name: Rolling update deployment
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: myapp
            namespace: production
          spec:
            replicas: 5
            strategy:
              type: RollingUpdate
              rollingUpdate:
                maxSurge: 1        # 1 extra pod during update
                maxUnavailable: 0  # 0 down at any time (safe!)
            template:
              spec:
                containers:
                  - name: app
                    image: "{{ new_image }}"
      
      # Kubernetes handles the rolling update!
      register: deployment
    
    - name: Wait for rollout complete
      kubernetes.core.k8s_info:
        kind: Deployment
        name: myapp
        namespace: production
        wait: yes
        wait_condition:
          type: Available
          status: True
      wait_timeout: 600
```

### Strategy 4: Shadow Deployment (Dark Launch)

```
Purpose: Test new version with real traffic without affecting users

Architecture:
┌──────────────────────────────────────┐
│ Ingress / Traffic Mirror             │
├──────────────┬───────────────────────┤
│              │                       │
▼              ▼                       ▼
Users ──%→ STABLE            SHADOW
(100%)  (v1.0 responses)  (v1.1 - no responses)

Traffic is duplicated to shadow - users never see shadow responses!

Implementation:
1. All traffic goes to v1.0 (users see responses)
2. Copy all requests to v1.1 (shadow, responses discarded)
3. Compare responses:
   ├─ Same result? Good!
   ├─ Different? Bug in v1.1
   ├─ Slower? Performance issue
   └─ Error? Exception found
4. Once confident, switch to canary/blue-green
```

#### Shadow Implementation

```yaml
---
- name: Shadow Deployment
  hosts: localhost
  
  tasks:
    # Deploy shadow version (no replicas), no traffic routed
    - name: Deploy shadow version
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: myapp-shadow
            namespace: production
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: myapp
                version: shadow
            template:
              spec:
                containers:
                  - name: app
                    image: "{{ new_image }}"
                    ports:
                      - containerPort: 8080
    
    # Set up traffic mirroring
    - name: Configure traffic mirroring
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: networking.istio.io/v1beta1
          kind: VirtualService
          metadata:
            name: myapp
            namespace: production
          spec:
            hosts:
              - myapp.example.com
            http:
              - match:
                  - uri:
                      prefix: "/"
                route:
                  - destination:
                      host: myapp-stable
                      port:
                        number: 8080
                    weight: 100
                mirror:
                  host: myapp-shadow
                  port:
                    number: 8080
                mirrorPercent: 100  # Mirror 100% of traffic
                timeout: 5s
    
    # Compare responses
    - name: Compare shadow vs stable responses
      block:
        - name: Get sample requests and responses
          shell: |
            # Capture requests from logs
            kubectl logs -l app=myapp,version=stable -n production | head -100 > /tmp/stable.log
            kubectl logs -l app=myapp,version=shadow -n production | head -100 > /tmp/shadow.log
        
        - name: Compare results
          shell: |
            diff -u /tmp/stable.log /tmp/shadow.log | \
              grep -c "^[<>]" || echo "0"
          register: diff_count
        
        - name: Report differences
          debug:
            msg: "Found {{ diff_count.stdout }} differences between stable and shadow"
    
    # Once confident, remove shadow
    - name: Remove shadow deployment
      kubernetes.core.k8s:
        state: absent
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: myapp-shadow
            namespace: production
      when: confidence_level | default('low') == 'high'
```

---

## 4. Common Mistakes

### Mistake 1: No Health Checks During Deployment

```yaml
# ❌ WRONG
- name: Deploy new version
  kubernetes.core.k8s:
    state: present
    definition: ...

# Doesn't wait for pods to be ready!
# Traffic routed before app starts listening

# ✅ CORRECT
- name: Deploy new version
  kubernetes.core.k8s:
    state: present
    definition: ...

- name: Wait for deployment ready
  kubernetes.core.k8s_info:
    kind: Deployment
    name: myapp
    wait: yes
    wait_condition:
      type: Available
      status: True
    wait_timeout: 300
```

### Mistake 2: Database Migrations + Deployment Race Condition

```sql
-- Problem scenario:
-- v1.0 expects: users.age (INT)
-- v1.1 expects: users.birth_date (DATETIME)
-- Migration: ALTER TABLE users RENAME COLUMN age TO birth_date
-- 
-- During deployment:
-- T+0: Start deploying v1.1 to Pod 1
-- T+1: Pod 1 starts, expects birth_date
-- T+2: Migration runs (too late!)
-- T+3: Pod 1 crashes (column not found!)

# ✅ CORRECT (migrations before deployment)
jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - name: Run migrations
        run: python manage.py migrate
  
  deploy:
    needs: migrate  # ← Wait for migrations!
    runs-on: ubuntu-latest
    steps:
      - name: Deploy new code
        run: kubectl set image deployment/app ...
```

### Mistake 3: No Rollback Plan

```bash
# ❌ WRONG
$ kubectl set image deployment/myapp app=newimage
# Deployed! But how do we rollback if broken?
# $ kubectl set image deployment/myapp app=...what was old image?

# ✅ CORRECT
# Store previous state
$ kubectl rollout history deployment/myapp
  revision 1: <old-image>
  revision 2: <new-image>

# Rollback instantly
$ kubectl rollout undo deployment/myapp
# Back to revision 1!
```

---

## 5. Debugging Deployment Issues

### Deployment Stuck in "Pending"

```bash
# Signs:
# - Pods won't start
# - "Awaiting node resources" message

# Debug:
$ kubectl describe pod myapp-xxxxx
# Shows: "Insufficient cpu", "Insufficient memory"

# Fix:
# 1. Reduce replicas
# 2. Reduce resource requests
# 3. Add more nodes
# 4. Update deployment strategy max_surge/max_unavailable
```

### Canary Metrics Show False Positives

```bash
# Problem:
# Canary looks healthy, but doesn't scale up
# Metrics show errors, but app works

# Debugging:
# 1. Check metrics collection
# 2. Verify Prometheus queries
# 3. Check for baseline comparison issues
# 4. Verify time windows match

# Solution:
# Define clear thresholds:
canary_stages:
  - percentage: 5
    error_threshold: 2  # % vs baseline
    latency_threshold: 500  # ms
```

---

## 6. Interview Questions

### **Q1**: You deploy a canary and metrics show latency spike, but user reports say app is fast. What's happening?

**Answer**:
```
Possible causes:

1. Metrics lag: Metrics collected 5 mins ago
   - Solution: Use real-time monitoring (Datadog, CloudWatch)

2. Canary metrics vs overall metrics
   - Metrics show only canary (which is cold/warming up)
   - Overall app still fast
   - Solution: Compare canary vs stable app metrics

3. Synthetic vs real traffic
   - Health checks run every 10 seconds
   - Real traffic has spikes
   - Solution: Use actual user metrics (RUM)

4. Baseline is wrong
   - Baseline from low-traffic period
   - Current traffic higher
   - Solution: Use dynamic baseline

Best fix:
- Compare canary latency vs stable app latency
- Use real user monitoring
- Set reasonable thresholds (not too strict)
- Manual review before auto-rollback
```

---

## 7. Deployment Checklist

```yaml
Before each deployment:

Pre-Deployment:
├─ [ ] Code reviewed and merged
├─ [ ] All tests passing
├─ [ ] No open security vulnerabilities
├─ [ ] Database migrations tested
├─ [ ] Rollback plan documented
├─ [ ] Slack notifications configured
└─ [ ] On-call engineer aware

Deployment:
├─ [ ] Monitoring dashboards open
├─ [ ] Baseline metrics captured
├─ [ ] Deploy to staging first
├─ [ ] Run staging smoke tests
├─ [ ] Get approval for production
├─ [ ] Start deployment
├─ [ ] Wait for health checks
└─ [ ] Verify metrics

Post-Deployment:
├─ [ ] Error rate < 1%
├─ [ ] Latency P95 < 500ms
├─ [ ] Memory usage normal
├─ [ ] CPU usage normal
├─ [ ] No increase in exceptions
├─ [ ] User reports positive
└─ [ ] Update release notes
```

---

## 8. Advanced Insights

### GitOps-Driven Deployments

```yaml
# Instead of manual kubectl commands:
# $ kubectl set image deployment/app ...

# Use GitOps:
# 1. Developer opens PR
# 2. CI validates
# 3. PR merged to main
# 4. CD controller (ArgoCD, Flux) detects change
# 5. CD controller updates cluster
# 6. Cluster state syncs with Git

# Benefits:
# - Audit trail (Git commits)
# - Easy rollback (git revert)
# - Disaster recovery (reapply Git state)
# - Team visibility (see PRs for changes)
```

---

## Conclusion

Deployment strategies are **critical for production**:

✅ **Blue-Green**: Instant switch, instant rollback  
✅ **Canary**: Gradual risk exposure  
✅ **Rolling**: No extra resources  
✅ **Shadow**: Test without risk  

Choose based on your:
- Budget (resources available?)
- Risk tolerance (can handle brief outage?)
- Deployment frequency (daily? monthly?)
- Team size (dedicated DevOps?)

Master deployment strategies and you enable:
- ✅ Zero-downtime deployments
- ✅ Instant rollbacks
- ✅ Safe experimentation
- ✅ Team confidence
- ✅ Production reliability
