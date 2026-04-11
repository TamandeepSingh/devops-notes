# Helm in Production: Deployment Patterns, CI/CD Integration, and Advanced Operations

## 1. Core Concept

Production Helm deployments require:

1. **Reliability**: Atomic operations, rollback capability, automated testing
2. **Traceability**: Chart versions, release history, audit logs
3. **Safety**: Secrets management, RBAC, policy enforcement
4. **Scale**: GitOps, multi-cluster, canary deployments
5. **Observability**: Release tracking, monitoring, alerts

### Production Architecture

```
Development Pipeline:
├─ Developer writes code + Chart
├─ CI pipeline: lint, test, build
├─ Push to chart repository (tagged version)

Deployment Pipeline:
├─ Monitor repository for new versions
├─ GitOps tool (ArgoCD/Flux) detects changes
├─ Automatic/manual approval
├─ helm upgrade in cluster
├─ Health checks, auto-rollback if unhealthy

Release Management:
├─ Track release history
├─ Rollback capability
├─ Audit trail (who deployed what when)
├─ Alerts on deployment failures

Observability:
├─ Monitor deployed pods
├─ Track resource usage
├─ Alert on anomalies
└─ Correlate with release version
```

---

## 2. Why Production Helm Management is Critical

### Problem: Manual Helm Operations

```bash
# Developer deploys manually
$ helm upgrade myapp ./chart \
  --set image.tag=v2.0.0 \
  -f values-prod.yaml

# Issues:
1. No audit trail (who deployed? when?)
2. Chart version not tracked
3. Manual processes = human error
4. No automated rollback if bad
5. Unclear which version is running
6. Multiple people = inconsistent deploys
7. Hard to reproduce issues
```

### Solution: Production Helm Workflow

```
GitOps-based approach:
├─ Code commits to Git
├─ CI/CD pipeline validates + packages
├─ Chart pushed to registry (version tagged)
├─ Git branch represents "desired state"
├─ ArgoCD watches Git + applies Helm
├─ Deployment automatic or approved
├─ Release history tracked
├─ Rollback = revert Git commit

Advantages:
✅ Audit trail (Git history)
✅ Version controlled (chart + values)
✅ Reproducible (exact same deployment)
✅ Safe rollback (Git revert)
✅ RBAC enforcement (Git-based)
✅ Disaster recovery (Git is source of truth)
```

---

## 3. Internal Working (Release Management, Versioning, Rollback)

### Release History and Metadata Storage

```bash
# When you run:
$ helm install myapp ./chart

# Helm creates:
1. ConfigMap in cluster:
   Name: sh.helm.release.v1.myapp.v1
   Contains: Release metadata, values, templates
   
2. Secret (if needed):
   Name: sh.helm.release.v1.myapp.v1
   Contains: Sensitive values

3. Records in cluster:
   ConfigMap: sh.helm.release.v1.myapp.v1
   (release: v1, revision: 1)
   
   ConfigMap: sh.helm.release.v1.myapp.v2  (after upgrade)
   (release: v2, revision: 2)

# Storage location:
$ kubectl get secret -n <namespace> \
  -l owner=helm,name=myapp

NAME                          TYPE                 DATA
sh.helm.release.v1.myapp.v1   helm.sh/release.v1   1
sh.helm.release.v1.myapp.v2   helm.sh/release.v1   1
sh.helm.release.v1.myapp.v3   helm.sh/release.v1   1
```

### Release Revision and Upgrade Flow

```bash
Step 1: Install release v1 (revision 1)
$ helm install myapp ./chart \
  --set image.tag=v1.0.0

Release: myapp
Revision: 1
Status: deployed
Image: v1.0.0

Step 2: Upgrade to v2 (revision 2)
$ helm upgrade myapp ./chart \
  --set image.tag=v2.0.0

Release: myapp
Revision: 2
Status: deployed (old revision: 1 kept)
Image: v2.0.0

Step 3: Check history
$ helm history myapp

REVISION  UPDATED                   STATUS      CHART        APP VERSION  DESCRIPTION
1         2024-04-10 10:00:00 UTC   SUPERSEDED  my-app-1.0.0  v1.0.0      Install complete
2         2024-04-10 11:00:00 UTC   DEPLOYED    my-app-1.0.0  v2.0.0      Upgrade complete

Step 4: Rollback if needed
$ helm rollback myapp 1

Release: myapp
Revision: 3 (new revision pointing to old version)
Status: deployed
Image: v1.0.0 (rolled back!)

Step 5: Check history again
$ helm history myapp

REVISION  UPDATED                   STATUS      CHART        APP VERSION  DESCRIPTION
1         2024-04-10 10:00:00 UTC   SUPERSEDED  my-app-1.0.0  v1.0.0      Install complete
2         2024-04-10 11:00:00 UTC   SUPERSEDED  my-app-1.0.0  v2.0.0      Upgrade complete
3         2024-04-10 11:05:00 UTC   DEPLOYED    my-app-1.0.0  v1.0.0      Rollback to 1
```

