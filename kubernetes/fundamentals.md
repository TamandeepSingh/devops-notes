# Kubernetes Fundamentals: Deep Dive

## 1. Core Concept (Deep Internal Working)

Kubernetes is a **declarative container orchestration platform** that manages containerized workloads across clusters of machines. At its core, it operates on the principle of **desired state management**:

```
User declares desired state → API Server receives → etcd persists → Controllers reconcile
```

### Key Internal Mechanics:

**API Server (kube-apiserver)**
- Single source of truth for all cluster state
- RESTful interface for all operations
- Validates requests, applies authorization policies
- Stores everything in etcd (distributed key-value store)

**Control Loop (Reconciliation)**
```
etcd (actual state) → Controller Manager → Observes state diff → Takes action → Updates etcd
```

Each controller (Deployment, StatefulSet, DaemonSet, etc.) runs an infinite loop:
1. Watch current state
2. Compare with desired state
3. Calculate difference
4. Execute actions
5. Repeat

**kubelet (Node Agent)**
- Runs on every node
- Receives Pod specifications from API Server
- Manages container runtime (usually containerd/Docker)
- Reports node and pod status back to API Server
- Performs health checks and restarts failed containers

**kube-proxy (Network Proxy)**
- Maintains network rules on each node
- Routes traffic to service endpoints
- Implements Service abstraction (more in networking section)

---

## 2. How Kubernetes Actually Works Internally

### Pod Lifecycle (Most Critical to Understand)

```
Create Pod → Pending → Container Runtime Pull Image → Running → (Healthy/Unhealthy)
     ↓                                                    ↓
  Scheduled                                        Ready/NotReady
```

**Phase 1: Admission & Scheduling**
```yaml
1. API Server validates Pod YAML
2. Mutating Webhooks modify Pod (inject sidecars, init secrets)
3. Validation Webhooks check business logic
4. Scheduler selects optimal node based on:
   - Resource requests/limits
   - Node affinity/anti-affinity
   - Pod affinity/anti-affinity
   - Taints & tolerations
   - Storage locality
```

**Phase 2: Node Assignment**
```
kubelet on selected node:
  1. Receives Pod spec
  2. Checks image locally (ImagePullBackOff if not found)
  3. Creates container via container runtime
  4. Mounts volumes
  5. Injects environment variables
  6. Starts container
```

**Phase 3: Readiness/Liveness Probes**
```yaml
- Readiness Probe: "Is this pod ready to serve traffic?"
  → Failing = removed from Service endpoints
  → Pod stays running but isolated

- Liveness Probe: "Is this pod still alive?"
  → Failing = container restarted immediately
  → kubelet kills container, container runtime respawns it

- Startup Probe: "Has pod finished initialization?"
  → Grace period before readiness/liveness checked
```

### etcd: The Source of Truth

```
etcd stores EVERYTHING:
├── /registry/pods/namespace/pod-name → Pod state
├── /registry/services/namespace/svc-name → Service
├── /registry/deployments/namespace/deploy-name → Deployment
└── /registry/... → All API objects

All writes go through API Server (etcd has no security)
API Server is the ONLY client to etcd
```

**Critical**: etcd is NOT transactional. Multiple updates can cause race conditions if not careful.

### Deployment Reconciliation Loop (Detailed)

```
Desired: Deployment with 3 replicas
  ↓
Deployment Controller watches Deployment object
  ↓
Current state: 1 Pod running, 1 Pod CrashLoopBackOff, 0 Pods pending
  ↓
Controller calculates: Need 2 new Pods created, 1 Pod should be deleted
  ↓
Creates 2 Pod spec objects via API Server
  ↓
Scheduler assigns them to nodes
  ↓
Kubelet creates containers
  ↓
Pods become Ready
  ↓
State matches Desired → Loop continues watching for changes
```

---

## 3. When to Use vs Not Use

### When Kubernetes is Right ✅

| Scenario | Why |
|----------|-----|
| **Microservices** | Multiple independent services that scale independently |
| **High availability required** | Self-healing, multi-node deployment |
| **Resource optimization** | Bin-packing containers on nodes, better utilization |
| **Multiple environments** | Same manifests for dev/staging/prod |
| **Frequent deployments** | Rolling updates, canary deployments built-in |
| **Large teams** | Multi-tenancy via namespaces |
| **Compliance/security** | RBAC, network policies, pod security policies |

