# Kubernetes Troubleshooting & Operations: Deep Dive

## 1. Core Concept (Deep Internal Working)

Kubernetes troubleshooting requires understanding the **stack of abstraction layers**:

```
Layer 1: Application (Your code)
  └─ Logs, debugging, profiling
  
Layer 2: Pod (Container environment)
  └─ Resource limits, security policies, probes
  
Layer 3: Node (Host OS)
  └─ Disk, network, kubelet status
  
Layer 4: Cluster (API Server, etcd, networking)
  └─ API errors, scheduling, network policies
```

Each layer can cause failures:
```
App pod fails   → Issue in Layer 1 or 2
Pod won't start → Issue in Layer 2 or 3
Can't schedule  → Issue in Layer 3 or 4
Network broken  → Issue in Layer 3 or 4
```

### Debugging Methodology

**STOP-LOOK-LISTEN Model**:
```
STOP: Gather facts (don't guess)
LOOK: Examine symptoms systematically
LISTEN: Check logs at each layer
```

**Troubleshooting Tree**:
```
Is pod running?
├─ NO (Pending) → Scheduler issue (Layer 4)
│  ├─ Resources insufficient
│  ├─ Taints/tolerations mismatch
│  ├─ Affinity rules too strict
│  └─ Network policies blocking
│
├─ NO (ImagePullBackOff) → Image issue (Layer 2)
│  ├─ Image doesn't exist
│  ├─ Registry auth failed
│  └─ Image format corrupt
│
├─ NO (CrashLoopBackOff) → App crashed (Layer 1 or 2)
│  ├─ Check app logs
│  ├─ Check readiness/liveness probes
│  └─ Check SecurityContext
│
└─ YES (Running)
   ├─ Can app reach dependencies?
   │  ├─ Check network policies
   │  ├─ Check DNS resolution
   │  └─ Check service endpoints
   │
   ├─ Is performance degraded?
   │  ├─ Check resource limits
   │  ├─ Check node capacity
   │  └─ Check latency metrics
   │
   └─ Are errors in logs?
      ├─ Parse error messages
      ├─ Check dependencies
      └─ Check configuration
```

---

## 2. How Kubernetes Actually Works Internally

### Observability: The Four Pillars

**1. Logs (Application output)**
```
kubectl logs pod-name
└─ Shows stdout/stderr from pod
└─ Most direct information
└─ Requires app to log properly

kubectl logs --previous pod-name
└─ Shows logs from previous pod (if crashed and restarted)

kubectl logs -l app=api --tail=100 -f
└─ Stream logs from all pods matching label
└─ Follow mode: stream new logs
```

**2. Metrics (Quantitative measurements)**
```
kubectl top pods
└─ CPU usage and memory usage
└─ Requires Metrics Server (installed by default)
└─ Updates every 15-60 seconds
└─ P99 not available (only instantaneous)

Prometheus (more detailed):
├─ Scrapes /metrics endpoint (Prometheus format)
├─ Stores time-series data
├─ Query language: PromQL
└─ Allows historical analysis

Example metrics to monitor:
├─ container_cpu_usage_seconds_total
├─ container_memory_working_set_bytes
├─ http_request_duration_seconds
└─ pod_network_receive_bytes_total
```

**3. Events (What happened)**
```
kubectl get events
└─ Ordered log of cluster actions
└─ E.g., "Pod created", "Image pulled", "Pod scheduled"
└─ Most recent first

kubectl describe pod pod-name | grep -A 20 Events
└─ Pod-specific events
└─ Shows exact timestamp and message

Events are ephemeral:
└─ Deleted after 1 hour (default)
└─ Large clusters: Events truncated

Best practice: Forward events to external system (Splunk, ELK)
```

**4. Traces (Request flow)**
```
Distributed tracing: Follow single request through system

Example: User request to microservice arch

Request starts:
  User → Public Internet
  │
  ├─ Ingress controller receives
  │  └─ Span: "ingress_routing" (2ms)
  │
  ├─ Sends to API Service
  │  └─ Span: "api_handler" (45ms)
  │
  ├─ API Service calls User Service
  │  └─ Span: "user_service_call" (20ms)
  │
  ├─ User Service queries Database
  │  └─ Span: "db_query" (30ms)
  │
  └─ Response back through same path
     └─ Total: 97ms

Tools:
├─ Jaeger (open-source)
├─ Zipkin (open-source)
├─ AWS X-Ray (proprietary)
└─ Datadog APM (SaaS)

Requires:
├─ App instrumentation (add tracing SDK)
├─ Trace collection backend
├─ Query UI
```

### Debugging Network Issues

