# Kubernetes Networking: Deep Dive

## 1. Core Concept (Deep Internal Working)

Kubernetes networking is built on the **network model**:

1. **Pod-to-Pod communication**: Every pod can reach every other pod directly via IP address
2. **Pod-to-Service communication**: Pods reach services via a stable DNS name
3. **External traffic**: Ingress controllers route external HTTP/HTTPS to services
4. **No NAT between pods**: Pod IP remains same across network hops

This contrasts with Docker/VMs where you need port mapping and NAT.

### Three Layers of K8s Networking

```
Layer 1: Container Networking Interface (CNI)
  └─ Assigns IP to each pod
  └─ Ensures pod-to-pod reachability
  └─ Implemented by: Flannel, Calico, Weave, Cilium

Layer 2: Service Abstraction
  └─ Maps Service IP → Pod IPs
  └─ Load balances traffic
  └─ Implemented by: kube-proxy (iptables, IPVS)

Layer 3: Ingress Controller
  └─ Routes HTTP/HTTPS to services
  └─ TLS termination
  └─ Implemented by: Nginx, HAProxy, Traefik
```

### IP Address Ranges (Critical Understanding)

```
Cluster CIDR (Pod IP range):
  10.0.0.0/8  or 10.244.0.0/16 (unique to cluster)
  Every pod gets an IP from this range
  Example: Pod1 = 10.0.2.1, Pod2 = 10.0.2.2

Service CIDR (Service IP range):
  10.1.0.0/16 (different from pod CIDR)
  Every service gets a virtual IP
  Example: Service = 10.1.0.100 (NOT a real interface)

Node IP (Node address):
  192.168.1.0/24 (actual VM/machine IP)
  Real network address where nodes live
  Example: Node1 = 192.168.1.10

CNI Network Interface:
  eth0 (actual network card on node) → 192.168.1.10
  veth* (virtual interfaces in pod namespace) → 10.0.2.1
  br0 (bridge on node) connects veth interfaces
```

---

## 2. How Kubernetes Actually Works Internally

### Networking Stack (Packet Journey)

#### Scenario: Pod A (10.0.2.1) sending traffic to Pod B (10.0.2.2)

**If on same node:**
```
Pod A:
  ├─ App sends traffic to 10.0.2.2:8080
  ├─ Container network stack looks for route
  └─ Finds: 10.0.2.2 is on local bridge (br0)

Node bridge (br0):
  ├─ Forwards packet from vethA to vethB
  └─ No routing needed (same subnet)

Pod B:
  ├─ Receives packet on veth interface
  ├─ App listens on port 8080
  ├─ Sends response back to 10.0.2.1
  └─ Success! (< 1ms latency)
```

**If on different node (Cross-Node):**
```
Pod A (Node1, 10.0.2.1):
  ├─ App sends to 10.0.3.1
  ├─ Container network: "Not local, need to route"
  └─ Sends to default gateway (Node1's IP)

Node1 (192.168.1.10):
  ├─ CNI plugin checks route: 10.0.3.0/24 → 192.168.1.20 (Node2)
  ├─ Encapsulation (if using overlay):
  │  └─ Original packet wrapped in VXLAN tunnel
  │  └─ Outer header: 192.168.1.10 → 192.168.1.20
  └─ Sends to Node2

Network (Physical network):
  └─ Packet travels: 192.168.1.10 → 192.168.1.20

Node2 (192.168.1.20):
  ├─ Receives VXLAN packet
  ├─ Decapsulates (removes outer header)
  ├─ Extracts original Pod A → Pod B packet
  ├─ Routes to Pod B on local bridge
  └─ Pod B receives packet

Response:
  └─ Pod B sends to 10.0.2.1
  └─ Reverse path back to Node1
  └─ Pod A receives response

Latency: 5-50ms depending on network
```

### Service Load Balancing (Most Complex Part)

**Scenario**: Client `curl service-name:8080` where service has 3 pod endpoints