### When Kubernetes is Overkill ❌

| Scenario | Better Alternative |
|----------|--------|
| **Single monolithic application** | Docker Swarm, Nomad, or managed services (Cloud Run) |
| **<3 containers** | VMs or application-level orchestration |
| **Prototype/MVP** | Docker Compose, serverless |
| **Real-time ML training** | Slurm, Ray, or specialized ML platforms |
| **Batch jobs** | AWS Batch, Google Cloud Tasks |
| **Minimal ops budget** | Managed services (App Engine, Heroku) |

### Trade-offs

```
Kubernetes Benefits:
  ✓ Self-healing and auto-scaling
  ✓ Multi-cloud portability
  ✓ Declarative configuration
  ✓ Large ecosystem

Kubernetes Costs:
  ✗ Steep learning curve (6-12 months to mastery)
  ✗ Operational overhead (etcd backup, node patching)
  ✗ Debugging complexity (distributed system problems)
  ✗ Cost overhead (25-30% higher than VMs)
```

---

## 4. Real-World Production Scenario

### Scenario: E-Commerce Black Friday Spike

**Initial State (Thursday before Black Friday)**
- 3-tier app: API (15 replicas), Database Connection Pool (5 replicas), Cache (3 replicas)
- Running on 10 nodes (3 master, 7 worker)

**Friday Morning Traffic Pattern**
```
00:00 → 05:00:   10% normal traffic
05:00 → 10:00:   50% normal traffic  
10:00 → 14:00:   300% normal traffic (PEAK)
14:00 → 23:59:   200% normal traffic
```

**What Kubernetes Does (Automatic)**

```yaml
# Horizontal Pod Autoscaler already configured
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-autoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 15
  maxReplicas: 200
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**Timeline**
```
05:00 - Kubelet reports CPU usage 40% (OK)
10:30 - CPU jumps to 85%, Memory 82%
        → HPA triggers immediately
        → Deployment controller creates 20 new Pods
        → Scheduler distributes across existing nodes
        
10:35 - All 20 Pods not fitting on nodes
        → Cluster Autoscaler (cloud integration) detected
        → Adds 3 new nodes
        → New Pods scheduled
        
11:00 - Load stabilized at predicted level
        → 140 API replicas running
        → 2 new nodes added
        
14:00 - Traffic starts dropping
        → HPA reduces to 90 replicas
        → Nodes with low utilization marked for draining
        → Pods evicted gracefully (50s grace period)
        → Cluster Autoscaler removes empty nodes
        
15:00 - Back to 15 replicas, 7 nodes
```

**What Could Go Wrong**
```
❌ Scenario 1: Insufficient resource limits
   → New Pods can't be scheduled
   → HPA keeps trying, Queue builds up
   → Requests timeout

❌ Scenario 2: Image pull takes too long
   → Pending Pods never become Running
   → No actual increase in capacity
   → Traffic cascades to remaining pods

❌ Scenario 3: Database connection pool breaks
   → 140 pods try to connect to 5 connection pool replicas
   → Connection refused errors
   → Cascade failure across services

❌ Scenario 4: Node runs out of disk space
   → kubelet marks node NotReady
   → Pods get evicted
   → New Pods scheduled to other nodes
   → Cascade effect