**Network stack troubleshooting**:
```
Client → Service IP → kube-proxy → Pod endpoint

At each step, test connectivity:

Step 1: Client can resolve DNS?
$ nslookup service-name
service-name.default.svc.cluster.local has address 10.1.0.100

Step 2: Client can reach service IP?
$ telnet 10.1.0.100 8080
Connected (or Connection refused)

Step 3: Is service endpoint populated?
$ kubectl get endpoints service-name
NAME            ENDPOINTS
service-name    10.0.2.1:8080,10.0.2.2:8080  (good)
service-name    <none>                        (bad, no endpoints)

Step 4: Can kube-proxy reach pod directly?
$ kubectl exec -it kube-proxy-pod -- \
  telnet 10.0.2.1 8080

Step 5: Pod firewall rules (iptables)?
$ kubectl exec -it pod-name -- \
  iptables -L -n
(usually app can't see these, but node can)

Step 6: CNI routing working?
$ ip route show
(on node)
10.0.0.0/24 dev cni0 proto kernel scope link

Step 7: Physical network reachability?
$ traceroute 10.0.2.1
```

### Debugging Pod Creation Failures

**Timeline of pod startup**:
```
T0: kubectl apply
  └─ YAML sent to API Server

T0+100ms: API Server validates + stores in etcd
  └─ Pod.phase = Pending

T0+200ms: Scheduler picks node
  └─ Pod.spec.nodeName = worker-1

T0+300ms: kubelet on worker-1 sees pod assignment
  └─ Calls container runtime: create container

T0+400ms: Container runtime pulls image
  └─ Contacts registry (could take seconds)
  └─ Stores layers in /var/lib/containers

T0+2000ms: Image ready, container created
  └─ Container process starts
  └─ kubelet sets pod.phase = Running

T0+2100ms: Startup probe runs (if defined)
  └─ Checks: Is app initializing?
  └─ No wait needed if not defined

T0+2500ms: Readiness probe runs
  └─ Checks: Is app accepting traffic?
  └─ If fails: Stays Running but not Ready

T0+3000ms: Pod becomes Ready (or NotReady if probe failed)
  └─ Added to service endpoints (if selector matches)
  └─ Traffic can now reach pod
```

**Failure scenarios at each stage**:
```
50ms: Pod stays Pending forever
  └─ kubelet never sees assignment
  └─ Check: API Server working? Webhook blocking?
  └─ Check: kubelet logs on node

500ms: ImagePullBackOff
  └─ Image download failed
  └─ Check: Image exists in registry?
  └─ Check: Registry auth (imagePullSecrets)?
  └─ Check: Disk space on node (pull destination)?

1000ms: CrashLoopBackOff immediately
  └─ Container exits < 1 second
  └─ Check: App binary present?
  └─ Check: Entry point correct (command/args)?
  └─ Check: Logs: kubectl logs --previous pod

2000ms: Running but not Ready
  └─ Readiness probe consistently fails
  └─ Check: app logs for startup errors
  └─ Check: dependencies available (DB, config)?
  └─ Check: HealthCheck endpoint responding?
```

---

## 3. When to Use vs Not Use (Anti-patterns)

### Anti-Pattern 1: Troubleshooting Without Logs

```yaml
# ❌ WRONG
Error: "Pod not starting"
(No investigation into why)
Action: Delete and recreate pod
Result: Still fails, time wasted

# ✅ RIGHT
Error: "Pod not starting"
$ kubectl describe pod stuck-pod
$ kubectl logs stuck-pod --previous
Logs: "Error: Database connection refused"
Action: Check if database pod is running
$ kubectl get pods -l app=database
Result: 0 replicas (database pod down)
Action: Scale up database deployment
Result: Fixed!
```

### Anti-Pattern 2: Relying on kubectl top for Performance Analysis

```yaml
# ❌ WRONG
$ kubectl top pods
# Shows instant CPU usage (could be 5 seconds old)
# Makes decision based on single data point
Decision: "Pod using 800m CPU, why so high?"
Action: Increase limits
Result: Might be a spike, limits increase, resources wasted

# ✅ RIGHT
Use Prometheus:
  query: max_over_time(container_cpu_usage_seconds_total[5m])
  └─ 5-minute rolling window
  └─ See peak usage
  └─ Make informed decisions
```

### Anti-Pattern 3: Using Exec for Debugging Temporary Issues