### Chart Version vs App Version vs Release Version

```
Three separate version concepts:

1. CHART VERSION (semantic versioning)
   Chart.yaml: version: 1.2.3
   When chart templates/structure changes
   Example: my-app-1.0.0 → my-app-1.0.1 (bug fix in template)

2. APP VERSION (application version)
   Chart.yaml: appVersion: 2.5.0
   Version of the application packaged
   Example: appVersion: 2.5.0 means app version 2.5.0
   
3. RELEASE REVISION (deployment counter)
   $ helm history myapp shows: REVISION 1, 2, 3, ...
   Sequential number for each release/upgrade
   Separate per release name

Example timeline:
Time  Command                        Chart    App     Release  Revision
T0    helm install app ./chart       1.0.0    2.0.0   deployed  1
T1    helm upgrade app ./chart       1.0.0    2.0.0   deployed  2
      --set image.tag=v2.0.1
T2    helm upgrade app ./chart       1.0.1    2.1.0   deployed  3
      (new chart with new app)
T3    helm rollback app 2            1.0.0    2.0.0   deployed  4
      (back to chart 1.0.0, app 2.0.0)
```

### Helm Secrets Storage

```bash
# Where Helm stores release data:
$ kubectl get secrets -n default -o yaml | grep -A 50 "name: sh.helm"

apiVersion: v1
kind: Secret
metadata:
  name: sh.helm.release.v1.myapp.v1
  namespace: default
type: helm.sh/release.v1
data:
  release: <BASE64-ENCODED-RELEASE-DATA>

# Encoded data contains:
{
  "name": "myapp",
  "namespace": "default",
  "version": 1,
  "released": "2024-04-10T10:00:00Z",
  "config": {
    ... user-provided values ...
  },
  "manifest": "apiVersion: v1\nkind: Deployment\n...",
  "hooks": [],
  "chart": {
    "metadata": {... chart metadata ...}
  },
  "chart_content": "base64 chart tarball"
}

# Implications:
1. Release data stored in cluster (etcd)
2. Secrets readable by anyone with kubectl access
3. Don't store sensitive values in release config
4. Use external secret management for actual secrets
```

---

## 4. Real-World Production Scenarios

### Scenario 1: GitOps Deployment Pipeline with ArgoCD

```
Repository Structure:
├── helm/
│   └── my-app/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│
└── deployments/
    ├── helmfile.yaml
    └── values/
        ├── dev.yaml
        ├── staging.yaml
        └── prod.yaml

CI Pipeline (GitHub Actions):
1. Developer pushes code to feature branch

2. Trigger workflows:
   ├─ helm lint ./helm/my-app
   ├─ helm template myapp ./helm/my-app
   ├─ kubeval rendered manifest
   ├─ conftest validate policies
   └─ Docker build + push image

3. Merge to main branch

4. Trigger release workflow:
   ├─ Bump Chart version (1.0.0 → 1.0.1)
   ├─ helm package ./helm/my-app
   ├─ helm push my-app-1.0.1.tgz oci://registry.example.com
   ├─ Create GitHub release
   └─ Tag: v1.0.1

CD Pipeline (ArgoCD):
1. ArgoCD watches:
   ├─ Helm repository (for chart updates)
   └─ Git repository (for value changes)

2. Detect new chart version (1.0.1)

3. Check deployment strategy:
   - Auto: Immediately upgrade
   - Manual: Wait for approval

4. Execute:
   $ helm upgrade myapp my-app \
     --version 1.0.1 \
     -f deployments/values/prod.yaml

5. Monitor upgrade:
   - Check pod rollout
   - Verify health checks
   - Monitor metrics
   - Auto-rollback if unhealthy

6. Notify team:
   - Deployment success/failure
   - Links to dashboard
   - Release notes
```