```

**Production Lessons Learned**
1. **Always set resource requests/limits** - HPA depends on accurate metrics
2. **Test scaling before production** - Simulate 10x traffic
3. **Use multiple metrics for scaling** - CPU alone is insufficient
4. **Plan for slow image pulls** - Large images cause delayed scaling
5. **Monitor etcd performance** - Heavy API load can overload etcd

---

## 5. Architecture Diagrams (Text-Based)

### Kubernetes Cluster Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      KUBERNETES CLUSTER                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              CONTROL PLANE (master nodes)               │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │                                                          │  │
│  │  ┌─────────────┐  ┌────────────────┐  ┌─────────────┐  │  │
│  │  │ API Server  │  │ Scheduler      │  │ Controller  │  │  │
│  │  │             │  │                │  │ Manager     │  │  │
│  │  └──────┬──────┘  └────────────────┘  └─────────────┘  │  │
│  │         │                                               │  │
│  │  ┌──────▼──────────────┐                               │  │
│  │  │  etcd (state DB)    │                               │  │
│  │  │  127.0.0.1:2379     │                               │  │
│  │  └─────────────────────┘                               │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          │                                     │
│                          │ (API calls)                         │
│                          ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                 WORKER NODES (pods)                     │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │                                                          │  │
│  │  ┌──────────────────┐  ┌──────────────────┐            │  │
│  │  │  Worker Node 1   │  │  Worker Node 2   │            │  │
│  │  ├──────────────────┤  ├──────────────────┤            │  │
│  │  │ ┌──────────────┐ │  │ ┌──────────────┐ │            │  │
│  │  │ │ Pod1         │ │  │ │ Pod3         │ │            │  │
│  │  │ │ (container)  │ │  │ │ (container)  │ │            │  │
│  │  │ └──────────────┘ │  │ └──────────────┘ │            │  │
│  │  │ ┌──────────────┐ │  │ ┌──────────────┐ │            │  │
│  │  │ │ Pod2         │ │  │ │ Pod4         │ │            │  │
│  │  │ │ (container)  │ │  │ │ (container)  │ │            │  │
│  │  │ └──────────────┘ │  │ └──────────────┘ │            │  │
│  │  │                  │  │                  │            │  │
│  │  │ ┌──────────────┐ │  │ ┌──────────────┐ │            │  │
│  │  │ │ kubelet      │ │  │ │ kubelet      │ │            │  │
│  │  │ │ (agent)      │ │  │ │ (agent)      │ │            │  │
│  │  │ └──────────────┘ │  │ └──────────────┘ │            │  │
│  │  │ ┌──────────────┐ │  │ ┌──────────────┐ │            │  │
│  │  │ │ kube-proxy   │ │  │ │ kube-proxy   │ │            │  │
│  │  │ │ (networking) │ │  │ │ (networking) │ │            │  │
│  │  │ └──────────────┘ │  │ └──────────────┘ │            │  │
│  │  │                  │  │                  │            │  │
│  │  └──────────────────┘  └──────────────────┘            │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Pod Lifecycle State Machine

```
                    ┌─────────────┐
                    │   Pending   │
                    └──────┬──────┘
                           │
                    Scheduler assigns
                    Image pull begins
                           │
                    ┌──────▼──────────┐
                    │  Container Init │
                    └──────┬──────────┘
                           │
                   Startup probe checks
                           │
        ┌──────────────────▼──────────────────┐
        │         Running                     │
        │ ┌────────────────────────────────┐  │
        │ │ Readiness Probe checks         │  │
        │ │ • Pass → Pod added to Service  │  │
        │ │ • Fail → Pod removed from Svc  │  │
        │ └────────────────────────────────┘  │
        │                                      │
        │ ┌────────────────────────────────┐  │
        │ │ Liveness Probe checks          │  │
        │ │ • Pass → Pod healthy           │  │
        │ │ • Fail → Container restarted   │  │
        │ └────────────────────────────────┘  │
        └──────────────┬───────────────────────┘
                       │
         Delete requested or failed
                       │
        ┌──────────────▼───────────────────┐
        │   Terminating                    │
        │ • preStop hook runs (if defined) │
        │ • Grace period: 30s (default)    │
        │ • SIGTERM sent to processes      │
        │ • After grace → SIGKILL          │
        └──────────────┬───────────────────┘
                       │
                    ┌──▼───┐
                    │ Dead │
                    └──────┘
```

### Service Discovery & Load Balancing

```
┌────────────────────────────────────────────────────────────┐
│ Service (service.namespace.svc.cluster.local)             │
│ IP: 10.0.1.100 (Cluster IP - virtual)                    │
└────────────────────────────────────────────────────────────┘
          │
          │ DNS resolution
          │ 10.0.1.100 → kube-proxy iptables rules
          │
    ┌─────┴─────────┬──────────────┬──────────────┐
    │               │              │              │
    ▼               ▼              ▼              ▼
 Pod1         Pod2           Pod3           Pod4
10.0.2.1    10.0.2.2       10.0.3.1       10.0.3.2
(Node1)     (Node1)        (Node2)        (Node2)
  ✓ Running  ✓ Running      ✓ Ready        ✓ Ready
  
Traffic load balanced across endpoints using:
  • iptables (default mode)
  • IPVS (scalable, for 1000+ services)
  • userspace (legacy)