```yaml
# ❌ WRONG (Pod will restart)
$ kubectl exec -it pod -- bash
# Manually investigate issue
# Make temporary fix
# Pod restarts (due to probe failure)
# Lost all manual work

# ✅ RIGHT (Debug while pod active)
1. Save pod spec to file:
   $ kubectl get pod pod-name -o yaml > pod.yaml

2. Add debugging container:
   spec:
     containers:
     - name: debugger
       image: busybox
       command: ["sleep", "3600"]
       volumeMounts:
       - name: shared
         mountPath: /shared
     - name: app
       volumeMounts:
       - name: shared
         mountPath: /shared

3. Exec into debugger container
   $ kubectl exec -it pod-name -c debugger -- bash
   # Share files with app via /shared volume

4. Debug non-destructively
```

### Anti-Pattern 4: Not Using PodTemplates for Debugging

```yaml
# ❌ WRONG (Lost when pod deletes)
$ kubectl exec -it pod -- sh
# Fix issue manually
# Pod crashes and restarts
# Fix lost

# ✅ RIGHT (Configuration as source of truth)
1. Update Deployment template:
   spec:
     template:
       spec:
         containers:
         - env:
           - name: DEBUG  # Enable debug logging
             value: "true"

2. Redeploy:
   $ kubectl apply -f deployment.yaml

3. New pods have fix
4. Fix persists across restarts
```

---

## 4. Real-World Production Scenario

### Scenario: Database Connection Pool Exhaustion

**Timeline**

```
Friday 14:00: Normal operations
  API pods: 20
  DB connections per pod: 10
  Total connections: 200 (pool max: 300, headroom: 100)

Friday 14:30: Traffic increase (9% per minute)
  API pods: 20 (HPA hasn't kicked in yet)
  DB connections per pod: 10
  Total: 200 (still OK)

Friday 14:35: HPA triggers scale-up
  API pods: 25
  DB connections: 25 × 10 = 250 (still OK, pool max 300)

Friday 14:40: Traffic continues increasing
  API pods: 35 (HPA scales more aggressively)
  DB connections: 35 × 10 = 350
  
  ⚠️ EXCEEDS POOL LIMIT (350 > 300)!
  
  Result: Connection refused errors
  Error rate: 16% (50 connections refused out of 300)
  API response time: Increases (retries, fallbacks)

Friday 14:42: Cascading failures
  - Some requests fail immediately
  - Requests retry (exponential backoff)
  - Retries create more connection attempts
  - More failures
  - p99 latency: 5000ms (was 100ms)
  - Error rate: 45%

Friday 14:45: On-call engineer alerted
  Alert: "Error rate > 40%"
  
  Starts troubleshooting:
  1. Check API pod logs
     Error: "Connection refused"
  
  2. Check database pod logs
     Error: "Too many open connections"
  
  3. Check HPA status
     $ kubectl get hpa
     API replicas: 45 (scaled too high)
  
  4. Check DB connection pool
     Current: 450 connections (pool max 300)
  
  5. Scale back API pods
     $ kubectl scale deployment api --replicas=20
  
  6. Wait for pods to drain
     3 minutes: All connections return
  
  7. Error rate drops: 45% → 0%
     Latency: 5000ms → 100ms (back to normal)
```

**Root Cause Analysis**

```
Why did this happen?

1. ❌ HPA only looks at CPU/memory
   └─ Didn't consider database connection pool limit
   
2. ❌ No monitoring on DB connection pool usage
   └─ Couldn't predict exhaustion
   
3. ❌ No circuit breaker in application
   └─ Retries made cascade worse
   
4. ❌ Database didn't auto-scale
   └─ Fixed pool size (manual scaling required)

5. ❌ No burst capacity in pool
   └─ Immediate rejection when full

Solution implemented:

1. Add custom metric to HPA
   $ kubectl set metric hpa api-hpa --metric db_connection_pool_usage
   └─ Scale API pods based on connection pool %
   └─ Before hitting limit
   
2. Increase DB connection pool
   config: max_connections = 500
   └─ Increased headroom
   
3. Add circuit breaker to app
   $ app config: circuit_breaker.threshold = 100 failures per min
   └─ Fail fast, don't retry
   └─ Prevents cascade
   
4. Add alerting
   $ alert: db_connection_pool_usage > 80%
   └─ Alert before problem happens
   
5. Database read replicas
   └─ Distribute load
   └─ More capacity
   
6. Connection pooling proxy
   $ pgBouncer sits between app and database
   └─ Multiplexes many app connections to fewer DB connections
   └─ 100 app connections → 10 DB connections
```

---

## 5. Architecture Diagrams (Text-Based)

### Troubleshooting Decision Tree

