# Kubernetes Security: Deep Dive

## 1. Core Concept (Deep Internal Working)

Kubernetes security operates on **defense in depth** principle: multiple layers prevent different attack vectors.

### Security Layers

```
Layer 1: Host/Node Security
  ├─ OS-level isolation (SELinux, AppArmor)
  ├─ SSH key rotation
  └─ Node patching & hardening

Layer 2: Container Runtime Security
  ├─ Container images (vulnerability scanning)
  ├─ Image registry authentication
  └─ Runtime enforcement (seccomp, AppArmor)

Layer 3: Kubernetes API Security
  ├─ Authentication (who are you?)
  ├─ Authorization (what can you do?)
  ├─ Admission control (is this allowed?)
  └─ Audit logging (what happened?)

Layer 4: Network Security
  ├─ Network policies (pod-to-pod firewall)
  ├─ Service mesh (mutual TLS, traffic rules)
  └─ Ingress TLS (external traffic encryption)

Layer 5: Application Security
  ├─ Secret management (encryption at rest)
  ├─ RBAC (least privilege)
  └─ Pod security policies (container restrictions)
```

### Authentication Flow

```
User executes: kubectl get pods

Signal from client:
  1. Load kubeconfig: ~/.kube/config
  2. Find credentials: certificate + key
  3. Client certificate: Signed by cluster's root CA
  4. Mutual TLS: Client verifies API Server's cert
  5. Client sends cert + key with request

API Server receives:
  1. Extracts client certificate
  2. Validates signature (is this signed by our CA?)
  3. Extracts CommonName and Organization from cert
     Example: CN=alice, O=engineering, O=team-a
  4. Sets X-Remote-User = "alice"
  5. Sets X-Remote-Groups = ["engineering", "team-a"]
  6. Passes to next phase: Authorization
  
Result: Username and Groups headers attached to request
```

---

## 2. How Kubernetes Actually Works Internally

### RBAC (Role-Based Access Control)

**Components**:
```
Role/ClusterRole:
  └─ Define WHAT actions are allowed
  └─ Example: "create", "read", "update", "delete" on "pods"

RoleBinding/ClusterRoleBinding:
  └─ Bind ROLES to USERS/GROUPS/SERVICE ACCOUNTS
  └─ Example: Bind developer-role to group "developers"

Reconciliation:
  User: "alice" in group "developers"
    ↓
  API Server checks all RoleBindings
    ↓
  Finds: RoleBinding "developer-binding" → developer-role
  Matches user alice → found in group "developers"
    ↓
  Loads developer-role permissions:
    - verbs: [get, list, watch, create, update, patch]
    - resources: [pods, deployments, services]
    - apiGroups: ["", "apps", ""]
    ↓
  User's action: "create deployment"
  Matches: "create" in verbs ✅
  Matches: "deployments" in resources ✅
  Matches: "apps" in apiGroups ✅
  Result: ALLOWED ✅

If any mismatch → FORBIDDEN
```

**Four-Level Scope**:
```
ClusterRole + ClusterRoleBinding:
  └─ Global permissions across entire cluster
  └─ Example: Cluster admin, system components

Role + RoleBinding:
  └─ Namespace-scoped permissions
  └─ Example: Developers can only access their namespace

Result:
  alice can: create pods in namespace "team-a"
  alice cannot: create pods in namespace "team-b" (different namespace binding)
  alice cannot: delete nodes (cluster-scoped, not namespace)
```

### Pod Security

**SecurityContext** (per pod/container):
```yaml
spec:
  securityContext:  # Pod-level
    runAsUser: 1000              # UID
    runAsNonRoot: true           # Forbid UID 0 (root)
    fsGroup: 2000                # GID for volumes
    seLinuxOptions:
      level: "s0:c123,c456"      # SELinux labels
    supplementalGroups: [3000]   # Additional GIDs
    sysctls:
    - name: net.ipv4.ip_unprivileged_port_start
      value: "0"                 # Allow ports < 1024
  
  containers:
  - name: app
    securityContext:  # Container-level (overrides pod)
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
      privileged: false
      runAsUser: 1001
```