```

---

## 6. Common Mistakes

### Mistake 1: Not Setting Resource Requests/Limits

```yaml
# ❌ WRONG
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:1.0
    # No requests or limits!

# ✅ CORRECT
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:1.0
    resources:
      requests:
        cpu: 100m        # Scheduler uses this for placement
        memory: 128Mi
      limits:
        cpu: 500m        # Pod killed if exceeded
        memory: 512Mi
```

**Why it matters**: Without requests, Scheduler can overcommit nodes. Without limits, a misbehaving pod can OOM-kill all pods on the node.

---

### Mistake 2: Relying on Readiness Without Understanding It

```yaml
# ❌ WRONG - No readiness probe
spec:
  containers:
  - name: api
    image: api:1.0
    # Pod immediately added to Service endpoints
    # For 5 seconds, pod is startup, connections fail

# ✅ PARTIAL - Present but insufficient
spec:
  containers:
  - name: api
    image: api:1.0
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      # Defaults: initialDelaySeconds: 0
      # This is too aggressive!

# ✅ CORRECT
spec:
  containers:
  - name: api
    image: api:1.0
    startupProbe:  # New in 1.16+
      httpGet:
        path: /ready
        port: 8080
      failureThreshold: 30  # 30 * 10s = 300s max startup
      periodSeconds: 10
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
      timeoutSeconds: 2
      failureThreshold: 3
    
    livenessProbe:
      httpGet:
        path: /alive
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 30
      timeoutSeconds: 2
      failureThreshold: 3
```

**Common misunderstanding**: Readiness ≠ Liveness
- **Readiness**: Can I send traffic? (Should I include in service endpoints?)
- **Liveness**: Is the process stuck? (Should I restart?)

---

### Mistake 3: Pod Eviction During Deployment

```yaml
# ❌ WRONG - Two extreme strategies
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0      # Never takes down a pod
    maxSurge: 100%         # Creates 100% new pods (2x memory spike)

# ✅ BETTER BALANCED
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1      # At most 1 pod unavailable
    maxSurge: 50%          # At most 50% over desired replicas
    
# ✅ PRODUCTION BEST PRACTICE
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%    # 25% of replicas can be unavailable
    maxSurge: 25%          # 25% more pods during rollout
  
spec:
  minReadySeconds: 10      # Pod must be ready for 10s before considered "ready"
  progressDeadlineSeconds: 600  # Timeout long deployments
  
  template:
    spec:
      terminationGracePeriodSeconds: 30
      # Gives pods 30s to gracefully shutdown
```

---

### Mistake 4: Not Understanding CNI (Container Network Interface)

```yaml
# ❌ MISCONCEPTION: Pods on same node share localhost
# They DON'T! Each pod has its own network namespace

# Pod1: 10.0.2.1:8080
# Pod2: 10.0.2.2:8080
# They cannot reach each other via localhost:8080

# They communicate via IP addresses (overlay network)
# Default CNI: 10.x.x.x range (not routable outside cluster)

# ✅ TRUTH: Every pod must be reachable from every other pod
# This is enforced by:
# 1. CNI plugin (Flannel, Calico, Weave)
# 2. Virtual network interfaces
# 3. Overlay or BGP routing
```

---

### Mistake 5: High Memory Requests, No Memory Limits

```yaml
# ❌ WRONG - Memory bomb
spec:
  containers:
  - name: app
    image: app:1.0
    resources:
      requests:
        memory: 8Gi  # Scheduler reserves 8Gi
      # No limits! Pod can use 30Gi

# Result: OOM killer cascades to other pods

# ✅ CORRECT
spec:
  containers:
  - name: app
    image: app:1.0
    resources:
      requests:
        memory: 2Gi   # Realistic estimate
      limits:
        memory: 2.5Gi # Hard cap, pod dies if exceeded
```

---

## 7. Debugging Scenarios (VERY IMPORTANT)

### Scenario 1: Pod Stuck in Pending

```bash
# Symptom
$ kubectl get pods
NAME     READY   STATUS    RESTARTS   AGE
app-1    0/1     Pending   0          5m

# Debug Step 1: Check events
$ kubectl describe pod app-1
Events:
  Type     Reason             Age    Message
  ----     ------             ----   -------
  Warning  FailedScheduling   5m     0/3 nodes available