### Scenario 2: Canary Deployment (5% → 25% → 100%)

```yaml
# helmfile.yaml (manages multiple releases)
releases:
  - name: myapp-stable
    namespace: prod
    chart: ./helm/my-app
    values:
      - deployments/values/prod.yaml
      - image:
          tag: v1.0.0   # Current stable
      - canary:
          enabled: false
          
  - name: myapp-canary
    namespace: prod
    chart: ./helm/my-app
    values:
      - deployments/values/prod.yaml
      - image:
          tag: v1.1.0   # New version
      - canary:
          weight: 10    # 10% traffic

# Istio VirtualService automatically routes:
# 90% → myapp-stable
# 10% → myapp-canary

Monitoring Phase (10 minutes):
├─ Track canary metrics
├─ Error rate < 1%?
├─ Latency stable?
└─ If good → proceed, else → rollback

Scale Up Phase:
$ helm upgrade myapp ./helm/my-app \
  --set canary.weight=50   # 50% traffic

Verify Phase (10 minutes):
├─ Still healthy?
└─ If good → full cutover, else → rollback

Final Cutover:
$ helm upgrade myapp ./helm/my-app \
  --set image.tag=v1.1.0 \
  --set canary.enabled=false

# All traffic → v1.1.0
# Stable version is now v1.1.0
```

### Scenario 3: Multi-Region Helm Deployment

```yaml
# helmfile.yaml (manages deployments across regions)
releases:
  - name: myapp-us
    namespace: prod
    chart: ./helm/my-app
    values:
      - deployments/values/prod-us.yaml
    kubeContext: prod-us-east
    
  - name: myapp-eu
    namespace: prod
    chart: ./helm/my-app
    values:
      - deployments/values/prod-eu.yaml
    kubeContext: prod-eu-west
    
  - name: myapp-apac
    namespace: prod
    chart: ./helm/my-app
    values:
      - deployments/values/prod-apac.yaml
    kubeContext: prod-apac-sg

# Deployment command:
$ helmfile sync
Syncing: myapp-us (context: prod-us-east)
Syncing: myapp-eu (context: prod-eu-west)
Syncing: myapp-apac (context: prod-apac-sg)
Completed: 3/3 releases deployed

# Advantages:
1. Single command deploys all regions
2. Consistent versioning across regions
3. Easy to detect drift
4. Support for region-specific values
5. Rollback: Single helmfile command
```

### Scenario 4: Blue-Green Deployment

```bash
# Blue: Current production (v1.0.0)
$ helm install myapp-blue ./chart \
  --set image.tag=v1.0.0 \
  --set service.weight=100

Status: Active (receives all traffic)

# Green: New version (v1.1.0) - not receiving traffic yet
$ helm install myapp-green ./chart \
  --set image.tag=v1.1.0 \
  --set service.weight=0

Status: Standby (not receiving traffic)

# Test green (no user traffic)
$ kubectl exec -it myapp-green-xxxx -- /bin/sh
$ curl -s http://localhost:8080/health
{"status": "healthy"}

# All looks good - Switch traffic to green
$ helm upgrade myapp-green \
  --set service.weight=100

# And disable blue:
$ helm upgrade myapp-blue \
  --set service.weight=0

Instant cutover! Blue still running if rollback needed.

# If problems detected:
$ helm upgrade myapp-blue \
  --set service.weight=100

# Traffic back to blue instantly!
# Green issue isolated and can be debugged
```

---

## 5. Common Production Mistakes

### Mistake 1: Not Testing Helm Changes in Dev First

```bash
# ❌ WRONG
$ helm upgrade myapp ./chart \
  --set image.tag=v2.0.0 \
  -f values-prod.yaml
# Applied directly to production!
# If broken, users see error

# ✅ CORRECT
# Test in dev first:
$ helm upgrade myapp-dev ./chart \
  --set image.tag=v2.0.0 \
  -f values-dev.yaml

# Monitor dev for 1 hour
$ kubectl logs -f -l app=myapp -n dev
# No errors? Good!

# Test in staging:
$ helm upgrade myapp-staging ./chart \
  --set image.tag=v2.0.0 \
  -f values-staging.yaml

# Monitor staging for 1 hour
$ kubectl logs -f -l app=myapp -n staging

# Only after staging success:
$ helm upgrade myapp ./chart \
  --set image.tag=v2.0.0 \
  -f values-prod.yaml
```