**How it works**:
```
Container Runtime (containerd/CRI):
  1. Receives pod spec including SecurityContext
  2. Creates cgroup with resource limits
  3. Sets UID/GID for container process
  4. Applies capabilities (Linux capabilities restrictions)
  5. Sets SELinux context if specified
  6. Mounts filesystem per readOnlyRootFilesystem
  7. Starts process with these restrictions

Example process in container:
  $ ps aux
  UID  PID  COMMAND
  1000 1    /app /main  ← Running as UID 1000 (not root)

If attempted: setuid("root") → DENIED (no CAP_SETUID)
   Result: Process can't escalate privileges
```

### Secrets Encryption (at rest)

**Without encryption** (dangerous):
```
$ etcdctl get /registry/secrets/namespace/my-secret
{
  "apiVersion": "v1",
  "kind": "Secret",
  "data": {
    "password": "cGFzc3dvcmQgPSBmaG9vYmFy"  ← BASE64, NOT ENCRYPTED!
  }
}

Anyone with file system access to etcd can:
  1. Read etcd data files
  2. Base64 decode
  3. Get plaintext secrets!
```

**With encryption** (correct):
```yaml
# /etc/kubernetes/encryption/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: O2e3K0x6K0e3K0x6K0e3K0x6K0e3K0x6= (32 bytes)
    - identity: {}  # Fallback: non-encrypted
      
# Start API Server with:
kube-apiserver --encryption-provider-config=/etc/kubernetes/encryption/encryption-config.yaml

Result:
$ etcdctl get /registry/secrets/namespace/my-secret
<binary encrypted data>
└─ Only API Server can decrypt (has keys in memory)
└─ Unreadable without API Server
```

**Lifecycle**:
```
1. User creates Secret
2. API Server encrypts with key1
3. Stored encrypted in etcd
4. User retrieves Secret
   ├─ API Server reads encrypted data
   ├─ Decrypts with key1
   ├─ Returns plaintext to user
5. User can use Secret in pod
   ├─ kubelet mounts secret file
   ├─ Pod reads plaintext (in memory)
   └─ Plaintext in pod RAM (can't avoid)

Important: Encryption at-rest protects etcd breaches only
           Once in pod, secret is plaintext in memory
```

### NetworkPolicy (Pod Firewall)

**How it works**:
```
NetworkPolicy object:
  └─ Defines ingress/egress rules
  └─ Stored in etcd

NetworkPolicy Controller:
  └─ Watches NetworkPolicy objects
  └─ Translates to CNI plugin rules

CNI Plugin (Calico/Cilium):
  └─ Receives rules from controller
  └─ Updates kernel netfilter rules (iptables/ipset)
  └─ Enforces at kernel level (high performance)

Packet arrives:
  1. Kernel netfilter intercepts
  2. Checks source pod IP
  3. Matches against NetworkPolicy rules
  4. ACCEPT or DROP
  5. v0ms latency (kernel-level, not userspace)
```

**Example policy evaluation**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
spec:
  podSelector:
    matchLabels:
      app: database  # Applies to pods with this label
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api   # Allow from pods with this label
    - namespaceSelector:
        matchLabels:
          name: secure  # But only in secure namespace
    ports:
    - protocol: TCP
      port: 5432

Evaluation for incoming packet:
  Packet: 10.0.2.5:54321 → database-pod:5432
  
  Check: Database pod exists? ✅
  Check: Database pod has app=database label? ✅
  Check: Source pod has app=api label? ✅
  Check: Source pod in secure namespace? ✅
  Check: Port 5432? ✅
  Result: ALLOW ✅

If source pod in default namespace instead:
  Check: Source pod in secure namespace? ❌
  Result: DROP ❌
```

---

## 3. When to Use vs Not Use

### RBAC Strategy

**Principle of Least Privilege**:
```yaml
# ❌ WRONG: Admin access for everyone
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: all-devs-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: Group
  name: developers  # ALL developers get full access