# Debug Step 2: Check what nodes have available resources
$ kubectl top nodes
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
worker-1   2000m        100%   8Gi             100%
worker-2   1500m        75%    6Gi             75%
worker-3   1000m        50%    4Gi             50%

# Debug Step 3: Check pod resource requests
$ kubectl get pod app-1 -o yaml | grep -A 5 resources

# 5 Possible Causes:
1. Insufficient resources: No node has 4 CPUs available
   → Scale cluster or reduce pod requests

2. Node selector mismatch:
   $ kubectl get nodes --show-labels
   → Pod requires label that doesn't exist

3. Taints & Tolerations:
   $ kubectl describe node worker-1 | grep Taints
   → Pod doesn't tolerate the taint

4. Storage: PVC not bound
   $ kubectl get pvc
   → Check if PVC is Pending

5. Image pull policy issues:
   $ kubectl describe pod app-1 | grep -i pull
   → ImagePullBackOff means image doesn't exist
```

### Scenario 2: Pod CrashLoopBackOff

```bash
# Symptom
$ kubectl get pods
NAME     READY   STATUS             RESTARTS   AGE
app-1    0/1     CrashLoopBackOff   5          2m

# This means: Container starts, fails, exits, kubelet restarts, repeat

# Debug: Check logs
$ kubectl logs app-1
Error: database connection refused
    at main.go:45

# Problem: Application is trying to connect to database that doesn't exist

# Check: Database service
$ kubectl get svc
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
database  ClusterIP   10.0.1.150      <none>        5432/TCP

# Check: Can communicate to database?
$ kubectl run debug --image=busybox -it --rm -- sh
/ # wget http://database:5432
# If this fails, database service is down

# Debug: Check database pod
$ kubectl get pods -l app=database
NAME                READY   STATUS    RESTARTS   AGE
database-1-abcde    1/1     Running   0          1h

# Check: Database logs
$ kubectl logs database-1-abcde
ERROR: Port 5432 already in use
# Two instances of database trying to use same port

# Solution: Fix the application to wait for database
---
# ✅ Fix: Add startup probe and connection retry logic
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        image: app:1.0
        startupProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - ./wait-for-db.sh
          failureThreshold: 30
          periodSeconds: 5
        env:
        - name: DB_HOST
          value: database
        - name: DB_PORT
          value: "5432"
---
```

### Scenario 3: Service Can't Find Pods

```bash
# Symptom
$ curl service-name:8080
Connection refused

# But pods are running!
$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
app-1-abcde             1/1     Running   0          5m
app-1-fghij             1/1     Running   0          5m

# Debug Step 1: Check service endpoints
$ kubectl get endpoints service-name
NAME            ENDPOINTS                   AGE
service-name    10.0.2.1:8080,10.0.2.2:8080   5m

# Good! Endpoints are populated. If empty, see below.

# Debug Step 2: Check readiness probe status
$ kubectl get pod app-1-abcde -o yaml | grep -A 20 readinessProbe

# If pods AREN'T in endpoints (empty list):
$ kubectl get endpoints service-name
NAME            ENDPOINTS   AGE
service-name    <none>      5m

# Causes:
1. Readiness probe failing
   $ kubectl logs app-1-abcde | tail -20
   # Check readiness probe logic

2. Pod labels don't match service selector
   $ kubectl get pod app-1-abcde --show-labels
   $ kubectl get svc service-name -o yaml | grep -A 5 selector
   # Labels must match selector exactly

3. Service and pods in different namespaces
   $ kubectl get pods -A
   # Check namespace mismatch

# Debug Step 3: Test DNS resolution
$ kubectl run debug --image=busybox -it --rm -- sh
/ # nslookup service-name
Address 1: 10.0.1.100 service-name.default.svc.cluster.local

# If DNS fails: CoreDNS might be broken
$ kubectl get pods -n kube-system -l k8s-app=kube-dns
```

### Scenario 4: Node NotReady

```bash
# Symptom
$ kubectl get nodes
NAME       STATUS     ROLES   AGE   VERSION
worker-1   NotReady   worker  1h    v1.28.0
worker-2   Ready      worker  1h    v1.28.0