```
┌──────────────────────────────────────────────────┐
│         IS THE POD RUNNING?                      │
│  kubectl get pods / kubectl describe pod        │
└──────────────────────────────────────────────────┘

Branch 1: Status = PENDING (Waiting to start)
├─ kubectl get events | grep pod-name
│  └─ Look for "FailedScheduling"
│  
├─ If FailedScheduling:
│  ├─ Reason: "Insufficient cpu" or "Insufficient memory"
│  │  └─ Check: kubectl describe nodes | grep "Allocated"
│  │  └─ Check: Do you have free resources?
│  │  └─ If no: Add nodes or reduce requests
│  │
│  ├─ Reason: "node(s) didn't match pod affinity"
│  │  └─ Check: Pod spec affinity rules
│  │  └─ Check: Available nodes matching rules
│  │  └─ If none: Adjust affinity or add matching nodes
│  │
│  ├─ Reason: "node(s) had taints"
│  │  └─ Check: kubectl describe nodes | grep Taints
│  │  └─ Check: Pod tolerations
│  │  └─ If missing: Add tolerations or remove taints
│
└─ If no events shown:
   └─ Pod stuck in admissions
   └─ Check: kubectl describe pod | grep "Admission"
   └─ May need to restart API Server

Branch 2: Status = ImagePullBackOff (Can't pull image)
├─ Check: Image exists?
│  └─ kubectl get pod -o yaml | grep image
│  └─ Manually pull: docker pull that-image
│  └─ If fails: Image doesn't exist
│
├─ Check: Registry auth?
│  └─ kubectl get pod -o yaml | grep imagePullSecrets
│  └─ Check ImagePullSecret exists
│  └─ Verify registry credentials
│
├─ Check: Disk space on node?
│  └─ ssh node and: df -h /var/lib/containers
│  └─ If full: Clean up images or add storage

Branch 3: Status = CrashLoopBackOff (App keeps crashing)
├─ Check: Previous logs
│  └─ kubectl logs pod-name --previous
│  └─ Look for error messages, stack traces
│
├─ Check: App entrypoint correct?
│  └─ kubectl get pod -o yaml | grep "command:\|args:"
│  └─ Manually test: docker run app:tag /bin/sh
│  └─ Does it run?
│
├─ Check: SecurityContext allows execution?
│  └─ kubectl get pod -o yaml | grep -A 5 "securityContext:"
│  └─ Check: runAsUser, fsGroup, allowPrivilegeEscalation
│  └─ Verify: App binary present in container image
│
└─ Check: Dependencies available?
   └─ kubectl logs pod-name --tail=20
   └─ Look for: "Error: Database connection refused"
   └─ Or: "Error: Config file not found"
   └─ Fix dependencies first

Branch 4: Status = Running (Pod started successfully)
├─ Is pod Ready? (kubectl get pods shows READY column)
│
├─ If READY = 0/1 or 0/2:
│  ├─ Readiness probe failing
│  ├─ kubectl logs pod-name
│  │  └─ Look for startup errors
│  │
│  ├─ Manual probe test:
│  │  └─ kubectl exec -it pod-name -- curl localhost:8080/health
│  │  └─ Should return HTTP 200
│  │
│  └─ If 500 error: Issue with app startup
│     └─ Check: Dependencies (database, config maps, secrets)
│
├─ If READY = 1/1 (Pod is Ready):
│  ├─ Can other pods reach this pod?
│  │  ├─ kubectl exec -it test-pod -- \
│  │  │  curl http://pod-ip:8080
│  │  │
│  │  ├─ If Connection refused: Network issue
│  │  │  └─ Check: Network policies
│  │  │  └─ Check: Service endpoints
│  │  │  └─ Check: Pod security policies
│  │  │
│  │  └─ If works: Good!
│  │
│  └─ Check pod performance
│     ├─ kubectl top pod
│     ├─ kubectl logs --tail=30 (look for errors)
│     └─ kubectl describe pod (look for events)

Any pod issue:
└─ Final check: Node status
   ├─ kubectl describe node where-pod-running
   ├─ Look for: MemoryPressure, DiskPressure, NotReady
   ├─ If unhealthy node: May need node reboot or repair
```

### Metrics Collection & Monitoring Stack