# Result: One developer can delete production cluster!

# ✅ CORRECT: Fine-grained roles
---
# Role: Developer (limited to namespace)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch", "create", "update"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]  # Can retrieve secrets but not create
---
# Binding: Only for dev namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer
subjects:
- kind: Group
  name: dev-team
---
# Production role: Different permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: operator
  namespace: prod
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]  # Read-only!
# Cannot: create, update, delete
```

### Pod Security Policy vs PSA

**Pod Security Policy (Deprecated in 1.25+)**:
```yaml
# Still available but discouraged
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
  - ALL
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  fsGroup:
    rule: 'RunAsAny'
  readOnlyRootFilesystem: false
```

**Pod Security Admission (Preferred in 1.25+)**:
```yaml
# Namespace label applies PSA standards
apiVersion: v1
kind: Namespace
metadata:
  name: secure
  labels:
    pod-security.kubernetes.io/enforce: restricted   # Block violations
    pod-security.kubernetes.io/audit: restricted    # Log violations
    pod-security.kubernetes.io/warn: restricted      # Warn violations
```

**Difference**: PSP = custom, PSA = built-in standards (baseline, restricted, privileged)

---

## 4. Real-World Production Scenario

### Scenario: Enterprise Multi-Tenant Kubernetes

**Requirements**:
- 15 teams, separate namespaces
- Each team can't access other team's data
- Ops team can view everything (read-only)
- Security team enforces compliance

**Implementation**:

```yaml
# 1. Namespace for each team with isolation
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
  labels:
    owner: team-a
    pod-security.kubernetes.io/enforce: restricted

---
# 2. Service Account for team-a app
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
  namespace: team-a

---
# 3. Pod Security Policy binding (RBAC)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-developer
  namespace: team-a
rules:
# Allow creating pods in their namespace
- apiGroups: [""]
  resources: ["pods", "pods/logs", "pods/exec"]
  verbs: ["get", "list", "watch", "create", "update"]
# Allow deployments
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update"]
# Allow services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch", "create"]
# Allow configmaps in their namespace
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create"]
# DENY: secrets (ops team manages)
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]  # Read-only, can't create

---
# 4. Bind developers to role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-developers
  namespace: team-a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-developer
subjects:
- kind: Group
  name: team-a-developers

---
# 5. Network policy: Isolate team-a
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-intra-team
  namespace: team-a
spec:
  podSelector: {}  # Applies to all pods in team-a
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}  # From team-a pods only
  egress:
  - to:
    - podSelector: {}  # To team-a pods only
  - to:  # DNS to kube-system
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  - to:  # External API
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8  # Block internal ranges except own

---
# 6. Ops team: Read-only access to all namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ops-viewer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "deployments", "statefulsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
# DENY: delete, create, update
# Cannot modify anything

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ops-team-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ops-viewer
subjects:
- kind: Group
  name: ops-team

---
# 7. Security team: Can view events & audit logs
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: security-auditor
rules:
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]
# In real cluster: Also read audit logs from file system
```

**Production Issues**:

```
❌ Issue 1: Team A pods can't reach external API

Network policy too restrictive:
  - All external traffic blocked (ipBlock wildcard issue)
  - Outbound traffic denied
  
Fix: Explicit allow for APIs
  - Design network policy carefully
  - Test before deployment
  - Monitor blocked traffic

❌ Issue 2: Developer accidentally creates privileged pod

Pod created with:
  securityContext:
    privileged: true
    
Pod refused: Pod Security Policy blocks it
Error: "unable to validate against any pod security policy"

Solution: Pod Security Policy working as intended!
  - Prevents security mistakes
  - Error message guides developer
  - Use unprivileged containers

❌ Issue 3: Secret in ConfigMap (security mistake)