# Debug Step 1: Check node conditions
$ kubectl describe node worker-1
Conditions:
  Type             Status  LastHeartbeatTime             Reason
  ----             ------  -----------------             ------
  Ready            False   2024-04-10T12:30:00Z          KubeletNotReady
  MemoryPressure   False   2024-04-10T12:29:00Z
  DiskPressure     True    2024-04-10T12:30:00Z          MinimumFilesizeViolated
  PIDPressure      False   2024-04-10T12:30:00Z

# DiskPressure=True means disk is full!

# Debug Step 2: SSH to node and check disk
$ ssh worker-1
$ df -h
Filesystem      Size  Used Avail Use%
/               50G   48G  2G   96%
/var/lib/kubelet 20G  19G  1G   95%

# The /var/lib/kubelet is full! This is where kubelet stores:
# - Pod logs
# - Container layers
# - Volume data

# Debug Step 3: Clean up
$ docker ps -a
# Hundreds of stopped containers taking space

# Fix: Trigger kubelet garbage collection
# kubelet automatically cleans when:
#   - evictionHard.imagefs.available < 15%
#   - evictionHard.nodefs.available < 10%

# Manual cleanup:
$ docker system prune -a --volumes  # Removes unused images/volumes

# Or configure kubelet to be more aggressive:
kubelet --eviction-hard=imagefs.available < 20%
```

### Scenario 5: High Latency Between Pods

```bash
# Symptom
$ kubectl exec -it app-1-abcde -- ping app-2-fghij
PING app-2-fghij (10.0.2.2) 56(84) bytes of data.
64 bytes from 10.0.2.2: time=250ms  # 250ms! Should be < 5ms

# Debug Step 1: Check if pods on same node
$ kubectl get pods -o wide
NAME            NODE       IP
app-1-abcde     worker-1   10.0.2.1
app-2-fghij     worker-2   10.0.2.2

# Different nodes. Check inter-node latency
$ kubectl exec -it app-1-abcde -- traceroute 10.0.2.2
# If high latency here, network issue between nodes

# Debug Step 2: Check MTU (packet fragmentation)
$ kubectl exec -it app-1-abcde -- ping -M do -s 1400 10.0.2.2
# If this fails but -s 1200 works, MTU issue
# CNI default: 1500 bytes MTU
# IP +UDP overhead: 28 bytes
# Safe payload: 1472 bytes

# Debug Step 3: Check kube-proxy mode
$ kubectl get nodes worker-1 -o yaml | grep -A 5 kubeProxyVersion

# Slow mode:
#   kube-proxy --mode=userspace  (deprecated)

# Better modes:
#   kube-proxy --mode=iptables   (default, good for <1000 services)
#   kube-proxy --mode=ipvs       (best for scale)