```
┌──────────────────────────────────────────────────────────┐
│               DATA COLLECTION LAYER                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  kubelet (every node):      Metrics Server (CPU/Mem):   │
│  ├─ container metrics       ├─ Polls kubelet every 15s  │
│  ├─ CPU, memory             ├─ Aggregates results       │
│  ├─ Network I/O             ├─ In-memory storage        │
│  ├─ Disk usage              └─ 15-minute retention      │
│  └─ /metrics endpoint                                   │
│                             Application Metrics:        │
│                             ├─ /metrics (Prometheus fmt)│
│                             ├─ HTTP requests/latency    │
│                             ├─ Business KPIs            │
│                             └─ Custom metrics           │
│                                                          │
└──────────────────────────────────────────────────────────┘
                                │
                ┌───────────────┼────────────────┐
                │               │                │
                ▼               ▼                ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌──────────────┐
    │   HPA           │ │   Prometheus    │ │    Loki      │
    │ (scales based   │ │ (time-series DB)│ │ (log store)  │
    │  on metrics)    │ │                 │ │              │
    └────────┬────────┘ └────────┬────────┘ └──────┬───────┘
             │                   │                 │
             ▼                   ▼                 ▼
    ┌─────────────────────────────────────────────────────┐
    │              QUERY/VISUALIZATION LAYER             │
    ├─────────────────────────────────────────────────────┤
    │                                                     │
    │  Grafana (dashboards):                             │
    │  ├─ CPU usage over time                            │
    │  ├─ Memory usage trends                            │
    │  ├─ Request rate & latency                         │
    │  ├─ Error rates                                    │
    │  └─ Custom business metrics                        │
    │                                                     │
    │  Alert Manager:                                    │
    │  ├─ Triggered when metrics exceed threshold        │
    │  ├─ CPU > 90% → Alert on-call engineer             │
    │  ├─ Error rate > 5% → Page team                    │
    │  └─ Send via: Email, Slack, SMS                    │
    │                                                     │
    └─────────────────────────────────────────────────────┘
```

---

## 6. Common Mistakes

### Mistake 1: Debugging Production by Modifying Running Pods

```yaml
# ❌ WRONG
Pod having issues:
1. kubectl exec -it prod-pod -- bash
2. Manually fix config file
3. Restart service in container
4. Pod works again
5. Week later: Pod restarted
6. Fix gone (container image doesn't have fix)

# ✅ CORRECT
1. Identify issue
2. Fix in source code or config
3. Update container image
4. Push to registry
5. Scale: kubectl set image deployment/app app=app:new-version
6. New pods get fixed image
7. Fix persists across restarts
```

---

### Mistake 2: Not Checking etcd Status During Cluster Issues

```yaml
# ❌ WRONG: Assuming API Server issue
$ kubectl top nodes
error: the server has received malformed response

Assumption: "API Server crashed"
Action: Restart API Server
Result: Still broken
Root cause: Actually etcd down!

# ✅ CORRECT
$ kubectl get nodes
error: Unable to connect to the server

Check 1: Is API Server running?
$ ps aux | grep kube-apiserver
# Yes, running

Check 2: Can API Server connect to etcd?
$ etcdctl endpoint health
127.0.0.1:2379, unhealthy, error: member 1234abcd not responding

Root cause found: etcd unhealthy

Fix: Restart etcd
$ systemctl restart etcd
```

---

### Mistake 3: Using kubectl logs without Following Actual Errors

```yaml
# ❌ WRONG
$ kubectl logs app-pod | grep -i error
# Shows old errors (noisy)

# ✅ CORRECT
$ kubectl logs app-pod --tail=50 | less
# See recent logs

$ kubectl logs app-pod -f
# Follow new logs in real-time

$ kubectl logs app-pod --timestamps=true | tail -20
# See exactly when errors started

For multiple pods:
$ kubectl logs -l app=api -f
# Stream from all api pods simultaneously
```

---

## 7. Debugging Scenarios (VERY IMPORTANT)

### Scenario 1: Application Extremely Slow (p99 latency 10 seconds)

```bash
# Step 1: Identify where slowness is
$ kubectl exec -it app-pod -- curl -w "@curl-format.txt" -o /dev/null \
  https://dependency-service:443/api
# Breaks down:
#   time_connect: 50ms
#   time_appconnect: 100ms
#   time_pretransfer: 150ms
#   time_redirect: 0ms
#   time_starttransfer: 9000ms (95% of slowness here!)
#   time_total: 9150ms

# Conclusion: Server response taking 9 seconds (not network)

# Step 2: Check if dependency pod is slow
$ kubectl describe pod dependency-pod | grep -i "ready\|restart"
READY 0/1  # Pod not ready! (readiness probe failing)

# Step 3: Check dependency pod logs
$ kubectl logs dependency-pod
Error: Database connection refused

# Step 4: Check if database is running
$ kubectl get pods -l app=database
NAME                 READY   STATUS    RESTARTS
database-0           0/1     Running   5

# Database pod not ready, keeps restarting

# Step 5: Check database pod startup
$ kubectl logs database-0 --previous
Error: Disk full
Unable to write to /var/lib/postgresql
```

Root cause: Database pod can't write to disk (full)