```
Step 1: DNS Resolution
  Client: curl service-name:8080
  DNS (CoreDNS): resolves service-name → 10.1.0.100 (Service IP)
  
Step 2: Client sends
  Source: 10.0.2.1:12345 (pod running curl)
  Destination: 10.1.0.100:8080 (Service IP)
  
Step 3: Node's kube-proxy (iptables mode)
  kube-proxy on node receives packet
  iptables rule matches: "if destination == 10.1.0.100:8080"
  
  Applies chain of rules:
  1. MASQUERADE: Change source IP to Node IP
  2. Random selection: Pick one of 3 endpoints (33% chance each):
     - Option 1: Rewrite to Pod1 (10.0.2.1:8080)
     - Option 2: Rewrite to Pod2 (10.0.2.2:8080)
     - Option 3: Rewrite to Pod3 (10.0.3.1:8080)
  
  Example: Selected Pod2
  Rewritten packet:
    Source: 192.168.1.10:54321 (Node IP, MASQUERADED)
    Destination: 10.0.2.2:8080 (Actual Pod IP)

Step 4: Packet routing to Pod2
  If Pod2 on same node → Direct to veth interface
  If Pod2 on different node → Tunnel via VXLAN/BGP

Step 5: Pod2 receives
  Sees request from 192.168.1.10:54321
  Sends response back to 192.168.1.10

Step 6: kube-proxy reverse NAT
  Response arrives at Node
  kube-proxy recognizes this is a reply
  Rewrites source back to 10.1.0.100:8080
  
Final packet back to client:
  Source: 10.1.0.100:8080 (Service IP)
  Destination: 10.0.2.1:12345 (Original client)

Result: Client thinks it talked to Service IP (0ms latency if local)
```

**Connection Tracking (iptables):**
```
Problem: How does reverse NAT know which Service?
Answer: iptables maintains connection tracking table:

Established connections:
  (10.0.2.1:12345) ← → (10.0.2.2:8080)
  Marked as: "associated with Service 10.1.0.100:8080"

When response arrives (10.0.2.2:8080 → 192.168.1.10:54321):
  iptables checks tracking table
  Finds: This is part of Service session
  Applies reverse NAT
  Source rewritten: 192.168.1.10:54321 → 10.1.0.100:8080
```

### Endpoint Updates (Service Discovery Dynamic)

```
Scenario: Deployment scales from 3 → 5 pods

Event 1: New pods created
  Pods: [Pod1, Pod2, Pod3, Pod4, Pod5]
  
Event 2: Endpoint Controller detects change
  "Service selection changed"
  Updates Endpoints object in etcd

Event 3: kube-proxy watches Endpoints
  "New endpoints: [10.0.2.1, 10.0.2.2, 10.0.3.1, 10.0.3.2, 10.0.3.3]"
  
Event 4: kube-proxy updates iptables rules
  NEW rules include:
    20% → 10.0.2.1
    20% → 10.0.2.2
    20% → 10.0.3.1
    20% → 10.0.3.2
    20% → 10.0.3.3

Timing:
  Endpoint update: Instant (etcd write)
  kube-proxy watch: 1-5 seconds
  iptables update: 100-500ms
  Total: 5 seconds for full traffic distribution
  
During transition:
  Some packets still go to 3 old pods (20% more traffic)
  Other packets go to 5 pods (normal load)
  Brief uneven distribution

Connection draining issue:
  Old pod gets evicted
  Active connections to old pod → DROPPED (not graceful)
  Solution: Use preStop hooks, Pod Disruption Budgets
```

---

## 3. When to Use vs Not Use

### CNI Plugin Selection

| CNI Plugin | When to Use | Pros | Cons |
|-----------|-----------|------|------|
| **Flannel** | Small clusters, simple setup | Easy to setup, low overhead | No network policies, limited features |
| **Calico** | Production, need network policies | Rich policies, great debugging tools | Higher overhead, learning curve |
| **Weave** | Multi-cloud, datacenter | Mesh networking works everywhere | Slower than others |
| **Cilium** | Performance-critical, eBPF capable | Fastest, advanced features, API-aware | Complex, requires newer kernels |
| **AWS VPC CNI** | EKS on AWS | Native AWS networking, no overlay | Only for AWS, not portable |

### Service Type Selection

| Service Type | When to Use | Typical Scenario |
|--------------|-----------|------------------|
| **ClusterIP** (default) | Pod-to-pod communication within cluster | Internal APIs, databases, cache |
| **NodePort** | External access without load balancer | Dev/test, emergency access |
| **LoadBalancer** | Cloud load balancer integration | External APIs in production |
| **ExternalName** | DNS alias to external service | Legacy systems, third-party APIs |

### Network Policy Strategy

```yaml
# Restrictive (recommended for production)
- Default deny all (deny-all policy)
- Allow specific flows explicitly

# Permissive (development)
- Allow all (default)
- Add restrictions as needed
```

---

## 4. Real-World Production Scenario

### Scenario: Multi-Region Traffic Routing