### Mistake 2: Not Using --wait and Health Checks

```bash
# ❌ WRONG
$ helm upgrade myapp ./chart
# Command returns immediately
# But pods still rolling out!
# Broken image? You won't know until later

# ✅ CORRECT
$ helm upgrade myapp ./chart \
  --wait \
  --timeout 5m
  
# Helm waits for pods to be ready
# Returns success only when healthy
# Returns error if timeout/unhealthy

# Even better: Add health checks to pods
# templates/deployment.yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5

# Now Helm detects unhealthy pods
$ helm upgrade myapp ./chart --wait
# Fails if pods not healthy!
```

### Mistake 3: Not Locking Chart Versions

```bash
# ❌ WRONG
helmfile.yaml:
releases:
  - name: myapp
    chart: mycompany/my-app
    # No version specified!
    # Uses 'latest' - unpredictable!

# Result:
# Today: Deploy with my-app-1.0.0
# Tomorrow: Same helmfile, but my-app-1.0.1 deployed!
# Nobody knows what version is running!

# ✅ CORRECT
helmfile.yaml:
releases:
  - name: myapp
    chart: mycompany/my-app
    version: 1.0.0          # LOCKED!

# Also use Chart.lock for dependencies:
Chart.yaml:
dependencies:
- name: mysql
  version: "8.0.0"
  repository: "..."

# $ helm dependency update
# Creates Chart.lock with exact versions
# Ensures reproducible deployments
```

### Mistake 4: Storing Secrets in Helm Release

```bash
# ❌ WRONG - values.yaml
database:
  password: super-secret-123

$ helm install myapp ./chart
$ helm get values myapp
database:
  password: super-secret-123  # Visible!
  
$ kubectl get secret \
  sh.helm.release.v1.myapp.v1 \
  -o yaml | grep password
# Base64 encoded (easily decoded!)

# ✅ CORRECT - Use external secret management
# values.yaml
database:
  existingSecret: db-credentials
  existingSecretPasswordKey: password

# Create secret separately:
$ kubectl create secret generic db-credentials \
  --from-literal=password=super-secret-123

# Or use ExternalSecrets operator:
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  secretStoreRef:
    name: vault
  target:
    name: db-credentials
  data:
  - secretKey: password
    remoteRef:
      key: prod/database/password

# Now secret not in Helm release!
```

### Mistake 5: Not Monitoring Release Health

```bash
# ❌ WRONG
$ helm upgrade myapp ./chart
$ # Trust it worked

# ✅ CORRECT - Comprehensive monitoring
$ helm upgrade myapp ./chart --wait

# Monitor rollout
$ kubectl rollout status deployment/myapp -w

# Check pod status
$ kubectl get pods -l app=myapp
NAME          READY   STATUS    RESTARTS
myapp-xxxxx   1/1     Running   0

# Tail logs
$ kubectl logs -f -l app=myapp

# Check events
$ kubectl describe deployment myapp

# Verify endpoints updated
$ kubectl get endpoints myapp

# Automated monitoring:
- Prometheus alerts on pod restart rate
- Alert on deploy failures
- Auto-rollback if metrics degrade
```

---

## 6. Debugging Production Helm Issues

### Scenario 1: Helm Upgrade Failed, Need Rollback

```bash
# Situation: $ helm upgrade myapp ./chart
# Error: ImagePullBackOff

# Step 1: Check release status
$ helm status myapp
Status: FAILED

# Step 2: Check pod status
$ kubectl get pods -l app=myapp
NAME                READY   STATUS              RESTARTS   AGE
myapp-new-xxxxx     0/1     ImagePullBackOff    0           2m
myapp-old-xxxxx     1/1     Running             0           1h

# Step 3: Check events
$ kubectl describe pod myapp-new-xxxxx
Error: ImagePullBackOff: image not found

# Step 4: Check image tag
$ helm values myapp | grep tag
tag: v2.0.0-typo

# Step 5: Rollback immediately
$ helm rollback myapp
Release "myapp" has been rolled back to previous release

# Step 6: Verify old version running
$ kubectl get pods
myapp-old-xxxxx   1/1   Running     0

# Step 7: Check what went wrong
$ helm history myapp
REVISION  UPDATED          STATUS      DESCRIPTION
1         2024-04-10 10:00  SUPERSEDED  Install complete
2         2024-04-10 11:00  SUPERSEDED  Upgrade complete
3         2024-04-10 11:05  DEPLOYED    Rollback to 1

# Step 8: Fix and retry
# Fix: Correct image tag in values
$ helm upgrade myapp ./chart \
  --set image.tag=v2.0.0 \
  --wait
```