# Debug Step 4: Check CNI plugin performance
$ kubectl get pods -n kube-system -l component=cni
# Check if CNI plugin is saturated
$ kubectl top pods -n kube-system
# CNI pods should have low CPU
```

---

## 8. Interview Questions (15+ Scenario-Based)

### Level 1: Fundamentals

**Q1**: You deploy a Deployment with 3 replicas. After 1 minute, you see 4 pods running. Explain why and how Kubernetes fixes this.

**Answer**: 
- Kubernetes desired state: 3 replicas
- Actual state: 4 running
- The Deployment controller continuously watches and compares
- It detects the mismatch (3 ≠ 4)
- It deletes 1 pod to reconcile
- The pod that gets deleted is typically the oldest one
- The reconciliation loop ensures desired state within 30-40 seconds

---

**Q2**: A pod has `requests: {cpu: 100m}` and `limits: {cpu: 500m}`. During traffic spike, the pod consumes 600m CPU. What happens?

**Answer**:
- The pod is **throttled** by the kernel
- CPU limits use the CFS (Completely Fair Scheduler) quota mechanism
- Kernel prevents the pod from using more than 500m
- The pod slows down (increased latency)
- It doesn't get killed (unlike memory limits)
- Memory limit exceeded = pod OOM-killed
- CPU limit exceeded = pod throttled and slowed

---

**Q3**: You have 3 readiness probes failing across your pods. They're still running and receiving traffic. Why?

**Answer**:
- This is the desired behavior!
- Readiness probe checks if pod can serve traffic
- Failing readiness = pod should be removed from Service
- The Service endpoints should NOT include these pods
- If they're still receiving traffic, it's a bug in:
  - kube-proxy (not updating iptables rules)
  - CoreDNS (returning stale IPs)
  - Application load balancer (not respecting Service endpoints)

---

### Level 2: Architecture & Internals

**Q4**: Explain the complete journey of creating a pod from `kubectl apply` to running.

**Answer**:
1. **Client (kubectl)**: YAML validated locally, sent as HTTP POST
2. **API Server**: Validates YAML schema, runs admission webhooks
3. **etcd**: API Server stores Pod object as JSON
4. **Scheduler**: Watches etcd for unscheduled Pods, calculates best node, updates Pod.spec.nodeName
5. **kubelet (on assigned node)**: Watches API Server for Pods with its nodeName
6. **Container Runtime**: Creates container, manages lifecycle
7. **kube-proxy**: Updates iptables rules to route traffic to pod IP
8. **CNI Plugin**: Assigns pod IP from cluster CIDR subnet, configures networking
9. **Pod reaches Running**: Container is created, stdout/stderr captured
10. **Readiness Probe**: kubelet runs HTTP GET to `/health`, if passes, pod added to Service endpoints

---

**Q5**: During a node failure, what prevents cascading failure across the cluster?

**Answer**:
- **Node becomes unreachable**: kubelet can't report heartbeat to API Server
- **API Server marks node NotReady**: After 40-50 seconds (configurable)
- **Pod Eviction Controller**: Detects NotReady node after grace period (~5-10 min)
- **Pods evicted gracefully**: 
  - Pods receive SIGTERM (30s grace period by default)
  - Pods have chance to close connections, flush data
  - After grace period, SIGKILL sent
- **New Pods created**: Deployment controller sees replicas < desired
- **Scheduler assigns to healthy nodes**: Ensures other nodes don't get overloaded
- **Temporary spike in latency**: During re-scheduling (usually <30s)

---

**Q6**: You accidentally deleted the etcd data. What's the state of running pods and services?

**Answer**:
- **Running pods**: Still running! They're managed by the container runtime, not etcd
- **Services**: Still working! kube-proxy cached the rules in iptables
- **New operations**: Fail immediately. API Server can't read/write state
- **Cluster is broken**: Can't schedule new pods, delete existing ones, or update deployments
- **Recovery**: Restore etcd backup, API Server rebuilds in-memory state
- **Lesson**: etcd is critical, must have backup strategy (3-4 node quorum, automated snapshots)

---

### Level 3: Production & Scaling

**Q7**: Your HPA is set to scale from 5 to 100 replicas at 80% CPU. Traffic arrives, but only scales to 20 replicas. Why?

**Answer** (multiple possibilities):
1. **Resource limits on node**: Nodes can only fit 20 more pods (memory/CPU constraints)
2. **Pending pods stuck**: New pods can't schedule → HPA sees no capacity
3. **Image pull taking too long**: Pods Pending for minutes → HPA think they're running
4. **Resource quota hit**: Namespace has quota limiting total CPU/memory
5. **Cluster autoscaler not triggered**: Nodes full, no space for new pods
6. **Metrics not updating**: HPA looks at 60-90 second old metrics, decision lag

**Diagnosis**:
```bash
# Check if pods are pending
kubectl get pods -o wide | grep Pending

# Check resource availability
kubectl describe nodes | grep "Allocated resources"

# Check quota
kubectl describe resourcequota -n namespace

# Check HPA status
kubectl describe hpa app-hpa
```

---

**Q8**: You're doing a rolling deployment with `maxSurge: 50%`. On a 10-pod deployment, your cluster runs out of memory. What happens?

**Answer**:
- Deployment controller creates 5 new pods (50% surge)
- Scheduler can't place them all (insufficient memory)
- Some pods become Pending
- Deployment pauses (can't continue rolling update)
- Old pods still running
- You have Pending pods consuming API resources
- **Eventual outcome**: Deployment timeout or manual intervention

**Prevention**:
- Pre-plan resource needs: 10 pods base + 5 surge = 15 pod capacity needed
- Use Cluster Autoscaler to add nodes before hitting limits
- Use Pod Disruption Budgets to prevent cascading evictions

---

### Level 4: Debugging & Troubleshooting

**Q9**: A service works from some pods but not others. All pods have the same deployment spec. What's wrong?

**Answer** (likely causes):
1. **Network policies**: Some namespaces blocked from accessing service
2. **DNS not working in some pods**: CoreDNS pod colocated on specific node
3. **kube-proxy not updated**: Not all nodes have latest iptables rules
4. **Pod network namespace isolation**: Incorrect CNI setup

**Diagnosis**:
```bash
# Test DNS from each pod
kubectl exec -it pod1 -- nslookup service-name
kubectl exec -it pod2 -- nslookup service-name