Solution: Clean up node or add storage

---

### Scenario 2: Intermittent Connection Timeouts (Not Always Failing)

```bash
# Symptom: Some requests work, some fail
# Pattern: Failures increase with load

# Step 1: Check if it's load-related
$ for i in {1..10}; do 
    curl http://service:8080 \
    && echo "Success $i" \
    || echo "Failed $i"
  done
# Success 1, Failed 2, Failed 3, Success 4, ...
# Intermittent (not every time)

# Step 2: Check service endpoints
$ kubectl get endpoints service-name
NAME            ENDPOINTS
service-name    10.0.2.1:8080,10.0.2.2:8080,10.0.2.3:8080

# 3 endpoints

# Step 3: Check pod readiness
$ kubectl get pods -l app=api
NAME         READY   STATUS
api-pod-1    1/1     Running
api-pod-2    1/1     Running
api-pod-3    0/1     Running  # One NOT ready!

# Step 4: Debug pod-3
$ kubectl logs api-pod-3
Warning: Probe failed 3 times, not restarting yet

# Readiness probe failing but pod still in endpoints!
# This is the intermittent issue

# Step 5: Check readiness probe
$ kubectl get pod api-pod-3 -o yaml | grep -A 10 "readinessProbe:"
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 3
  periodSeconds: 10
  initialDelaySeconds: 5

# Threshold: 3 failures before removing from endpoints
# But sometimes requests get routed to unready pod

# Fix: Reduce initialDelaySeconds or improve health check
$ kubectl set probe deployment api --readiness \
  --initial-delay-seconds=0 \
  --period-seconds=5 \
  --failure-threshold=1  # Remove immediately on failure
```

---

### Scenario 3: Memory Leak Suspected

```bash
# Symptom: Memory usage grows over time
# pod uses 200Mi today, 400Mi tomorrow, 800Mi next day

# Step 1: Confirm memory growth
$ kubectl top pods api-pod -A
# Shows current usage

# For historical: Use Prometheus
Prometheus query:
container_memory_working_set_bytes{pod="api-pod"}
# Graph over 7 days (shows upward trend)

# Step 2: Check if it's actual leak or just data accumulation
$ kubectl describe pod api-pod | grep "memory:"
memory: 512Mi (limit)

# Pod can use up to 512Mi
# If growing: Likely leak or increasing dataset

# Step 3: Get memory snapshot
$ kubectl exec -it api-pod -- \
  jmap -heap $pid  # For Java apps
# or
$ kubectl exec -it api-pod -- \
  ps aux
# Shows RSS memory

# Step 4: Profile application
$ kubectl exec -it api-pod -- \
  /opt/app/profiler.sh --memory --output=/tmp/profile.txt

$ kubectl cp api-pod:/tmp/profile.txt ./profile.txt

# Analyze profile to find what's consuming memory

# Step 5: Solutions
Option A: Increase memory limit
  limits: memory: 1Gi  (was 512Mi)
  Temporarily fixes, but doesn't solve leak

Option B: Restart pod periodically
  Add CronJob to restart pod every 6 hours
  Workaround, not ideal

Option C: Fix the leak
  Update application code
  Deploy new version
  Memory remains stable

Best: Option C (but requires work)

# Temporary mitigation:
$ kubectl set resources pod api --limits=memory=1Gi
$ kubectl set env deployment api JAVA_OPTS="-XX:MaxHeapSize=800m"
# Restart pods with changes
```

---

### Scenario 4: Cluster Nearly Out of Disk Space (90% full)

```bash
# Symptom: Pods getting evicted
$ kubectl describe node node-1
MemoryPressure:        False
DiskPressure:          True  # Disk pressure active!
PIDPressure:           False

# Step 1: SSH to node and check disk
$ ssh node-1
$ df -h
Filesystem      Size  Used Avail Use%
/               50G   45G  5G   90%
/var/lib/kubelet 100G  95G  5G   95%

# /var/lib/kubelet is the issue

# Step 2: See what's taking space
$ du -sh /var/lib/kubelet/*
50G  ./pods
30G  ./plugins
10G  ./cache

# Pods directory taking most space

# Step 3: Check what pods have large volumes
$ du -sh /var/lib/kubelet/pods/*/volumes/*
1G  /var/lib/kubelet/pods/app-uuid/volumes/configMap/data
800M /var/lib/kubelet/pods/cache-uuid/volumes/emptyDir/cache
2G  /var/lib/kubelet/pods/app-uuid/volumes/persistentVolumeClaim/data

# Large emptyDir consuming 800M

# Step 4: Which pod?
$ kubectl get pods -A --field-selector spec.nodeName=node-1 | grep cache
cache-pod-1  Running

# Check pod's emptyDir spec
$ kubectl get pod cache-pod-1 -o yaml | grep -A 20 "volumes:"

# Step 5: Solutions

Option A: Clean up pod data
$ kubectl exec -it cache-pod-1 -- rm -rf /data/*
# Clears cache, saves 800M

Option B: Reduce emptyDir size limit
spec:
  volumes:
  - name: cache
    emptyDir:
      sizeLimit: 500Mi  # Cap at 500Mi

Option C: Add storage to node
# Ask cloud provider for larger volume

Option D: Add new node
# Scale cluster, spread load

Step 6: Prevent recurrence
# Monitor disk usage
# Alert at 80%
# Implement: Log rotation, old pod cleanup
```