### Scenario 2: Release Works in Dev, Fails in Prod

```bash
# Issue: $ helm upgrade myapp ./chart -f values-prod.yaml
# Pods crashing with no obvious error

# Step 1: Compare dev vs prod releases
$ helm get values myapp-dev > /tmp/dev-values.yaml
$ helm get values myapp-prod > /tmp/prod-values.yaml
$ diff /tmp/dev-values.yaml /tmp/prod-values.yaml

# Found difference:
# Prod has: database.host: prod-db.internal

# Step 2: Check if prod database is accessible
$ kubectl get pods myapp-prod-xxxxx -o yaml | \
  grep DATABASE_HOST

DATABASE_HOST: prod-db.internal

# Step 3: Test connectivity from pod
$ kubectl exec -it myapp-prod-xxxxx -- nc -zv prod-db.internal 5432
# Connection timeout!

# Step 4: Check DNS
$ kubectl exec -it myapp-prod-xxxxx -- nslookup prod-db.internal
# prod-db.internal not found (wrong namespace?)

# Step 5: Check actual database service
$ kubectl get services -A | grep db
prod  db.prod.svc.cluster.local

# Step 6: Fix values
values-prod.yaml:
database:
  host: db.prod.svc.cluster.local  # Corrected

# Step 7: Re-deploy with fix
$ helm upgrade myapp ./chart \
  -f values-prod.yaml \
  --wait
```

### Scenario 3: Release Revision Mismatch (Stale Cache)

```bash
# Issue: $ helm status myapp
# Shows old status even after upgrade

# Step 1: Force refresh from cluster
$ helm get values myapp
# Still shows old values?

# Step 2: Check actual release in cluster
$ kubectl get secret sh.helm.release.v1.myapp.v* -o name
sh.helm.release.v1.myapp.v1
sh.helm.release.v1.myapp.v2  # <- Latest is v2!

# Step 3: Force Helm to refresh
$ helm repo update
$ helm pull myapp --version 2.0.0

# Step 4: Verify fresh status
$ helm status myapp
Status: DEPLOYED
Revision: 2

# Step 5: If still wrong, check local Helm cache
$ rm -r ~/.cache/helm
$ helm status myapp
# Should show correct now
```

---

## 7. Interview Questions (15+ Scenario-Based)

### Level 1: Production Basics

**Q1**: What information does Helm store in the cluster after deploying a release?

**Answer**:
Helm stores release metadata in Kubernetes secrets:

```bash
Stored in: Secrets in release namespace
Name pattern: sh.helm.release.v1.<release-name>.v<revision>

Contains:
1. Release name, namespace, version
2. User-provided values (all configuration)
3. Rendered manifest (exact YAML deployed)
4. Chart metadata
5. Release hooks (if defined)
6. Timestamp, status

Example:
$ kubectl get secret sh.helm.release.v1.myapp.v1 -o yaml
data:
  release: <base64-encoded-json>

Implications:
- Release history fully recoverable
- Values must not contain secrets (use external secret mgmt)
- Only readable by users with secret access
- Stored in etcd (secure Kubernetes access)
```

---

**Q2**: Explain the difference between a failed and succeeded Helm upgrade.

**Answer**:
```bash
# Failed upgrade
$ helm upgrade myapp ./chart --set image.tag=v2.0.0
Error: Service account default cannot create resources

$ helm status myapp
Status: FAILED

# Results:
- Pods may be 0/1 or mix of old+new
- Release updated to new revision but pods not ready
- Can rollback or retry

# kubectl get pods:
old-pod-xxxxx    1/1    Running    0
new-pod-yyyyy    0/1    Pending    0  (waiting for resources)

---

# Succeeded upgrade
$ helm upgrade myapp ./chart --set image.tag=v2.0.0
Succeeded

$ helm status myapp
Status: DEPLOYED

# Results:
- All pods running
- All replicas at desired count
- Release at new revision

# kubectl get pods:
new-pod-yyyyy    1/1    Running    0
new-pod-zzzz     1/1    Running    0

Difference: Status shows cluster actual state, not just Helm operation success
```

---

### Level 2: Advanced Production

**Q3**: Design a Helm upgrade strategy for a stateful application (database).