**Setup**
- 2 regions: US-East, EU-West
- Each region has independent cluster (different etcd, control plane)
- Shared DNS for global load balancing
- 200ms latency between regions

**Architecture**
```
Global DNS (Route53 / CloudDNS)
  ├─ api.example.com
  └─ Resolves to:
     - US-East Ingress IP: 1.1.1.1 (40% traffic by geo-proximity)
     - EU-West Ingress IP: 2.2.2.2 (40% traffic by geo-proximity)
     - Fallback: Both accept all traffic
```

**Within US-East Cluster**
```
Traffic arrives at Ingress Controller:
  ├─ 1.1.1.1:443 (Nginx Ingress running on Node)
  │
  ├─ Terminates TLS
  ├─ Adds header: X-Original-IP
  ├─ Routes to Service based on:
     - Host (app.example.com → service-app)
     - Path (/api/* → service-api, /admin/* → service-admin)
  │
  ├─ Service: app-service (ClusterIP 10.1.0.100)
  │
  ├─ kube-proxy load balances to:
     - app-pod-1 (10.0.2.1)
     - app-pod-2 (10.0.2.2)
     - app-pod-3 (10.0.3.1)
  │
  ├─ Pod receives request
  ├─ Processes and sends response
  │
  └─ Response: 10.0.2.1 → Ingress → Client
```

**Production Issues That Arose**

```
❌ Issue 1: Connection storms on scale-down
   When pods terminate, active connections dropped
   
   Solution:
   apiVersion: apps/v1
   kind: Deployment
   spec:
     strategy:
       type: RollingUpdate
       rollingUpdate:
         maxUnavailable: 1
     template:
       spec:
         terminationGracePeriodSeconds: 60  # Give pods time to drain
         containers:
         - name: app
           lifecycle:
             preStop:
               exec:
                 command: ["/bin/sh", "-c", "sleep 30"]  # Give LB time to remove

❌ Issue 2: DNS TTL too high (3600s)
   After failover to EU-West, US clients still routed to dead endpoint
   
   Solution:
   Set DNS TTL to 30 seconds
   Update Route53 health checks to 10 seconds
   Use connection draining (timeout: 30s)

❌ Issue 3: Service IP not routable outside cluster
   External load balancer tried to health check 10.1.0.100 (Service IP)
   Failed (virtual IP, not real)
   
   Solution:
   Health check against Node port (30000-32767)
   Or use Endpoint health check directly

❌ Issue 4: Asymmetric routing
   Request: 1.1.1.1 → 10.0.2.1
   Response: 10.0.2.1 → 2.2.2.2 (goes to EU!)
   
   Cause: Pod's default gateway is different node
   Solution: Fix CNI plugin routing or use ExternalTrafficPolicy: Local

❌ Issue 5: High latency spikes (1000ms p99)
   Root cause: Large pod shuffling during deployment
   
   Causes:
   - 100 pods evicted at once
   - All scramble to find new nodes
   - Scheduler overloaded
   - New pods pending for 5+ seconds
   - No capacity to handle requests
   
   Solution:
   maxUnavailable: 10% (more gradual)
   Pod Disruption Budgets (minAvailable: 80%)
   Cluster scaling in advance
```

---

## 5. Architecture Diagrams (Text-Based)

### Pod Networking (CNI Level)

```
┌──────────────────────────────────────────────────────────┐
│                      WORKER NODE                         │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Host Network Namespace                         │   │
│  │                                                 │   │
│  │  eth0 (physical NIC) ← 192.168.1.10            │   │
│  │  br0 (bridge)     ← 10.0.2.254 (virtual)       │   │
│  │  │                                              │   │
│  │  ├─ veth1 → (to Pod1)                           │   │
│  │  ├─ veth2 → (to Pod2)                           │   │
│  │  └─ veth3 → (to Pod3)                           │   │
│  │                                                 │   │
│  │  iptables rules (kube-proxy management)        │   │
│  │  conntrack table (connection tracking)         │   │
│  └─────────────────────────────────────────────────┘   │
│         │           │           │                       │
│         ▼           ▼           ▼                       │
│    ┌─────────┐ ┌─────────┐ ┌─────────┐                │
│    │ Pod 1   │ │ Pod 2   │ │ Pod 3   │                │
│    │ NS      │ │ NS      │ │ NS      │                │
│    ├─────────┤ ├─────────┤ ├─────────┤                │
│    │ veth1   │ │ veth2   │ │ veth3   │                │
│    │(peer)   │ │(peer)   │ │(peer)   │                │
│    │         │ │         │ │         │                │
│    │ eth0    │ │ eth0    │ │ eth0    │                │
│    │ 10.0.2.1│ │ 10.0.2.2│ │ 10.0.2.3│                │
│    │         │ │         │ │         │                │
│    │ loopback│ │ loopback│ │ loopback│                │
│    │ 127.0.0.1│ │ 127.0.0.1│ │ 127.0.0.1│                │
│    └─────────┘ └─────────┘ └─────────┘                │
│        │           │           │                       │
│        │ App       │ App       │ App                   │
│        ▼           ▼           ▼                       │
│     app:8080    app:8080    app:8080                  │
│                                                          │
└──────────────────────────────────────────────────────────┘

How packets flow between Pod1 and Pod2:
  Pod1 → veth1 → br0 → veth2 → Pod2
  (All in kernel, < 100 microseconds)
```