---

## 8. Interview Questions (15+ Scenario-Based)

### Level 1: Log Analysis

**Q1**: Pod logs show "connection refused", but the service exists. What do you check?

**Answer**:
```bash
1. Check if service has endpoints:
$ kubectl get endpoints service-name
# If empty: Pods not matching selector

2. Check pod labels:
$ kubectl get pods --show-labels | grep app-name

3. Check if pods are ready:
$ kubectl get pods -o wide
# If READY = 0/1: Pod not ready

4. Check if readiness probe is working:
$ kubectl logs pod-name
# Look for probe failures

5. Check application logs on the pod:
$ kubectl exec -it pod-name -- cat /var/log/app.log
# Look for specific why connection refused

6. Try telnet to pod directly:
$ kubectl exec -it debug-pod -- telnet pod-ip:8080
# If works: Service routing issue
# If fails: Pod not listening
```

---

**Q2**: How do you find which node a pod is running on?

**Answer**:
```bash
Method 1: kubectl describe
$ kubectl describe pod pod-name
Node: worker-1/10.0.0.5

Method 2: jsonpath
$ kubectl get pod pod-name -o jsonpath='{.spec.nodeName}'

Method 3: wide output
$ kubectl get pods -o wide
# NODE column shows node

Method 4: Once on node, find pod:
$ ssh worker-1
$ ps aux | grep container-id
(find running container)

Why find node?
- SSH to node for low-level debugging
- Check node disk/memory
- Check CNI logs
- Verify network rules
```

---

### Level 2: Production Operations

**Q3**: How do you gracefully drain a node without impacting users?

**Answer**:
```bash
Step-by-step drain:

1. Cordone node (stop new pods from scheduling)
$ kubectl cordon node-1
# Marks node: SchedulingDisabled

2. Check what pods will be evicted
$ kubectl get pods --field-selector spec.nodeName=node-1

3. Create Pod Disruption Budgets for critical apps
$ kubectl apply -f pdb.yaml
(ensure minAvailable is reasonable)

4. Drain pods (one at a time, gracefully)
$ kubectl drain node-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=300 \
  --timeout=10m
# Waits up to 10 min for pods to terminate
# Gives 300s grace period per pod

5. Pods terminate gracefully:
- Send SIGTERM to app
- App has 300s to shutdown cleanly
- Close connections, flush data
- After 300s: SIGKILL

6. Pods evicted and scheduled to other nodes
# Service endpoints updated
# Zero downtime (if PDB set properly)

7. After drain complete:
$ kubectl get pods --field-selector spec.nodeName=node-1
# No results (all gone)

8. Perform maintenance on node
$ ssh node-1
# Update, reboot, repair

9. Uncordon node
$ kubectl uncordon node-1
# Node available for new pods

Why this works:
- Cordoning prevents new pods landing here
- PDB ensures min pods always running
- Grace period allows graceful shutdown
- --timeout prevents infinite waiting
```

---

### Level 3: Root Cause Analysis (RCA)

**Q4**: Your production cluster had 30 minutes downtime. How do you investigate?