**Answer**:
```yaml
# helm/postgresql/values.yaml
# Stateful deployment requires careful upgrade

image:
  tag: v14.5         # Current version

persistence:
  enabled: true
  storageClassName: fast-ssd
  size: 100Gi

# For major version upgrades:
upgrade:
  strategy: pg_upgrade

# Deployment strategy: StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
spec:
  serviceName: postgresql
  replicas: 1
  updateStrategy:
    type: OnDelete  # Update only when pod deleted
  
# Upgrade process:
# 1. Create backup
$ kubectl exec -it postgresql-0 -- pg_dump > backup.sql

# 2. Scale down to ensure data consistency
$ helm upgrade postgresql ./chart \
  --set replicas=0 \
  --wait

# 3. Add migration annotation
$ helm upgrade postgresql ./chart \
  --set image.tag=v15.0 \
  --set upgrade.strategy=backup_and_migrate \
  --wait

# 4. StatefulSet re-creates pod with new version
# 5. Helm includes migration init container:
  initContainers:
  - name: pg-upgrade
    image: postgres:15
    command:
    - /bin/bash
    - -c
    - pg_upgrade --old-bindir=/usr/lib/postgresql/14/bin --new-bindir=/usr/lib/postgresql/15/bin

# 6. If upgrade fails, data unchanged on disk
$ helm rollback postgresql
# Old version pod reads same data

# 7. Verify
$ kubectl logs postgresql-0
# Check migration logs
```

---

**Q4**: Design a rollback strategy for a production incident.

**Answer**:
```bash
# Monitoring detects issue (5 minutes after deploy)
Alert: Error rate 5% (threshold: 1%)

# Step 1: Immediate assessment (2 minutes)
$ helm status myapp
Status: DEPLOYED
Revision: 15

$ helm history myapp | head -5
REVISION  UPDATED              STATUS        CHART          DESCRIPTION
15        2024-04-10 11:30:00  DEPLOYED      my-app-1.2.3   Upgrade complete
14        2024-04-10 11:00:00  SUPERSEDED    my-app-1.2.2   Upgrade complete
13        2024-04-10 10:30:00  SUPERSEDED    my-app-1.2.2   Rollback to 14

# Step 2: Immediate mitigation (2 minutes)
# Rollback without investigation
$ helm rollback myapp 14
Release "myapp" has been rolled back to previous release

# Step 3: Verify rollback
$ kubectl rollout status deployment/myapp -w
# Wait for pods to stabilize

# Step 4: Verify metrics
$ prometheus query: rate(errors[5m])
# Should show error rate dropping

# Step 5: Notify team
"Rolled back to v1.2.2 due to error spike. RCA in progress."

# Step 6: RCA (root cause analysis) in dev
$ helm get values myapp-v15-debug -f values-staging.yaml
# Reproduce the issue

# Step 7: Fix and re-deploy
$ helm upgrade myapp ./chart \
  --set image.tag=v1.2.3-fix \
  --set debug.enabled=false \
  --wait

# Step 8: Monitor
$ kubectl logs -f -l app=myapp | grep -i error
# Should show no errors

# Step 9: Post-incident
- Document the issue
- Add automated test to prevent
- Review change process
```

---

## 8. Advanced Insights (Helm Ecosystem & Patterns)

### Pattern 1: Multi-Stage Helm Promotions

```
Development:
├─ Branch: feature/xyz
├─ Values: values-dev.yaml
├─ Registry: my-company-dev/my-app
├─ Frequency: Push on every commit

Staging:
├─ Branch: release/
├─ Values: values-staging.yaml
├─ Registry: my-company-staging/my-app
├─ Frequency: Nightly builds
├─ Requirement: 24-hour stability test

Production:
├─ Branch: main (only from release/)
├─ Values: values-prod.yaml
├─ Registry: my-company-prod/my-app (immutable)
├─ Frequency: Manual approval
├─ Requirement: Staging success + manual approval

Promotion pipeline:
feature → develop → staging → production

Git Flow Integration:
$ git checkout -b feature/new-api
# Dev auto-deploys

$ git commit && git push
# Dev: auto-redeploy on each push

# Testing complete
$ git checkout release/
$ git merge feature/new-api
# Staging: auto-deploy for 24h testing

# Staging stable
$ git checkout main
$ git merge release/
# Production: Manual approval required
$ helm upgrade myapp ./chart \
  --version 1.2.3 \
  -f values-prod.yaml

# Result: Tested, approved, production-ready
```