### Service Load Balancing (kube-proxy)

```
┌────────────────────────────────────────────────────────────┐
│                   SERVICE ABSTRACTION                      │
└────────────────────────────────────────────────────────────┘

Service Resource (api-service):
  Name: api-service
  Namespace: default
  ClusterIP: 10.1.0.100
  Port: 8080
  Selector: app=api

Endpoints Resource (auto-generated):
  Name: api-service
  Subsets:
    - addresses: [10.0.2.1, 10.0.2.2, 10.0.3.1]  ← actual pods
      ports: [8080]

kube-proxy translates this to:
┌────────────────────────────────────────────────────────────┐
│              IPTABLES RULES (on every node)               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ Jump to service-specific chain:                           │
│   if (dest == 10.1.0.100:8080) → KUBE-SVC-API            │
│                                                            │
│ Service chain:                                            │
│   KUBE-SVC-API:                                           │
│     - Random (33%): → KUBE-EP-API-1 (10.0.2.1:8080)      │
│     - Random (33%): → KUBE-EP-API-2 (10.0.2.2:8080)      │
│     - Random (33%): → KUBE-EP-API-3 (10.0.3.1:8080)      │
│                                                            │
│   KUBE-EP-API-1: MASQUERADE + DNAT 10.1.0.100 → 10.0.2.1 │
│   KUBE-EP-API-2: MASQUERADE + DNAT 10.1.0.100 → 10.0.2.2 │
│   KUBE-EP-API-3: MASQUERADE + DNAT 10.1.0.100 → 10.0.3.1 │
│                                                            │
└────────────────────────────────────────────────────────────┘

packet transformation:
  Before: 10.0.2.5:12345 → 10.1.0.100:8080
  After:  192.168.1.10:54321 → 10.0.2.1:8080
```

### Ingress Controller (External Traffic)

```
┌────────────────────────────────────────────────────────────┐
│              EXTERNAL TRAFFIC FLOW                         │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Client: curl https://api.example.com/data               │
│    │                                                       │
│    ▼                                                       │
│  DNS: api.example.com → 1.1.1.1 (Ingress LB IP)          │
│    │                                                       │
│    ▼                                                       │
│  External Load Balancer: 1.1.1.1:443                     │
│    │                                                       │
│    ▼                                                       │
│  ┌──────────────────────────────────────────────────────┐ │
│  │   INGRESS CONTROLLER (Nginx Pod)                    │ │
│  ├──────────────────────────────────────────────────────┤ │
│  │ 1. Terminates TLS (has ssl certificate)             │ │
│  │ 2. Reads Host header: api.example.com               │ │
│  │ 3. Looks up Ingress rules:                          │ │
│  │    Ingress: api-ingress                             │ │
│  │    Rules:                                           │ │
│  │      - host: api.example.com                        │ │
│  │        http:                                        │ │
│  │          paths:                                     │ │
│  │            - path: /data                            │ │
│  │              backend:                               │ │
│  │                service: api-service (10.1.0.100:80) │ │
│  │                                                      │ │
│  │ 4. Route to Service: 10.1.0.100:8080                │ │
│  │ 5. kube-proxy load balances to pod IP               │ │
│  │ 6. Pod processes and responds                       │ │
│  │                                                      │ │
│  │ 7. Ingress controller sends response back with:     │ │
│  │    - Original Source: Client IP (X-Forwarded-For)   │ │
│  │    - Original Host: api.example.com (Host header)   │ │
│  └──────────────────────────────────────────────────────┘ │
│    │                                                       │
│    ▼                                                       │
│  Response: 200 OK {user_data}                            │
│    │                                                       │
│    ▼                                                       │
│  Client receives                                          │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 6. Common Mistakes

### Mistake 1: Exposing Service Port Without Port/TargetPort

```yaml
# ❌ WRONG
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
  - port: 8080
    # Missing targetPort! Defaults to port (8080)
    # But pod listens on 3000!

