# Kubernetes Architecture: Deep Dive

## 1. Core Concept (Deep Internal Working)

Kubernetes architecture is built on the **orchestration pattern**: separate control plane (brain) from worker nodes (muscles). The control plane makes decisions; the worker nodes execute them. This separation enables:

- **Decoupling**: Control plane can be patched/upgraded independently
- **Scalability**: Single control plane manages 5000+ nodes (GKE)
- **Fault isolation**: Worker node failure doesn't affect control plane stability

### Architectural Layers

```
┌─────────────────────────────────────────────────┐
│          USER (kubectl, UI, APIs)               │
└─────────────────────────────────────────────────┘
                        │
        ┌───────────────┴───────────────┐
        │  API Server (REST endpoint)   │
        └───────────────┬───────────────┘
                        │
        ┌───────────────┴────────────────────────────┐
        │         Controllers (reconciliation)        │
        │  • Deployment → Pod creation               │
        │  • Service → Endpoint management           │
        │  • ReplicaSet → Pod counting & scaling     │
        └───────────────┬────────────────────────────┘
                        │
                    ┌───▼───┐
                    │ etcd  │  ← Single source of truth
                    └───────┘
                        │
    ┌───────────────────┴───────────────────┐
    │       kubelet (every node)            │
    │  • Pod creation/destruction           │
    │  • Health monitoring                  │
    │  • Resource enforcement               │
    └───────────────────┬───────────────────┘
                        │
        ┌───────────────┴───────────────┐
        │  Container Runtime (Docker)   │
        │  • Runs actual containers     │
        └───────────────────────────────┘
```

---

## 2. How Kubernetes Actually Works Internally

### Control Plane Components (Detailed)

#### API Server (kube-apiserver)

```
Request Flow:
  kubectl command
    ↓
  HTTP request (REST)
    ↓
  Admission Chain:
    ├─ Authentication (JWT, certs)
    ├─ Authorization (RBAC)
    ├─ Audit logging
    ├─ Mutating webhooks (modify request)
    └─ Validating webhooks (accept/reject)
    ↓
  API Handler (Find resource type)
    ↓
  etcd write/read
    ↓
  Response back to client
```

**Key behavior**: API Server is STATELESS
- No state in memory
- All state in etcd
- Multiple API Server instances work (load balanced)
- If one crashes, others continue
- Restart means reading etcd again

#### Controller Manager (kube-controller-manager)

Runs MULTIPLE controllers simultaneously:

```
Controller Manager:
  ├─ Deployment Controller
  │  └─ Watches: Deployment objects
  │     Action: Create ReplicaSets
  │     
  ├─ ReplicaSet Controller
  │  └─ Watches: ReplicaSets + Pods
  │     Action: Scale up/down pods
  │
  ├─ Service Controller
  │  └─ Watches: Services
  │     Action: Allocate load balancer IPs
  │
  ├─ Node Controller
  │  └─ Watches: Node heartbeats
  │     Action: Mark NotReady, evict pods
  │
  ├─ Endpoint Controller
  │  └─ Watches: Pods + Services
  │     Action: Create Endpoints (which pods get traffic)
  │
  └─ ... 30+ more controllers
```

**Critical insight**: Each controller is independent
- Each has its own reconciliation loop
- Each independently tries to fix state
- Coordination happens through etcd (shared source of truth)
- No inter-controller communication

#### Scheduler (kube-scheduler)

```
Scheduler Loop (every 30ms):
  1. Get list of Pending pods (Pod.spec.nodeName == empty)
  2. For each node:
     a. Filter: Does the node have resources?
     b. Filter: Do affinity rules allow this node?
     c. Filter: Are node taints tolerated?
     d. Score: Which node is best for this pod?
  3. Select highest-scoring node
  4. Write Pod.spec.nodeName = selected_node
  5. kubelet on that node sees the assignment
  6. kubelet creates container
```

**Important**: Scheduler is OPTIMISTIC (best-effort)
- It doesn't reserve resources
- Multiple pods can be scheduled to same node and fail
- kubelet enforces actual resource limits

### Worker Node Components

#### kubelet (node agent)

Runs independent of control plane:

```
kubelet Loop (every 10 seconds):
  1. Read PodSpec from API Server (for this node)
  2. Read current container state (from runtime)
  3. Compare desired vs actual:
     - Missing container? Create it
     - Wrong image? Kill and recreate
     - Container exited? Check restart policy
     - Resource limit exceeded? Kill container
  4. Run probes (ReadinessProbe, LivenessProbe)
  5. Report status back to API Server
  6. Repeat
```