# Check network policies
kubectl get networkpolicies

# Check kube-proxy logs on different nodes
kubectl logs -n kube-system kube-proxy-<node1> | grep service-name
kubectl logs -n kube-system kube-proxy-<node2> | grep service-name
```

---

**Q10**: Pods keep getting evicted even though the node looks healthy. Why?

**Answer**:
- **Kubelet eviction**: node is hitting thresholds
  - Memory pressure: 85% full (configurable)
  - Disk pressure: 90% full
  - inode pressure: 90% inode utilization
  - PID pressure: 90% of /proc/sys/kernel/pid_max

- **Priority-based eviction**:
  - Pods with no requests evicted first
  - Then pods exceeding requests
  - Then guaranteed pods (last)

**Check**:
```bash
# Node conditions
kubectl describe node | grep -i pressure

# Kubelet log
ssh node-name
journalctl -u kubelet | grep evicting

# Disk usage
df -h
du -sh /var/lib/kubelet
```

---

## 9. Advanced Insights (Performance, Cost, Scaling)

### Performance Optimization

**API Server Performance**
- etcd read latency SLA: <100ms (watch operations are continuous streaming)
- API Server throttles at high request rates to protect etcd
- Default: 400 requests per second across entire cluster
- Solution: Increase `--max-requests-inflight=1500`

**Network Performance**
- Pod-to-pod latency: Target <5ms same-node, <20ms cross-node
- Service DNS lookup: 100-200ms first time (cache after)
- Ingress latency: 50-200ms depending on implementation

**Storage Performance**
- etcd I/O critical: Use SSD, not HDD
- etcd disk I/O: p99 latency should be <10ms
- Large objects stored in etcd slow down all operations
- Solution: Use external storage for large data (>1MB values)

---

### Cost Optimization

**Resource Sizing**
```
1. Measure actual usage (prometheus + VPA)
2. Set requests = p99 + 20% buffer
3. Set limits = requests + 50% headroom
4. Over-provisioning = wasted cost
5. Under-provisioning = crashes and throttling
```

**Node Utilization**
```
Good target: 70% CPU + 75% memory
  (leaves headroom for spikes, node eviction, pod overhead)

Bad targets:
  50% utilization = 50% cost wasted
  95% utilization = frequent evictions, cascades
  100% utilization = cluster unstable
```

**Cost Reduction Strategies**
1. Use spot/preemptible instances for fault-tolerant workloads
2. Co-locate pods with different traffic patterns (web + batch jobs)
3. Regular cleanup of unused resources (PVCs, ConfigMaps, Secrets)
4. Implement resource quotas per teams/namespaces

---

### Scaling Patterns

**Horizontal Pod Autoscaling (HPA)**
```yaml
# Scales pods (not great for large jumps)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 10
  maxReplicas: 1000
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: request_latency_p99
      target:
        type: AverageValue
        averageValue: "200m"  # 200 milliseconds
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scaling down
      policies:
      - type: Percent
        value: 50  # Scale down by 50% max
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60   # Wait 1min before scaling up again
      policies:
      - type: Percent
        value: 100  # Double the pods
        periodSeconds: 30
```

**Vertical Pod Autoscaling (VPA)**
```yaml
# Adjusts resource requests/limits based on actual usage
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: app
  updatePolicy:
    updateMode: "Auto"  # Evict and recreate pods with new resources
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
```

**Note**: HPA + VPA together need careful tuning (can cause oscillation)

---

### Conclusion

Kubernetes fundamentals revolve around three concepts:
1. **Desired state management** - Declare what you want, K8s makes it happen
2. **Declarative configuration** - YAML describes infrastructure, not imperative steps
3. **Fault tolerance** - Built-in self-healing, high availability, graceful degradation

Master these three, and everything else (networking, storage, security) becomes easier to understand.