# Result: kube-proxy routes 10.1.0.100:8080 → 10.0.2.1:8080
# Pod listens on 3000, connection refused!

# ✅ CORRECT
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
  - port: 8080              # External port
    targetPort: 3000        # Pod's actual port
    protocol: TCP
```

---

### Mistake 2: Using Service IP in pod-to-pod communication

```yaml
# ❌ SUBOPTIMAL (works but adds latency)
# Pod A connects to Pod B via Service IP
curl http://api-service:8080
# Traffic: Pod A → Service (10.1.0.100) → kube-proxy → Pod B

# Extra hops: +5-10ms latency due to DNAT

# ✅ BETTER (direct pod-to-pod)
curl http://pod-b-name.default.svc.cluster.local:3000
# Traffic: Pod A → Pod B directly (no service)
# Better for latency-sensitive workloads

# When to use Service:
  - Load balancing needed
  - Multiple replicas behind same endpoint
  - Pod IPs ephemeral (change on restart)
  - DNS remains stable (service DNS doesn't change)

# When to use direct DNS:
  - Lowest latency required
  - Specific pod targeting (ordered init, leader election)
  - StatefulSets with headless service
```

---

### Mistake 3: Cross-Namespace Service Communication

```yaml
# ❌ WRONG
# In pod from namespace A trying to reach service in namespace B
curl http://service-name:8080
# This resolves to service-name.A.svc.cluster.local (wrong namespace)

# ✅ CORRECT
curl http://service-name.namespace-b.svc.cluster.local:8080
# FQDN includes namespace
# pod-to-pod works across namespaces if network policy allows

# Or via environment variable:
curl http://$SERVICE_HOST:$SERVICE_PORT
# Values injected from namespace B service
```

---

### Mistake 4: No Network Policies (Open Cluster)

```yaml
# ❌ DEFAULT: No network policies
# All pods can reach all other pods
# All namespaces accessible from each other
# Major security risk!

# ✅ PRODUCTION: Default Deny Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}  # Applies to all pods in namespace
  policyTypes:
  - Ingress       # No incoming traffic allowed
  - Egress        # No outgoing traffic allowed

# Then explicitly allow needed connections:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-from-web
spec:
  podSelector:
    matchLabels:
      app: api      # Target pods with this label
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web  # Only allow from pods with this label
    ports:
    - protocol: TCP
      port: 8080
```

---

### Mistake 5: DNSPolicy Causing External DNS Failures

```yaml
# ❌ WRONG: Default policy has limitations
# ClusterFirst policy looks in cluster DNS first
# If not found in cluster, uses host's /etc/resolv.conf
# Problem: Host DNS might not be configured in pod!

# ✅ CORRECT: Explicit DNS policy
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  dnsPolicy: ClusterFirst  # Default, OK for most cases
  
  # OR for specific needs:
  # dnsPolicy: Default          (use host's /etc/resolv.conf)
  # dnsPolicy: ClusterFirstWithHostNet  (headless pods)
  # dnsPolicy: None             (explicit nameservers only)
  
  dnsConfig:  # Custom DNS settings
    nameservers:
    - 8.8.8.8
    - 8.8.4.4
    searches:
    - svc.cluster.local
    - example.com
    options:
    - name: ndots
      value: "2"
```

---

## 7. Debugging Scenarios (VERY IMPORTANT)

### Scenario 1: Pod can't reach external service

```bash
# Symptom
$ kubectl exec -it app-1-abcde -- curl http://external-api.example.com
curl: (28) Connection timed out after 30 seconds

# Debug Step 1: Check if DNS works
$ kubectl exec -it app-1-abcde -- nslookup external-api.example.com
# If DNS lookup fails, several causes

# Cause 1: CoreDNS pod down
$ kubectl get pods -n kube-system -l k8s-app=kube-dns
# or
$ kubectl get pods -n kube-system -l k8s-app=kube-dns-autoscaler

# If no pods:
$ kubectl get events -n kube-system | grep -i dns

# Cause 2: DNS service not responding
$ kubectl exec -it app-1-abcde -- nslookup kubernetes.default
# If internal DNS works but external doesn't,
# check /etc/resolv.conf in pod

$ kubectl exec -it app-1-abcde -- cat /etc/resolv.conf
nameserver 10.0.0.10  # CoreDNS service IP
search default.svc.cluster.local svc.cluster.local cluster.local

# Debug Step 2: Manual DNS query
$ kubectl exec -it app-1-abcde -- dig @10.0.0.10 external-api.example.com
# Check if query reaches CoreDNS

# Debug Step 3: Check network connectivity
$ kubectl exec -it app-1-abcde -- ping external-api.example.com
# If DNS works but ping fails, network routing issue

# Debug Step 4: Check egress network policies
$ kubectl get networkpolicies
# If any egress policy exists, default-deny might block external traffic

# Required NetworkPolicy for external access:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-dns
spec:
  podSelector:
    matchLabels:
      app: app
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}  # All namespaces
    ports:
    - protocol: UDP
      port: 53  # DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443  # HTTPS to external
    - protocol: TCP
      port: 80   # HTTP to external