Developer puts password in ConfigMap (thinking it's secure):
$ kubectl create configmap secrets --from-literal=password=foo123
$ kubectl get configmap secrets -o yaml
data:
  password: foo123  # Plaintext!

Problem: ConfigMaps NOT encrypted by default
Solution: Use Secrets (still not ideal, but at least encrypted at-rest)
Better: Use external secret management (Vault, AWS Secrets Manager)

❌ Issue 4: Cross-namespace pod can reach database

Network policy in different namespace:
  Team A and Team B in separate namespaces
  Both can reach shared database service
  
Pod from Team B running command:
  curl http://database.team-a.svc.cluster.local:5432
  Success! (shouldn't be allowed)

Root cause: Network policy only in team-a namespace
           Database in team-a namespace but open to all service IPs
           
Fix: Add ingress policy to database deployment
  - Explicitly allow from app pods only
  - Deny from team-b namespace
```

---

## 5. Architecture Diagrams (Text-Based)

### RBAC Authorization Flow

```
User authenticates (certificate/token) → Extracted: username="alice", groups=["dev"]
                                              ↓
                        API Server receives: "get pods in namespace dev"
                                              ↓
                        RBAC Authorizer checks:
        ┌───────────────────────────────────────────────────────┐
        │ 1. Find all RoleBindings + ClusterRoleBindings        │
        │ 2. Match: Is "alice" in subjects?                     │
        │    Check groups: Is dev-team group in subject list?   │
        │    YES → Found RoleBinding "dev-binding"              │
        │         └─ References Role "dev-role"                 │
        │                                                        │
        │ 3. Load Role "dev-role":                              │
        │    rules:                                             │
        │    - apiGroups: [""]                                  │
        │      resources: ["pods"]                              │
        │      verbs: ["get", "list", "watch"]                  │
        │                                                        │
        │ 4. Check: Does rule allow "get" on "pods"?            │
        │    verbs include "get"? YES ✅                         │
        │    resources include "pods"? YES ✅                    │
        │    namespace match? dev-binding is namespace-scoped   │
        │                 namespace="dev" ✅                     │
        │                                                        │
        │ 5. DECISION: ALLOW ✅                                  │
        └───────────────────────────────────────────────────────┘
                                ↓
                    API Server returns pods list to alice

If rule didn't match:
  → Continue checking other RoleBindings
  → If no rule matches → DENY (default deny policy)
```

### Secret Encryption Flow

```
Writing Secret:
  kubectl create secret generic app-secret --from-literal=password=mysecret
       ↓
  Cloud sends to API Server via HTTPS
       ↓
  API Server receives plaintext
       ↓
  Encryption Provider (AES-CBC) encrypts
  plaintext: {"password": "mysecret"}
  encrypted: \x8f\xa3\x2c...  (binary)
       ↓
  Writes encrypted data to etcd
  Key stored in /etc/kubernetes/encryption/encryption-config.yaml (on master node)


Reading Secret:
  kubectl get secret app-secret
       ↓
  API Server reads encrypted data from etcd
       ↓
  Decryption Provider decrypts
  encrypted: \x8f\xa3\x2c...
  plaintext: {"password": "mysecret"}
       ↓
  Returns plaintext to kubectl (over HTTPS TLS)
       ↓
  User sees: password: mysecret


Pod using Secret:
  Volume mount brings secret into pod
       ↓
  kubelet reads encrypted data from etcd
       ↓
  kubelet decrypts (API Server does this, kubelet asks API)
  OR
  kubelet reads from API Server (using token)
       ↓
  Mounts file containing plaintext
  /var/run/secrets/kubernetes.io/serviceaccount/password
       ↓
  Pod can read plaintext

Security levels:
  1. At-Rest (etcd): Encrypted ✅
  2. In-Transit (API → kubelet): TLS ✅
  3. In-Pod (filesystem): Plaintext ✅ (can't avoid, app needs to use password)
```

### NetworkPolicy Enforcement

```
┌─────────────────────────────────────────────────────────┐
│  NetworkPolicy Object (stored in etcd)                 │
│                                                         │
│  Selector: app=database                                │
│  Ingress:                                               │
│    from: app=api (in same namespace)                    │
│    ports: [5432]                                        │
│                                                         │
│  Effect: "Only pods labeled app=api can connect to     │
│           database pods on port 5432"                   │
└─────────────────────────────────────────────────────────┘
            ↓ (NetworkPolicy Controller watches)
┌─────────────────────────────────────────────────────────┐
│  CNI Plugin (Calico/Cilium)                            │
│                                                         │
│  Translates to kernel rules:                           │
│  - Map pod labels to IPs                               │
│  - Create ipset: "database-pods" = [10.0.2.1, ...]    │
│  - Create ipset: "api-pods" = [10.0.2.5, ...]        │
│  - Create iptables rule:                              │
│    target: database-pods                               │
│    match: source ∈ api-pods AND dest port 5432        │
│    action: ACCEPT                                       │
│    else: DROP (implicit default deny)                  │
└─────────────────────────────────────────────────────────┘
            ↓ (Kernel netfilter)
Packet arrives at database pod:
  src: 10.0.2.5 (api pod)
  dst: 10.0.2.1 (database pod)
  port: 5432
  
  Kernel checks ipset:
  - Is src in api-pods? YES ✅
  - Is port 5432? YES ✅
  - Rule matches → ACCEPT ✅

If wrong source:
  src: 10.0.3.5 (batch-job pod)
  dst: 10.0.2.1 (database pod)
  port: 5432
  
  Kernel checks ipset:
  - Is src in api-pods? NO ❌
  - No rule matches → DROP ❌
  - Packet silently dropped
  - Connection timeout from source
```

---

## 6. Common Mistakes

### Mistake 1: Using "system:masters" group for everyone

```yaml
# ❌ CRITICAL MISTAKE
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: all-devs-are-admins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:masters  # HARDCODED ADMIN
subjects:
- kind: Group
  name: developers

# Result: ALL developers = cluster admin = can delete everything

# ✅ CORRECT
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: devs-edit-own-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit  # Built-in role for namespace admin
subjects:
- kind: Group
  name: developers
# This only applies to their own namespace (via namespace scoping)
```

---

### Mistake 2: Storing secrets in ConfigMap

```yaml
# ❌ WRONG - This is just obfuscation, not security
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_PASSWORD: "mysecretpassword"  # Plaintext!
  API_KEY: "sk_live_abcd1234"

$ kubectl get configmap app-config -o yaml
# Easy to read

# ✅ CORRECT - Use Secret (encrypted at-rest)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DB_PASSWORD: bXlzZWNyZXRwYXNzd29yZA==  # Base64 (not encryption!)
  API_KEY: c2tX2xpdmVfYWJjZDEyMzQ=

$ kubectl get secret app-secrets -o yaml
# Still shows base64, but at least encrypted in etcd

# ✅✅ BEST - Use external secret management
# Hashicorp Vault, Azure Key Vault, AWS Secrets Manager
# Rotate secrets automatically
# Audit all access
```

---

### Mistake 3: No Network Policies (Open Cluster)

```yaml
# ❌ DEFAULT BEHAVIOR
# No network policies defined = all traffic allowed
Pod-A talks to Pod-B: ✅ Allowed
Pod-B talks to Database: ✅ Allowed
Pod-B talks to Payment Service: ✅ Allowed (OOPS!)
Attacker Pod talks to Database: ✅ Allowed

# ✅ PRODUCTION STANDARD
# 1. Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: prod
spec:
  podSelector: {}  # All pods
  policyTypes:
  - Ingress
  - Egress
  # No allow rules = nothing allowed

# 2. Allow specific flows
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-customers-to-api
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - port: 8080

# 3. Allow egress to database only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-database
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - port: 5432
```

---

## 7. Debugging Scenarios (VERY IMPORTANT)

### Scenario 1: User gets "permission denied" unexpectedly

```bash
# Error
$ kubectl describe pod my-pod
error: pods "my-pod" is forbidden: User "alice" cannot get resource "pods" in API group "" in the namespace "dev"

# Debug Step 1: Check user's identity
$ kubectl auth whoami
Username: alice
Groups:
- dev-team
- system:authenticated

# Debug Step 2: Check RBAC for this user
$ kubectl auth can-i get pods --as=alice -n dev
no

# "alice" cannot get pods in dev namespace

# Debug Step 3: Check RoleBindings in namespace
$ kubectl get rolebindings -n dev
NAME           ROLE     USERS   GROUPS      AGE
web-binding    web-role web     web-team    1h
api-binding    api-role api     api-team    1h

# alice is in dev-team group, but no binding for dev-team!

# Debug Step 4: Check ClusterRoleBindings
$ kubectl get clusterrolebindings | grep dev-team
# No results

# Root cause: dev-team group not bound to any role!

# Fix: Create RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view  # Can view but not modify
subjects:
- kind: Group
  name: dev-team

# Verify fix works
$ kubectl auth can-i get pods --as=alice -n dev
yes
```

### Scenario 2: Pod can communicate with pod it shouldn't

```bash
# Symptom
Pod-B can connect to Database pod
But Network Policy should block this!

# Debug Step 1: Check all network policies
$ kubectl get networkpolicies -A
NAMESPACE   NAME           POD-SELECTOR
prod        deny-all       {}
prod        allow-api-db   app=api

# Database pod is not protected by any policy
# Pods have no label so don't match any selectors!

# Debug Step 2: Apply labels
$ kubectl label pod database-1 app=database tier=backend -n prod

# Debug Step 3: Create policy for database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-ingress
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - port: 5432

# Debug Step 4: Verify policy
$ kubectl get networkpolicies database-ingress -o yaml

# Debug Step 5: Test connectivity (should fail now)
$ kubectl exec -it pod-b -- curl database:5432
# Should timeout (connection refused)
```

### Scenario 3: Encrypted secret not working

```bash
# Symptom
$ kubectl get secret my-secret -o yaml
# Shows unencrypted data
# Encryption should be enabled!

# Debug Step 1: Check if encryption is enabled
$ ps aux | grep kube-apiserver | grep encryption
/usr/bin/kube-apiserver ...[--encryption-provider-config=/etc/kubernetes/encryption-config.yaml]

# Config file not specified!

# Debug Step 2: Check if file exists
$ ls -la /etc/kubernetes/encryption-config.yaml
-rw-r--r-- 1 root root 500 Jan 1 12:00 /etc/kubernetes/encryption-config.yaml
# File exists

# Debug Step 3: Check file permissions and content
$ cat /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: [base64-key-here]
  - identity: {}  # PROBLEM!
    
# fallback to identity (no encryption)!

# Debug Step 4: Remove identity provider
$ vim /etc/kubernetes/encryption-config.yaml
# Remove "- identity: {}" line

# Debug Step 5: Restart API Server
$ systemctl restart kube-apiserver

# Debug Step 6: Re-create secret (old ones still unencrypted)
$ kubectl delete secret my-secret
$ kubectl create secret generic my-secret --from-literal=password=foo

# Verify encryption
$ etcdctl get /registry/secrets/default/my-secret
<binary garbage>  # Encrypted!
```

### Scenario 4: RBAC policy not being evaluated correctly

```bash
# Symptom
$ kubectl auth can-i delete pods --as=alice
yes

# But alice shouldn't have delete permission!

# Debug Step 1: Check binding details
$ kubectl get rolebindings -A --all-namespaces | grep alice
prod    alice-admin    edit ...

# "alice-admin" binding in prod namespace points to "edit" role

# Debug Step 2: Check the "edit" role
$ kubectl get clusterrole edit -o yaml | head -30
- apiGroups: [""]
  resources: ["configmaps", "endpoints", "persistentvolumeclaims", ...]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]

# Edit role allows DELETE!

# Debug Step 3: Use restricted role
$ kubectl get clusterrole view -o yaml | grep verbs
- ["get", "list", "watch"]  # View only, no delete

# Fix: Change binding to use "view" instead of "edit"
$ kubectl edit rolebinding alice-admin -n prod
# Change roleRef.name from "edit" to "view"

# Verify fix
$ kubectl auth can-i delete pods --as=alice
no
```

---

## 8. Interview Questions (15+ Scenario-Based)

### Level 1: Fundamentals

**Q1**: Explain the difference between authentication and authorization.

**Answer**:
- **Authentication** (AuthN): "WHO are you?"
  - Client presents certificate + private key
  - API Server verifies signature
  - Result: Confirmed identity (alice, bob, service-account-name)

- **Authorization** (AuthZ): "WHAT can you do?"
  - Given identity, check permissions
  - RBAC checks roles and role bindings
  - Result: Allowed or Forbidden

Analogy:
- Authentication = Show ID at airport gate
- Authorization = Check if your ticket is for this flight

---

**Q2**: What happens if etcd gets compromised and someone reads all secrets?

**Answer**:
- If encryption at-rest NOT enabled:
  - All secrets visible in plaintext
  - Passwords, API keys, SSH keys exposed
  - Complete infrastructure compromise

- If encryption enabled:
  - Encrypted secrets unreadable without keys
  - Attacker needs encryption key (stored on master node)
  - Master node also compromised → keys available
  - Still: Better than nothing (protects disk theft, backup theft)

- Defense: Multi-layer
  1. Encrypt secrets at-rest
  2. Protect master nodes (network isolation)
  3. Use external secret management (Vault)
  4. Rotate keys regularly

---

### Level 2: RBAC Deep Dive

**Q3**: You have 100 microservices. Each needs to communicate with database. How do you RBAC?

**Answer**:
- Setup: 100 ServiceAccounts (one per service)
- Problem: Can't write 100 individual Roles

Solution: Use ClusterRole + RoleBinding pattern:
```yaml
# 1. Create ClusterRole for database access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: database-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-credentials"]  # Only this secret
  verbs: ["get"]

# 2. Create RoleBinding for each service
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: service-1-db-access
  namespace: services
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: database-reader
subjects:
- kind: ServiceAccount
  name: service-1
  namespace: services

# OR: Bind all services to same role (if all need same access)
subjects:
- kind: Group
  name: database-readers  # Group all service accounts
```

**Scalability**:
- 1 ClusterRole + 100 RoleBindings = maintainable
- All services get same permissions
- Update once = applies to all

---

**Q4**: A developer's Kubeconfig is exposed. What's compromised?

**Answer**:
Kubeconfig contains:
```yaml
clusters:
- cluster:
    certificate-authority-data: <CA cert>
    server: https://api.example.com
users:
- name: alice
  user:
    client-certificate-data: <alice's client cert>
    client-key-data: <alice's private key>  # ← CRITICAL
```

If private key exposed:
- Attacker can impersonate alice
- Make API calls as alice
- Have all of alice's permissions
- Steal secrets, delete pods, scale deployments

Immediate actions:
1. Revoke user's certificate (can't be done in Kubernetes)
2. Remove alice's RBAC bindings (immediate)
3. Issue new certificate (new kubeconfig)
4. Audit logs for suspicious activity

Prevention:
- Short-lived tokens (OIDC instead of certificates)
- Regular cert rotation
- Webhook authentication (centralized)

---

### Level 3: Network Policy

**Q5**: Write a Network Policy allowing database writes only from "api" tier and reads from "batch" tier.

**Answer**:
```yaml
# Problem: Network policies don't have rdwr distinction
# All connections are TCP connections (can't differentiate read/write)

# Workaround: Use port separation
# - Port 3306 (MySQL): Write port (only API)
# - Port 3307 (MySQL read replica): Read port (only batch)

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-write-api-only
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: database
      role: write
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 3306

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-read-batch-only
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: database-read
      role: read
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: batch
    ports:
    - protocol: TCP
      port: 3307
```

**Note**: Real read/write separation best done at application layer or via service mesh (Istio)

---

### Level 4: Production Security

**Q6**: Your staging cluster is used by 50 developers. How do you prevent accidental deletions?

**Answer**:
Multiple layers:

```yaml
# Layer 1: RBAC - No delete permission
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: staging
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch", "create", "update"]
# DELETE NOT INCLUDED

# Layer 2: Pod Disruption Budget (PDB)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
  namespace: staging
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: app
# Even if someone somehow gets delete permission,
# PDB ensures at least 1 pod always running

# Layer 3: Admission Webhook
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: prevent-delete
webhooks:
- name: prevent-delete.example.com
  admissionReviewVersions: ["v1"]
  clientConfig:
    service:
      name: webhook
      namespace: webhook
      path: "/prevent-delete"
    caBundle: ...
  rules:
  - operations: ["DELETE"]
    apiGroups: [""]
    resources: ["pods"]
    scope: "Namespaced"
  failurePolicy: Fail  # Fail closed (deny if webhook down)
  sideEffects: None

# Webhook logic:
if request.verb == "DELETE" and request.namespace == "staging":
  if request.user NOT in ["platform-team"]:
    return Deny("Deletions disabled in staging")
return Allow()

# Layer 4: Audit Logging
--audit-policy-file=/etc/kubernetes/audit-policy.yaml

# audit-policy.yaml:
- level: RequestResponse
  omitStages:
  - RequestReceived
  resources:
  - group: ""
    resources: ["pods", "services"]
  verbs: ["delete", "deleteCollection"]

# Audit trail logs all delete attempts
```

---

## 9. Advanced Insights (Compliance, Cost, Performance)

### Security Compliance

**Standards that Kubernetes must meet**:
```
CIS Kubernetes Benchmark:
  - 128 security checks
  - Covers: API security, RBAC, network policies, secrets

PCI DSS (Payment Card Industry):
  - Encryption in transit + at-rest
  - Access controls
  - Network segmentation

HIPAA (Healthcare):
  - Encryption of PHI data
  - Audit logging
  - Access control

SOC 2:
  - System and organization controls
  - Change management
  - Access controls

Compliance checklist:
  ✓ RBAC enabled + configured
  ✓ Pod Security Policies enforced
  ✓ Network Policies in place
  ✓ Secrets encrypted at-rest
  ✓ TLS for all communications
  ✓ Audit logging enabled
  ✓ RBAC for compliance team (read-only)
  ✓ Regular penetration testing
```

### Cost of Security

```
Security adds CPU/Memory overhead:
  - NetworkPolicy: 5-10% latency for policy evaluation
  - Secrets encryption: 500-1000ms per secret decrypt
  - RBAC checks: <1ms per API request
  - Audit logging: Disk I/O bottleneck

Optimization:
  - Cache RBAC decisions (10s TTL)
  - Use CNI native network policies (Calico, Cilium)
  - Profile encryption: Use hardware acceleration
  - Compress/ship audit logs (not local disk)
```

### Key Rotation Strategy

```
Encryption key rotation:
  1. Add new key to encryption-config.yaml
  2. Restart API Server (uses both old + new keys)
  3. Mark old key as deprecated
  4. Trigger full re-encryption:
     $ kubectl get secret -A -o json | kubectl apply -f -
  5. Remove old key from config
  6. Restart API Server (only new key)

Timeline:
  T0: Add new key
  T1: Re-encrypt all secrets (could take hours for large clusters)
  T2: Remove old key
  
Total secure period: Usually do quarterly or on demand
```

---

## Conclusion

Kubernetes security is defense-in-depth: **Authentication → Authorization → Admission → Audit**

Master these layers and you can build production-grade secure clusters!

Key principles:
1. Least privilege (RBAC)
2. Network isolation (NetworkPolicy)
3. Secret protection (encryption at-rest)
4. Audit trail (logging)
5. Regular testing (security scans)