**Critical insight**: kubelet continues working even if API Server crashes
- It keeps running existing pods
- It prevents new pods from starting (can't read specs)
- It reports status to API Server when it comes back online

#### kube-proxy

Maintains network rules on each node:

```
kube-proxy Mode Comparison:

userspace (legacy):
  Pod A → kube-proxy (userspace) → kube-proxy redirects → Pod B
  
iptables (default):
  Pod A → iptables rule (kernel) → Pod B directly
  
ipvs (best for scale):
  Pod A → ipvs (kernel, L4 LB) → Pod B directly
```

---

### Data Flow: Creating a Pod

```
User: kubectl apply -f pod.yaml

↓ kubectl validates locally and sends HTTP POST to API Server

API Server:
  ├─ Decode YAML
  ├─ Run authentication (who are you?)
  ├─ Run authorization (do you have permission?)
  ├─ Run mutating webhooks:
  │  └─ Inject sidecar? Add metadata? Modify image?
  ├─ Run validating webhooks:
  │  └─ Check business rules (e.g., image must start with corporate.io/)
  ├─ Write to etcd: /registry/pods/namespace/pod-name
  └─ Return pod spec to client

↓ Scheduler watches etcd (uses watch API, not polling)

Scheduler:
  ├─ Sees new Pod with empty spec.nodeName
  ├─ Runs filtering/scoring
  ├─ Selects best node (e.g., worker-1)
  └─ Updates etcd: Pod.spec.nodeName = worker-1

↓ kubelet on worker-1 watches etcd

kubelet:
  ├─ Sees new pod assigned to this node
  ├─ Tells container runtime to create container
  ├─ Waits for container to start
  ├─ Runs readiness probe
  ├─ Reports status to API Server: Pod.status.phase = Running

↓ Endpoint Controller watches Services + Pods

Endpoint Controller:
  ├─ Service app has selector: app=web
  ├─ Finds pods with matching labels
  ├─ Updates endpoints: Endpoints.subsets.addresses = [10.0.2.1, 10.0.2.2]

↓ kube-proxy on all nodes updates iptables

kube-proxy:
  ├─ Service "app" has IP 10.0.1.100
  ├─ Endpoints: [10.0.2.1, 10.0.2.2]
  ├─ Updates iptables:
  │  iptables -A KUBE-SVC-APP -j KUBE-EP-APP
  │  iptables -A KUBE-EP-APP -m random -p 50% -j KUBE-MARK-MASQ
  │  iptables -A KUBE-EP-APP -j DNAT --to-destination 10.0.2.1
  │  # Similar for 10.0.2.2
  └─ Result: Traffic to 10.0.1.100:8080 load-balanced to pod endpoints

↓ Everything is ready

Client can now:
  curl http://app:8080
  → DNS resolves app → 10.0.1.100
  → kube-proxy iptables redirects → 10.0.2.1 or 10.0.2.2
  → Success!
```

---

## 3. When to Use vs Not Use

### Architecture Types: When to Choose

**Single Master (Development Only)**
```
✅ Development, learning, prototyping
✓ Easy to understand
✓ 1-10 nodes maximum
✓ Loss of master = cluster down

❌ Production (single point of failure)
```

**High Availability Cluster (Production)**
```
✅ Production critical systems
✓ 3+ master nodes (prevents split-brain)
✓ Load balanced API Server
✓ etcd quorum ensures consensus

Recommended topology:
  Stacked: Masters on worker nodes (cost-efficient, 5-50 nodes)
  Unstacked: Separate master nodes (recommended, >50 nodes)
```

**Managed Kubernetes**
```
✅ GKE, EKS, AKS
✓ Google/AWS/Azure manages control plane
✓ You manage worker nodes only
✓ Better reliability SLA

❌ On-premise if you don't have ops team
```

---

### Architectural Decisions

**API Server Redundancy**
```
Option 1: Single API Server
  Pros: Simple, fewer resources
  Cons: Downtime if crashes

Option 2: Multiple API Servers (2-3)
  Pros: High availability
  Cons: Need etcd cluster too
  
Recommendation: 2-3 for production, 1 for dev/test
```

**etcd Sizing**
```
Small cluster (< 100 nodes):
  Single etcd instance on master
  
Large cluster (> 100 nodes):
  3-node etcd cluster (fault tolerance)
  5-node etcd for massive scale
  
etcd hardware:
  - SSD (non-negotiable for production)
  - Low latency to API Server
  - Network isolated (etcd is critical)
```

**Controller Manager Redundancy**
```
Option 1: Single Controller Manager (standby)
  ├─ Active on node A
  ├─ Standby on node B
  ├─ Use leader election (automatic)
  └─ If A crashes, B takes over in <15s

Option 2: Multiple Controller Managers
  ├─ All running on all nodes
  ├─ Leader election ensures only 1 active
  └─ Prevents conflicts
```

---

## 4. Real-World Production Scenario

### Scenario: Upgrading Kubernetes from 1.27 to 1.28

**Initial State**
- 3 master nodes, 20 worker nodes
- etcd: 3-node cluster, 50GB of state
- 500 pods running across cluster

**Upgrade Strategy**
```
Phase 1: Backup etcd
  - Snapshot etcd data to external storage
  - Verify restore works
  - Keep backup for 30 days

Phase 2: Upgrade one master (staggered)
  - Drain master node (evict pods)
  - Stop API Server, Controller, Scheduler
  - Upgrade kubelet binary on master
  - Start services with new version
  - Verify health checks pass

Phase 3: Repeat Phase 2 for other masters
  - Master B upgrades (API traffic goes to A+C)
  - Master C upgrades (API traffic goes to A+B)
  - All masters now running 1.28

Phase 4: Upgrade worker nodes
  - For each worker node:
    1. Drain node (evict all pods)
    2. Stop kubelet
    3. Update kubelet binary
    4. Start kubelet
    5. Uncordon node (allow new pods)
    6. Wait for pods to reschedule
    7. Verify health

Phase 5: Verify cluster
  - Run integration tests
  - Monitor for API deprecations
  - Check for broken workloads
```

**What Could Go Wrong**

```
❌ Scenario 1: etcd compact not running
   Result: etcd grows to 100GB
   During upgrade: etcd restore takes 2 hours
   Fix: Run "etcdctl compact" regularly

❌ Scenario 2: Pod disruption budgets not set
   Result: Drain evicts all pods from a node
   Impact: Service goes down during upgrade
   Fix: Set PDB to ensure >1 pod always running

❌ Scenario 3: Deprecated API used
   Result: Upgrade succeeds, but workloads break
   Error: "apiVersion batch/v1beta1 no longer supported in 1.28"
   Fix: Audit cluster before upgrade (K8s API

 group migration tools)

❌ Scenario 4: etcd leader election failed
   Result: etcd quorum split, cluster unresponsive
   Fix: Manually fix quorum using etcdctl

Architecture Lesson:
  - You need monitoring, backups, runbooks for upgrades
  - Single master = 2-3 hours downtime
  - 3-master HA cluster = 0 downtime if done right
  - Test upgrade in dev cluster first (mandatory)
```

---

## 5. Architecture Diagrams (Text-Based)

### Multi-Master HA Cluster

```
┌─────────────────────────────────────────────────────────────┐
│                   KUBERNETES CLUSTER                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │          CONTROL PLANE (masters)                  │   │
│  ├────────────────────────────────────────────────────┤   │
│  │                                                    │   │
│  │    Master-1         Master-2         Master-3    │   │
│  │  ┌───────────┐   ┌───────────┐   ┌───────────┐  │   │
│  │  │ API Srvr  │   │ API Srvr  │   │ API Srvr  │  │   │
│  │  │ Ctrl Mgr  │   │ Ctrl Mgr  │   │ Ctrl Mgr  │  │   │
│  │  │ Scheduler │   │ Scheduler │   │ Scheduler │  │   │
│  │  └─────┬─────┘   └─────┬─────┘   └─────┬─────┘  │   │
│  │        │               │               │        │   │
│  │        └───────────────┼───────────────┘        │   │
│  │                        │                        │   │
│  │  ┌─────────────────────▼──────────────────────┐ │   │
│  │  │    etcd cluster (distributed DB)          │ │   │
│  │  │  Master-1  Master-2      Master-3         │ │   │
│  │  │  (follower)(leader)      (follower)        │ │   │
│  │  │                                            │ │   │
│  │  │  All changes replicated via Raft          │ │   │
│  │  │  Quorum: 2/3 must agree on changes        │ │   │
│  │  └────────────────────────────────────────────┘ │   │
│  │                                                    │   │
│  └────────────────────────────────────────────────────┘   │
│                        │                                  │
│         ┌──────────────┼──────────────┐                  │
│         │              │              │                  │
│    ┌────▼────┐    ┌────▼────┐   ┌────▼────┐            │
│    │ Worker  │    │ Worker  │   │ Worker  │            │
│    │ Node-1  │    │ Node-2  │   │ Node-3  │            │
│    ├─────────┤    ├─────────┤   ├─────────┤            │
│    │ kubelet │    │ kubelet │   │ kubelet │            │
│    │ kube-   │    │ kube-   │   │ kube-   │            │
│    │ proxy   │    │ proxy   │   │ proxy   │            │
│    │ ────    │    │ ────    │   │ ────    │            │
│    │ Pod-1   │    │ Pod-3   │   │ Pod-5   │            │
│    │ Pod-2   │    │ Pod-4   │   │ Pod-6   │            │
│    └────────?┘    └────────?┘   └────────?┘            │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Leader Election:
  Only 1 Controller/Scheduler active at a time
  Leader writes lease to etcd every 2 seconds
  If leader dies, new leader elected in 15 seconds
```

### API Request Flow Through Architecture

```
kubectl apply -f deployment.yaml
       │
       ▼
    ┌──────────────────────────────────────────┐
    │      kube-apiserver (port 6443)          │
    ├──────────────────────────────────────────┤
    │ 1. Authentication                        │
    │    ├─ Check certificate                  │
    │    ├─ Verify JWT token                   │
    │    └─ Reject if invalid                  │
    │                                          │
    │ 2. Authorization (RBAC)                  │
    │    ├─ Check if user can "create deployments"
    │    ├─ Check namespace permissions        │
    │    └─ Reject if forbidden                │
    │                                          │
    │ 3. Mutating Webhooks                    │
    │    ├─ Pod security policy injection      │
    │    ├─ Service mesh sidecar injection     │
    │    ├─ Storage provisioner annotations    │
    │    └─ Modify deployment before store     │
    │                                          │
    │ 4. Object Validation                    │
    │    ├─ Schema validation                  │
    │    ├─ Field immutability checks          │
    │    └─ Custom validations                 │
    │                                          │
    │ 5. Validating Webhooks                  │
    │    ├─ PodSecurityPolicy evaluation       │
    │    ├─ Cost optimizer checks              │
    │    └─ Company-specific rules             │
    │                                          │
    │ 6. Write to etcd                        │
    │    └─ Store deployment manifest          │
    └──────────────────────────────────────────┘
                        │
                        ▼
    ┌──────────────────────────────────────────┐
    │  etcd (Deployment stored in db)          │
    │  Key: /registry/deployments/default/app  │
    │  Value: {json deployment spec}           │
    └──────────────────────────────────────────┘
                        │
                        ▼
    ┌──────────────────────────────────────────┐
    │  Deployment Controller (watches etcd)    │
    │                                          │
    │  Sees new deployment → Creates ReplicaSet
    │  Updates etcd with ReplicaSet            │
    └──────────────────────────────────────────┘
                        │
                        ▼
    ┌──────────────────────────────────────────┐
    │  ReplicaSet Controller (watches etcd)    │
    │                                          │
    │  Sees new ReplicaSet replicas: 3         │
    │  Creates 3 Pod objects                   │
    │  Writes Pods to etcd                     │
    └──────────────────────────────────────────┘
                        │
                        ▼
    ┌──────────────────────────────────────────┐
    │  Scheduler (watches etcd for Pending)    │
    │                                          │
    │  Finds 3 unscheduled pods                │
    │  Filters nodes, scores them              │
    │  Assigns to best nodes                   │
    │  Updates Pod.spec.nodeName in etcd       │
    └──────────────────────────────────────────┘
                        │
             ┌──────────┼──────────┐
             │          │          │
             ▼          ▼          ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │ kubelet-1    │ │ kubelet-2    │ │ kubelet-3    │
    │              │ │              │ │              │
    │ Reads Pods   │ │ Reads Pods   │ │ Reads Pods   │
    │ from etcd    │ │ from etcd    │ │ from etcd    │
    │              │ │              │ │              │
    │ Creates      │ │ Creates      │ │ Creates      │
    │ containers   │ │ containers   │ │ containers   │
    │              │ │              │ │              │
    │ Reports      │ │ Reports      │ │ Reports      │
    │ status back  │ │ status back  │ │ status back  │
    └──────────────┘ └──────────────┘ └──────────────┘
```

---

## 6. Common Mistakes

### Mistake 1: Single Master Production Cluster

```yaml
# ❌ WRONG: Single point of failure
Master: 1 instance
  ├─ API Server
  ├─ Controller Manager
  ├─ Scheduler
  └─ etcd
Workers: 10 instances

Problem:
  - Master crashes → Entire cluster unmanageable
  - Can't schedule new pods
  - Can't delete pods
  - Can't update deployments
  - Running pods still work (kubelet independent)
  - But cluster is unavailable for 30-60 minutes (time to fix)

# ✅ CORRECT: HA Masters
Masters: 3 instances (minimum for quorum)
  All running API Server, Controller, Scheduler, etcd
  
Workers: 10 instances

Upgrade strategy:
  - Stop one master at a time
  - Traffic redirects to other 2
  - Zero downtime!
```

---

### Mistake 2: Overloading etcd

```yaml
# ❌ WRONG: No resource limits on API Server
- Large request body: 10MB ConfigMap
- Frequent updates: Pod annotations updated 100x/sec
- No etcd compaction: etcd grows unbounded to 500GB

Result:
  - API Server latency: 500ms (vs normal 50ms)
  - etcd CPU: 100% (stuck)
  - All operations timeout
  - Cluster appears hung

# ✅ CORRECT: Protect etcd
1. Set ResourceQuota to limit per-object size
2. Enable etcd compaction (automated)
3. Regular etcd backups
4. Monitor etcd metrics:
   - etcd_server_has_leader (must be 1)
   - etcd_server_leader_changes_seen (should be 0 usually)
   - etcd_disk_backed_commit_duration_seconds_bucket (should be <100ms)

Compaction:
  etcdctl --endpoints=localhost:2379 compact $(etcdctl --endpoints=localhost:2379 endpoint status | awk '{print $2}' | tail -1 | tr -d ']')
```

---

### Mistake 3: Poor Leader Election Setup

```yaml
# ❌ WRONG: No leader election
Controllers:
  ├─ Node A: Active scheduler
  ├─ Node B: Active scheduler
  └─ Node C: Active scheduler

Problem: All 3 creating pods simultaneously → race conditions

# ✅ CORRECT: Built-in leader election
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-controller-manager-leader-election
  namespace: kube-system

Controllers use ConfigMap to coordinate:
  - Write lease every 2 seconds
  - Only one controller has valid lease
  - Others wait for timeout (15sec)
  - Then try to acquire
```

---

### Mistake 4: Not Monitoring Control Plane

```yaml
# ❌ WRONG: Only monitoring pods/nodes
kubectl get pods  # App pods look fine
kubectl get nodes # Nodes look fine

But API Server is broken:
  - etcd disk 99% full
  - API Server can't write
  - Hidden from kubectl (because kubectl uses API Server!)

# ✅ CORRECT: Monitor control plane components
Prometheus alerts:
  - etcd_server_has_leader != 1
  - apiserver_client_certificate_expiration_seconds_total < 604800 (7 days)
  - etcd_db_total_size_in_bytes > 8589934592 (8GB threshold)
  - etcd_server_leader_changes_seen > 0 (leader election happening)
  - apiserver_request_duration_seconds_bucket[le="+Inf"] > 1sec (slow API)
```

---

## 7. Debugging Scenarios (VERY IMPORTANT)

### Scenario 1: API Server Unresponsive

```bash
# Symptom
$ kubectl get nodes
Unable to connect to the server: dial tcp 10.0.0.1:6443: i/o timeout

# Check 1: Is API Server pod running?
$ kubectl get pods -n kube-system -l component=kube-apiserver
# Error: Can't even run this without API Server!

# SSH to master and check manually:
$ ssh master-1
$ ps aux | grep kube-apiserver
# Check if process running

# Check 2: Is API Server listening?
$ netstat -tulpn | grep 6443
tcp6  0  0 :::6443  :::*  LISTEN  

# Check 3: API Server logs
$ journalctl -u kube-apiserver -n 100
# Look for errors

# Possible causes:
1. etcd down
   $ systemctl status etcd
   $ curl localhost:2379/v3/status
   
2. API Server OOM (out of memory)
   $ journalctl -u kube-apiserver | grep "OOM"
   
3. API certificate expired
   $ openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A 2 "Not After"
   
4. API Server crashed
   $ journalctl -u kube-apiserver -n 50
```

### Scenario 2: Scheduler Not Assigning Pods

```bash
# Symptom
$ kubectl get pods
NAME       STATUS    AGE
app-1      Pending   5m
app-2      Pending   5m
# Pods stuck in Pending for hours

# Debug: Check scheduler logs
$ kubectl logs -n kube-system kube-scheduler-master-1
E0410 12:30:00.123456 12345 factory.go:456] Error getting node
# Scheduler failing to get nodes!

# Check 1: Is scheduler running?
$ kubectl get pods -n kube-system -l component=kube-scheduler
NAME                     READY   STATUS    RESTARTS
kube-scheduler-master-1  1/1     Running   0

# Check 2: Can scheduler access API Server?
$ kubectl -n kube-system exec kube-scheduler-master-1 -- \
  curl -k https://localhost:6443/api/v1/nodes

# Check 3: Are there any nodes?
$ kubectl get nodes
# Should have at least 1 worker node

# Check 4: Scheduler filtering debug
$ kubectl describe pod app-1 | grep Events
Events:
  Type     Reason             Message
  ----     ------             -------
  Warning  FailedScheduling   0/3 nodes available: 3 Insufficient cpu

# Reason: Nodes don't have enough CPU

# Diagnosis:
$ kubectl describe nodes | grep "Allocated resources" -A 5
cpu:  3000m / 4000m (75%)
# All nodes are >70% CPU
```

### Scenario 3: etcd Data Corruption

```bash
# Symptom
$ kubectl get pods
error: the server has received a request with duplicate headers

# This happens when etcd keys are corrupted

# Check 1: etcd health
$ etcdctl endpoint health
127.0.0.1:2379, unhealthy, error=OK, took=25ms  # "OK" but timestamp wrong

# Check 2: Try to read etcd directly
$ etcdctl --endpoints=localhost:2379 get /registry/pods/default/app-1
Error: resource already exists

# Check 3: Defrag etcd
$ etcdctl defrag
Finished defragmenting etcd member

# Check 4: If corruption persists, restore from backup
$ etcdctl snapshot restore etcd-backup-2024-04-10.db --data-dir=/var/lib/etcd-restored

# Check 5: Stop etcd, move data, start with restored
$ systemctl stop etcd
$ rm -rf /var/lib/etcd
$ mv /var/lib/etcd-restored /var/lib/etcd
$ systemctl start etcd
```

### Scenario 4: Controller Manager Has No Replicas

```bash
# Symptom
$ kubectl scale deploy app --replicas=5
# But no new pods created

# Debug: Check if ReplicaSet controller is running
$ kubectl logs -n kube-system -l component=kube-controller-manager | grep -i replica

E0410 12:30:00 replicaset.go:1234] Failed to sync replicaset

# Possible cause: API Server write failures
# ReplicaSet controller tries to create pods but fails

# Check: Can controller manager write pods?
$ kubectl create pod test-pod --image=nginx -o yaml | kubectl apply -f -
error: the server could not find the requested resource (post pods)

# Error creating pod means controller manager also can't

# Check API Server authorization:
$ kubectl get clusterrole system:kube-controller-manager
# Check if it has create/update/patch on pods

# If missing, restore RBAC:
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/saltbase/salt/kube-controller-manager/kube-controller-manager.yaml
```

---

## 8. Interview Questions (15+ Scenario-Based)

### Level 1: Architecture Basics

**Q1**: Explain the difference between control plane and worker nodes. Why are they separate?

**Answer**:
- **Control plane**: Makes decisions (scheduling, monitoring, reconciliation)
- **Worker nodes**: Execute decisions (run containers)

Separation advantages:
- Control plane can be upgraded/rebooted without affecting running pods
- Scalability: 1 control plane can manage 5000+ worker nodes
- Fault isolation: Node failure doesn't affect master
- Security: Control plane isolated from untrusted workloads
- Cost: Small control plane, large worker pool

---

**Q2**: What happens to running pods if the API Server crashes, but no new deployments are made?

**Answer**:
- **Running pods continue running**: kubelet is independent, managed by container runtime
- **New pods can't be scheduled**: kubelet can't read pod specs from API Server
- **Liveness/Readiness probes still run**: kubelet has local copies
- **Pod eviction won't happen**: Node health checks won't be reported
- **Cluster appears frozen** to users (kubectl commands timeout)
- **When API Server recovers**: kubelet syncs state, everything continues normally

---

**Q3**: You have 3 master nodes but all etcd instances fail. What's the cluster state?

**Answer**:
- **Immediate**: API Server can't read/write state, becomes unresponsive
- **Running pods**: Still running (kubelet works independently)
- **New operations**: All fail (can't schedule, scale, update)
- **etcd quorum**: Lost (can't make decisions)
- **Recovery**: Restore from backup (must be from < 5 minutes ago)
- **Lesson**: etcd is THE most critical component, must have:
  - Regular backups (every hour minimum)
  - Redundant storage
  - Quick recovery procedure tested

---

### Level 2: Control Loop & Reconciliation

**Q4**: You manually edit a Pod's CPU limit in etcd (using etcdctl). But kubelet doesn't enforce the new limit. Why?

**Answer**:
- kubelet doesn't read etcd directly (only indirectly via API Server's watch)
- kubelet caches Pod spec in memory
- Changes to etcd aren't automatically propagated to kubelet
- kubelet would need API Server to notify it (watch mechanism)
- **NEVER edit etcd directly** - always use API Server

**Correct approach**: Update Pod spec via kubectl, which:
1. Sends to API Server
2. API Server validates and writes to etcd
3. API Server sends watch notification to kubelet
4. kubelet receives and enforces new limits

---

**Q5**: Scheduler assigned a pod to worker-1, but kubelet on worker-1 never creates the container. What could be wrong?

**Answer** (multiple possibilities):
1. **kubelet crashed on worker-1**
   - Pod stuck in Pending forever
   - Solution: Restart kubelet

2. **kubelet can't pull image**
   - Pod shows ImagePullBackOff
   - Solution: Image registry credentials wrong, or image doesn't exist

3. **Volume mount fails**
   - Pod shows UnmountVolume Failed
   - Solution: PVC doesn't exist or is in wrong namespace

4. **SecurityContext not allowed**
   - Pod shows CreateContainerConfigError
   - Solution: RunAsUser conflicts with PodSecurityPolicy

5. **Node kubelet not syncing with API Server**
   - Pod stuck in Pending with old scheduler
   - Solution: Check kubelet connectivity to API Server

**Debug**:
```bash
$ kubectl describe pod app-1
# Events section shows exact reason

$ kubectl logs app-1 --previous
# If container crashed, shows previous logs
```

---

### Level 3: High Availability

**Q6**: During a master node upgrade, you have 3 masters. One is down for 15 minutes. What happens?

**Answer**:
- **etcd**: Quorum maintained (2/3 nodes still running)
  - Writes still work (replicated to 2 remaining nodes)
  - Performance slightly degraded

- **Controller Manager**: Leader election still works
  - If active controller on master-1, it keeps running
  - If master-1 is the one being upgraded, new leader elected from master-2/3

- **Scheduler**: Same as Controller Manager
  - Leader election ensures one active

- **API Server**: Traffic load-balanced to master-2/3
  - No downtime

- **Result**: Cluster is fully operational with 1 master down

---

**Q7**: All 3 etcd instances simultaneously lose their data (hard disk failure). What's lost?

**Answer**:
- **Everything**: All pod definitions, deployments, services, configs
- **Running pods**: Still running (container runtime managed by kubelet)
- **State**: Completely lost
- **Recovery**: From backup only
- **Lesson**: etcd must have:
  - Regular automated backups (external storage)
  - Backup tested & verified (restore practiced)
  - Retention: At least 7 days

---

### Level 4: Debugging & Operations

**Q8**: A pod is running but stuck in NotReady. The events show "Readiness probe failed". The pod seems fine. What do you check?

**Answer**:
1. **Pod logs for errors**:
   ```bash
   $ kubectl logs app-1
   Error: Database connection refused
   ```

2. **Readiness probe definition**:
   ```bash
   $ kubectl get pod app-1 -o yaml | grep -A 10 readinessProbe
   # Check endpoint, timeout, thresholds
   ```

3. **Can the pod reach the endpoint?**:
   ```bash
   $ kubectl exec -it app-1 -- sh
   / # curl http://localhost:8080/health
   ```

4. **Is the endpoint returning correct status?**:
   ```bash
   $ kubectl exec -it app-1 -- curl -v http://localhost:8080/health
   # Check HTTP status code (must be 200-399)
   ```

5. **Is there a dependency missing?**:
   ```bash
   $ kubectl get svc database
   # Database service doesn't exist
   ```

**Solution**: Fix the application or its dependencies

---

**Q9**: Master node disk is 95% full during an upgrade. New pods can't be scheduled. Which directory is full?

**Answer**:
Most likely: `/var/lib/etcd/` (etcd data)

**Why**: Etcd grows as cluster operates. During upgrade, if not compacted, can grow rapidly.

**Check**:
```bash
$ du -sh /var/lib/etcd
250GB  # Way too large!

$ etcdctl endpoint status
# Check size
```

**Fix**:
```bash
$ etcdctl compact $(etcdctl endpoint status | awk '{print $2}' | tail -1 | tr -d ']')
$ etcdctl defrag
$ etcdctl snapshot save backup.db

# Remove old etcd data and rebuild
$ rm -rf /var/lib/etcd/*
$ etcdctl snapshot restore backup.db --data-dir=/var/lib/etcd
$ systemctl restart etcd
```

---

**Q10**: You upgrade a master node's API Server from 1.27 → 1.28. Now old kubectl (1.27) can't communicate. Why?

**Answer**:
- API Server 1.28 might have removed old API endpoints
- Old kubectl uses old REST endpoints that no longer exist
- API compatibility promise: New server supports old clients (for 1-2 versions)
- But very old clients (1.25) won't work with new server (1.28)

**Standard practice**:
- Upgrade server first, then clients
- Or upgrade clients in parallel
- kubectl version should be ±1 minor version from server

**Check**:
```bash
$ kubectl version
Client Version: v1.27.0
Server Version: v1.28.0

# This is OK (difference of 1 minor version)
```

---

## 9. Advanced Insights (Performance, Cost, Scaling)

### Control Plane Scaling

**Single Master Limitations**:
```
- 1000 nodes: API latency starts degrading
- 5000 nodes: Single master struggles
- 10000+ nodes: Need multiple sharded control planes
```

**Multip master**: Each master can handle ~5000 nodes
```
Cluster size > 5000 nodes:
  - Multiple independent control planes
  - Each manages ~5000 nodes
  - Share same persistent storage (shared etcd)
  - Complex, rarely done (GKE does this internally)
```

### etcd Performance Tuning

```
Bottleneck identification:
1. API Server latency high?
   → Check etcd write latency: etcd_server_leader_sum_slow_commits

2. Specific resource slow?
   → Check object size: avg size > 1MB is bad
   → ConfigMaps should never be > 1MB

3. Leader election issues?
   → Check network latency between etcd members
   → Should be < 5ms (same datacenter required)

Optimization:
- Increase etcd --quota-backend-bytes=8GB (default 2GB)
- Increase --max-request-bytes=33554432 (default 10MB)
- Regular compaction (automated)
- SSD required (HDD causes 10x+ latency)
```

### API Server Rate Limiting

```yaml
# Default limits (shared cluster-wide):
--max-requests-inflight=400          # Concurrent requests
--max-mutating-requests-inflight=200 # Write requests

For large clusters (> 1000 nodes):
--max-requests-inflight=1500
--max-mutating-requests-inflight=500

# User-level rate limiting (per API user):
--enable-priority-and-fairness=true
# Prevents single user from overwhelming API server
```

### Disaster Recovery Architecture

```
Critical components that must be backed up:
  1. etcd (entire database snapshot)
  2. Static pod manifests (/etc/kubernetes/manifests/)
  3. SSL certificates (/etc/kubernetes/pki/)
  4. kubelet config (/etc/kubelet/kubelet.conf)

Backup schedule:
  - etcd: every 1 hour (automated)
  - Certificates: when created/rotated
  - Config: only when changed

Restore procedure:
  1. Understand what was lost
  2. Restore etcd from backup
  3. Verify permissions, certificates
  4. Restart control plane components
  5. Run integration tests
  6. Verify from kubectl

Testing:
  - Regularly test restore in non-prod
  - Time it: should take < 15 minutes
  - Verify data integrity after restore
```

### Multi-Cluster Considerations

When to use multiple clusters:
```
✅ Geographic redundancy (US + EU)
✅ Blast radius containment (one cluster fails, others ok)
✅ Cost optimization (use spot instances in different regions)
✅ Compliance (data residency requirements)
✅ Performance (reduce cross-zone latency)

Setup:
  - Separate kubeconfigs for each cluster
  - Service mesh or cross-cluster ingress
  - Shared storage (if stateful)
  - Synchronized configuration (GitOps)
```

---

## Conclusion

Kubernetes architecture is about **separation of concerns**:
- **Control plane** ↔ **Worker nodes** (independent operation)
- **API Server** ↔ **etcd** (state management)
- **Scheduler** ↔ **kubelet** (distributed scheduling)
- **Controllers** ↔ **State** (reconciliation loops)

Master these concepts and you'll understand any cluster issue!