```

### Scenario 2: Service has no endpoints

```bash
# Symptom
$ kubectl get endpoints service-name
NAME            ENDPOINTS   AGE
service-name    <none>      5m

# Debug Step 1: Check service selector
$ kubectl get svc service-name -o yaml | grep -A 5 selector
selector:
  app: web
  version: v1

# Debug Step 2: Check if pods with label exist
$ kubectl get pods --show-labels | grep -E "app=web.*version=v1"
# If no results, labels don't match

# Debug Step 3: Pod labels
$ kubectl get pod specific-pod -o yaml | grep -A 5 labels
labels:
  app: web
  version: v2  # Doesn't match "v1"!

# Fix: Either update pod labels or update service selector

# Debug Step 4: Check if pod is ready
$ kubectl get pods -o wide
NAME          READY   STATUS    RESTARTS
specific-pod  0/1     Running   0  # NOT READY!

# Pod is running but readiness probe failing
$ kubectl describe pod specific-pod | grep -A 10 "Readiness"
Readiness probe failed!

# Fix readiness probe:
$ kubectl logs specific-pod
[ERROR] Database connection failed

# Pod can't start properly

# Debug Step 5: If pod ready but no endpoints still
$ kubectl get endpoints service-name -o yaml
# Check if subsets.addresses is empty

# This shouldn't happen if:
# - Pod labels match
# - Pod is ready
# - Service selector matches

# Possible cause: Endpoint controller broken
$ kubectl logs -n kube-system -l component=kube-controller-manager | grep -i endpoint
```

### Scenario 3: High latency between pods on same node

```bash
# Symptom
$ kubectl exec -it pod1 -- ping pod2
64 bytes from pod2: time=50ms  # Should be <1ms

# Debug Step 1: Verify pods on same node
$ kubectl get pods -o wide
NAME       NODE
pod1       worker-1
pod2       worker-1

# Same node, should be local bridge, should be fast

# Debug Step 2: Check CNI bridge
$ ssh worker-1
$ brctl show  # Show bridge connections
bridge name     bridge id           STP enabled   interfaces
br0             8000.da1c2b5d6e7f   no            vethXXXX vethYYYY

# Debug Step 3: Check iptables rules
$ iptables -L -n -v | grep pod1-pod2
# High packet counts on slow rules indicate bottleneck

# Debug Step 4: Check if traffic is being tunneled unnecessarily
$ tcpdump -i br0 -n host pod1-ip and host pod2-ip
# Should see direct communication

# If seeing VXLAN packets:
4499 src 192.168.1.10 dst 192.168.1.10
# Tunnel to same node = misconfiguration!