### Pattern 2: Secrets Rotation with Helm

```yaml
# Problem: Secret stored in cluster, need rotation every 90 days

# Solution: ExternalSecrets + Helm
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h          # Refresh hourly
  secretStoreRef:
    name: vault
    kind: SecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: prod/db/username
  - secretKey: password
    remoteRef:
      key: prod/db/password

# Helm template mounts secret:
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password

# Rotation process:
1. Vault admin rotates secret in Vault
2. ExternalSecret controller detects change
3. Kubernetes Secret automatically updated
4. App pods notice env change
5. App reconnects with new credentials

# Zero downtime, automatic rotation!
# No Helm re-deploy needed!
```

### Pattern 3: Helm + Service Mesh Integration

```yaml
# Helm chart automatically integrates with Istio/Linkerd

# Chart values for mesh support
serviceMesh:
  enabled: true
  type: istio  # or: linkerd

# Chart templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  template:
    metadata:
      {{- if eq .Values.serviceMesh.type "istio" }}
      annotations:
        sidecar.istio.io/inject: "true"
      {{- else if eq .Values.serviceMesh.type "linkerd" }}
      annotations:
        linkerd.io/inject: enabled
      {{- end }}
    spec:
      containers:
      - name: app
        ports:
        - name: http
          containerPort: 8080

# Helm-managed VirtualService (Istio traffic management)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: {{ .Release.Name }}
spec:
  hosts:
  - {{ .Release.Name }}
  http:
  - route:
    - destination:
        host: {{ .Release.Name }}
      weight: {{ sub 100 .Values.canary.weight }}
    - destination:
        host: {{ .Release.Name }}-canary
      weight: {{ .Values.canary.weight }}

# Helm-managed DestinationRule (retry policy)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: {{ .Release.Name }}
spec:
  host: {{ .Release.Name }}
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: {{ .Values.maxConnections }}
      http:
        http1MaxPendingRequests: {{ .Values.maxPending }}
        maxRequestsPerConnection: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s

# Features enabled:
- Service mesh injection (automatic sidecar)
- Traffic management (canary, A/B testing)
- Resilience (retry, timeout, circuit breaker)
- Observability (tracing, metrics)
- Security (mTLS)
```

### Pattern 4: Helm Values in Policy Enforcement

```yaml
# Kyverno policy validates Helm values before deploy
# Ensures security, compliance, cost requirements

apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: helm-values-validation
spec:
  validationFailureAction: audit
  rules:
  - name: check-resource-limits
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "CPU limit required"
      pattern:
        spec:
          containers:
          - resources:
              limits:
                memory: "?*"
                cpu: "?*"

  - name: check-image-registry
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Only approved registries allowed"
      pattern:
        spec:
          containers:
          - image: "registry.example.com/*"

  - name: check-pdb-for-replicas
    match:
      resources:
        kinds:
        - Deployment
    validate:
      message: "replicas > 1 requires PodDisruptionBudget"
      preconditions:
        - key: "{{request.object.spec.replicas}}"
          operator: greaterThan
          value: 1
      # Must have matching PDB

# When user applies Helm release with non-compliant values:
$ helm upgrade myapp ./chart \
  --set image.registry=docker.io
Error: policy violation: Only approved registries allowed

# Forces Helm users to comply with org policies
# Example policies:
- Resource limits required (cost control)
- Images from approved registries (security)
- Replicas >= 2 (reliability)
- Non-root containers (security)
```

---

## Conclusion

Production Helm requires:

1. **Robust Versioning**: Chart version + app version + release revision tracking
2. **Safe Deployments**: Testing, dry-runs, automated rollback
3. **Release History**: Full audit trail via Git + Helm
4. **Secret Management**: External secret storage, never in values
5. **Observability**: Monitor releases, health checks, alerts
6. **GitOps**: Git as source of truth, automated deployments
7. **Policy Enforcement**: Kyverno/OPA ensures compliance
8. **Ecosystem Integration**: Service mesh, secret rotation, multi-region

Master these patterns and you can:
- Deploy applications reliably at scale
- Recover quickly from incidents
- Maintain audit trails for compliance
- Integrate with modern cloud-native tooling
- Manage multi-region, multi-environment production systems

**Key principle**: Helm is powerful, but production requires automation, policy, and careful operational practices!