**Answer**:
```bash
RCA Steps:

1. Determine exact timeline
$ kubectl get events -A --sort-by='.lastTimestamp' | tail -100
# Find: When did things start breaking?
# Look for: "Node NotReady", "Pod evicted", "API error"

2. Check control plane status
$ kubectl get componentstatus
# Was API Server, Controller, Scheduler running?

3. Check node status at time of incident
$ prometheus select:
  kube_node_status_condition{condition="Ready", status="false"}
# Time range: During incident
# Shows which nodes became NotReady

4. Check events for that node
$ kubectl get events --field-selector involvedObject.name=node-1
# What happened to node-1?

5. SSH to affected node (if available)
$ ssh node-1
$ journalctl -u kubelet -n 200 --until="2024-04-10 14:30:00"
# Look for kubelet errors around incident time

6. Check etcd health
$ etcdctl --endpoints=localhost:2379 endpoint health
# Was etcd quorum maintained?

7. Check API Server logs
$ kubectl logs -n kube-system pod/kube-apiserver-master-1 \
  --since=30m | grep -i error

8. Check for resource exhaustion
$ prometheus select:
  node_memory_MemAvailable_bytes
  node_filesystem_avail_bytes
# Time: During incident
# Were nodes running out of resources?

9. Check for network issues
$ kubectl top nodes --all-namespaces
# CPU/Memory spikes possible?

10. Compile timeline
Time    Event                        Component   Impact
14:20   Node-1 disk full (90%)      Node        Kubelet slowdown
14:21   Kubelet stops reporting      Control     Node marked NotReady
14:22   API Server can't schedule    Scheduler   Pods pending
14:23   Cascade: More pods pending   Nodes       Queue builds up
14:30   On-call alerted              Ops         Manual intervention
14:35   Disk cleaned                 Node        Kubelet recovers
14:50   All pods rescheduled         Scheduler   Service restored

Root cause: Disk full on node-1

Prevention:
- Monitor disk usage (80% alert)
- Implement log rotation
- Regular cleanup of old images
- Larger disk in future
```

---

## 9. Advanced Insights (Observability, SRE, Cost)

### Choosing the Right Observability Stack

**Small cluster (< 50 nodes, < 500 pods)**:
```yaml
# Bare minimum:
- kubectl logs / kubectl top
- kubectl events
- Metrics Server (included in kubeadm)

Cost: $0 (included)
Time to debug: 5-10 minutes per issue
```

**Medium cluster (50-500 nodes, 500-5000 pods)**:
```yaml
Recommended:
- Prometheus (metrics collection and storage)
- Grafana (dashboards)
- Loki (centralized logs)
- AlertManager (alerting)

Cost: ~$500/month (self-hosted)
Time to debug: 1-2 minutes per issue
```

**Large cluster (500+ nodes, 5000+ pods)**:
```yaml
Recommended:
- Managed Prometheus (AWS AMP, Google Cloud Monitoring)
- Commercial tool (Datadog, New Relic, Splunk)
- Distributed tracing (Jaeger)
- Custom dashboards

Cost: $5000-50000/month
Time to debug: < 1 minute per issue
```

### SRE Golden Signals (What to Monitor)

```yaml
1. Latency
   └─ p50, p95, p99 response times
   └─ Goal: p99 < 200ms for web apps

2. Traffic
   └─ Requests per second
   └─ Goal: Scale within 5 minutes of surge

3. Errors
   └─ Error rate (5xx responses)
   └─ Goal: < 0.1% error rate

4. Saturation
   └─ How full is the system?
   └─ CPU/Memory utilization
   └─ Database connection pool usage
   └─ Goal: Keep < 70% saturated

Example alerting:
- Latency p99 > 500ms for 5m → Page on-call
- Error rate > 1% for 2m → Page on-call
- Saturation > 90% for 10m → Alert (not urgent)
- Traffic > 2x baseline → Scale + Alert
```

### Cost Optimization Through Monitoring

```yaml
Case study: SaaS platform

Before monitoring:
- Deployed with high resource limits (conservative)
- requests: 500m CPU / 1Gi memory
- 100 pods running always
- Monthly bill: $10,000

After Prometheus + Grafana:
1. Analyzed actual usage
   - p99 CPU: 200m (50% of requests)
   - p99 Memory: 300Mi (30% of requests)

2. Right-sized resources:
   - requests: 200m / 300Mi
   - savings: 60% less reserved

3. Implemented VPA for further optimization
   - Automatically right-size per pod

4. Added HPA:
   - Scale to 150 pods during peak
   - Scale to 10 pods during low traffic

Result:
- Peak capacity: 150 × 0.2 = 30 cores (was 50)
- Off-peak: 10 × 0.2 = 2 cores (was 50)
- Cost reduced to $3,000 (70% savings!)
- Actual performance: Better (less overprovisioning)
```

---

## Conclusion

Kubernetes troubleshooting is a **systematic process**:

1. **Gather facts** - Don't guess
2. **Check stack** - Application → Pod → Node → Cluster
3. **Use logs/metrics** - Observability is key
4. **Root cause** - Not just symptoms
5. **Prevention** - Monitoring alerts before problems

Success pattern: **Logs first, metrics second, traces third**
- Logs answer: "What happened?"
- Metrics answer: "How much/how often?"
- Traces answer: "Why did it take so long?"

Master troubleshooting and you become invaluable to your team!