# Fix: Check CNI configuration
$ cat /etc/cni/net.d/*
# Verify backend settings, should not create tunnel for same-node

# Debug Step 5: Check MTU (packet fragmentation)
$ ip link show br0
mtu 1450  # Smaller than usual 1500, might cause fragmentation

# Test with specific packet size
$ kubectl exec -it pod1 -- ping -M do -s 1400 pod2
# If fails, MTU issue

# Fix: Increase MTU or use smaller packets
```

### Scenario 4: Service loadbalancing very uneven (80% traffic to one pod)

```bash
# Symptom
$ kubectl logs pod1 | wc -l
50000  # High traffic

$ kubectl logs pod2 | wc -l
10000  # Much lower

$ kubectl logs pod3 | wc -l
10000  # Much lower

# Should be roughly 33% each

# Debug Step 1: Check kube-proxy mode
$ ps aux | grep kube-proxy
--proxy-mode=iptables

# Check connection tracking
$ conntrack -L | grep service-ip | wc -l
1000  # Number of active connections

# Each connection randomly picks endpoint
# But established connections stick to same endpoint!

# Debug Step 2: Long-lived connections
# If clients using connection pooling:
  Connection pool connects to pod1 → sticks there
  Pool reuses same connection → all traffic to pod1
  
# Solution: Use session affinity
  apiVersion: v1
  kind: Service
  spec:
    sessionAffinity: None        # Default, random per packet
    # OR
    sessionAffinity: ClientIP    # Stick connection to pod

# Debug Step 3: Service has no persistent TCP
# If pods reconnecting frequently:
  Older connections keep hitting same pod
  New connections randomly pick
  Eventual rebalancing happens
  
# Debug Step 4: IPVS vs iptables
# Switch to IPVS for better balancing
$ kube-proxy --proxy-mode=ipvs
# IPVS uses consistent hashing (better distribution)

# Debug Step 5: Health check backend health
$ kubectl describe endpoints service-name
# Check if all endpoints marked ready
# If one endpoint not ready but still in list, bug in endpoint controller
```

---

## 8. Interview Questions (15+ Scenario-Based)

### Level 1: Fundamentals

**Q1**: Explain how a pod gets its IP address.

**Answer**:
1. Pod created (via API Server)
2. kubelet receives pod spec
3. kubelet creates container via container runtime (Docker/containerd)
4. CNI plug-in is invoked by kubelet
5. CNI checks local IPAM (IP Address Management)
6. Allocates IP from pod CIDR (e.g., 10.0.2.1)
7. Creates virtual network interface (veth pair)
8. Attaches to bridge (br0)
9. Returns IP to kubelet
10. kubelet updates Pod status with IP

Total time: 200-500ms

---

**Q2**: Can two pods on the same node have the same IP address?

**Answer**:
- No, IPs must be unique within the cluster
- CNI maintains IPAM state (who has which IP)
- If IPAM misconfigured or data lost, duplicate IPs = disaster
- Each pod's veth interface has unique MAC address (virtual NIC)
- Network bridge uses MAC address to forward, so even with duplicate IPs, could work at L2
- But DNS breaks (same IP, multiple pods)
- Solution: IPAM must be reliable (state persisted)

---

### Level 2: Service & Load Balancing

**Q3**: Why does kube-proxy use MASQUERADE (source NAT) instead of direct routing?

**Answer**:
- Direct routing: Pod1 → Pod2, Pod2 responds directly to Pod1
- But Pod2 says "response from 10.0.2.2" (its own IP)
- Pod1 configured to reach 10.1.0.100 (service IP)
- Pod1: "Response says 10.0.2.2, but I requested 10.1.0.100!"
- TCP resets connection (IP mismatch)

Solution: MASQUERADE
- kube-proxy rewrites source: 10.0.2.1 → 192.168.1.10 (node IP)
- Pod2 responds to 192.168.1.10
- kube-proxy reverse-NAT: 192.168.1.10 → 10.0.2.1
- Pod1 receives from expected source

Without MASQUERADE: asymmetric routing = broken

---

**Q4**: What happens to active connections when a pod is evicted from a node?

**Answer**:
- Pod receives SIGTERM signal
- Grace period: 30 seconds (default)
- Application should close connections gracefully
- New connections rejected (kube-proxy removes from endpoints)
- Active connections:
  - If app handles SIGTERM: connections drained, closed properly
  - If app ignores SIGTERM: after 30s, SIGKILL sent
  - kube-proxy removes pod from endpoints
  - In-flight packets: dropped (no destination)
- Result: Client receives "Connection reset by peer"

Prevention: Use preStop hooks, Pod Disruption Budgets

---

### Level 3: Advanced Networking

**Q5**: You have a Deployment with 100 replicas. Suddenly network latency spikes to 500ms p99. What could cause this?

**Answer** (multiple causes):
1. **kube-proxy updating rules**: iptables lock contention
   - 100+ pods all updating simultaneously
   - Single-threaded iptables lock
   - Solution: Switch to IPVS mode

2. **Connection tracking table full**: conntrack -L shows 1M+ entries
   - Each connection tracked in memory
   - Hashtable collisions when full
   - Solution: Increase net.netfilter.nf_conntrack_max

3. **Pod disruption during deployment**: 
   - Eviction storms cause new pod creation
   - Scheduler overloaded
   - Network becomes saturated with traffic rerouting

4. **CNI plugin saturated**:
   - VXLAN encap/decap CPU-bound
   - Or overlay network congested

5. **Container runtime issues**:
   - Docker daemon slow (disk I/O on /var/lib/docker)
   - All pod creations queued

Diagnosis:
```bash
$ watch 'conntrack -L | wc -l'  # Track connection count
$ sar -n TCP1  # Network stats
$ top -p $(pgrep kube-proxy)  # CPU usage
```

---

**Q6**: You route external traffic via Ingress controller. A client reports occasional 5-second delays. What's the issue?

**Answer** (most likely):
- **Pod eviction during rolling update**:
  - Ingress controller pod evicted
  - Endpoint removed from load balancer
  - Requests land on other Ingress pod (cold cache)
  - First request: 5s delay

- Other causes:
  1. SSL handshake timeout (cert validation)
  2. DNS TTL expired (re-lookup)
  3. Pod startup probe delay
  4. Readiness probe on new pod failing

Prevention:
```yaml
terminationGracePeriodSeconds: 60  # Drain connections
  lifecycle:
    preStop:
      exec:
        command: ["/bin/sh", "-c", "sleep 15"]
```

---

### Level 4: Production Scenarios

**Q7**: During promotion to production, you notice that from outside the cluster, pods can directly communicate with database pods (security risk). How do you fix this?

**Answer**:
1. **Implement Network Policy**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-database
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: middleware  # Only allow from middleware tier
    ports:
    - port: 5432
```

2. **Isolate database**:
   - Place in separate namespace
   - No external ingress controller routes to database
   - Only app pods can access via service DNS

3. **Use network policies for default deny**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-external
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress: []  # Deny all
  # Then add explicit allow rules
```

---

**Q8**: Your API service has 10 replicas. During scale-up to 50, some clients report "Connection refused". Why?

**Answer**:
- New pods created (pending)
- Scheduler assigns to nodes
- kube-proxy watches Endpoints
- Ingress load balancer not in sync!

Timeline:
```
T0: Endpoint update: [pod1-pod10]
T1: kube-proxy updates iptables: [pod1-pod10]
T2: Ingress controller watches (30s old): still [pod1-pod10]
T3: New pods becoming ready: [pod1-pod50]
T4: Ingress LB sees endpoint update (5s update cycle)
T5: Ingress updates its backends

During T0-T4: New load balancer traffic going somewhere
```

Solution:
1. HPA must be gradual: maxSurge: 25%
2. Pod readiness delay: minReadySeconds: 10
3. Ingress sync faster: reduce sync interval
4. Service circuit breaker: retry failed connections

---

## 9. Advanced Insights (Performance, Cost, Scaling)

### Network Plugin Performance

```yaml
Performance ranking (in single-node benchmarks):
1. IPVS mode (kube-proxy)    - < 100μs per connection
2. Cilium nativeRouting      - < 200μs per connection  
3. Calico BGP                - < 500μs per connection
4. iptables mode             - < 1ms per connection
5. Flannel VXLAN             - < 2ms per connection
6. Weave                     - < 5ms per connection

Notes:
- At scale (1000+ pods), iptables contention increases latency 10x
- VXLAN overhead: 50 bytes encapsulation per packet
- Per-packet latency + throughput throughput: tradeoff
```

### Scaling Patterns

```
Small cluster (< 100 nodes):
  - Single CNI plugin sufficient
  - Default kube-proxy (iptables) OK
  - DNS: Single CoreDNS pod sufficient

Medium cluster (100-500 nodes):
  - Consider IPVS (services > 1000)
  - Multiple CoreDNS replicas (3-5)
  - Monitor kube-proxy resource usage

Large cluster (500-5000 nodes):
  - Must use IPVS (iptables unusable)
  - Multiple Ingress controllers (HA)
  - Distributed CoreDNS (1 per node)
  - Network policy controller replicas for stability
```

### DNS Scaling

As cluster grows, CoreDNS becomes bottleneck:

```yaml
# Default: 2 CoreDNS pods
$ kubectl get deployment -n kube-system coredns -o yaml

# Under load: scale to 1 per 100 nodes
minReplicas: max(2, cluster_nodes / 100)

# Enable autohealing:
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: coredns
  namespace: kube-system
spec:
  scaleTargetRef:
    kind: Deployment
    name: coredns
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## Conclusion

Kubernetes networking is elegantly simple at the pod level but complex at scale. Master:
1. **Pod-to-pod**: CNI ensures every pod reaches every pod
2. **Service abstraction**: kube-proxy makes load balancing transparent
3. **Ingress**: Controllers route external traffic to services
4. **Network policies**: Control which pods can communicate

Performance and reliability follow from understanding these three layers.
