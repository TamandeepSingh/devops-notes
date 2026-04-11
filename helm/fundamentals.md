# Helm Fundamentals: Deep Dive

## 1. Core Concept

Helm is the **package manager for Kubernetes** - it simplifies application deployment by:

1. **Packaging**: Bundle YAML manifests + configuration into reusable charts
2. **Templating**: Dynamic YAML generation from values files
3. **Versioning**: Track application versions like Docker image tags
4. **Release Management**: Install, upgrade, rollback in coordinated fashion

### Three Core Concepts

**Chart**: 
- Directory structure containing templates + values + metadata
- Similar to: Package in npm, rpm, Maven
- Location: Chart repositories (like DockerHub for images)

```
my-app/
├── Chart.yaml              # Metadata (name, version, etc)
├── values.yaml             # Default values
├── templates/              # Manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl        # Template helpers
└── charts/                 # Dependency charts
```

**Values**:
- Configuration data (environment-specific)
- Can be in files or passed via CLI
- Example: `replicas: 3`, `image.tag: v1.2.3`

**Release**:
- Running instance of a chart with specific values
- Like: "mysql-production" (release name) = mysql chart + prod values
- Each release tracked independently (versions, history, rollback)

### Helm Workflow

```
1. Create Chart (package structure)
   └─ Write templates, default values, Chart.yaml

2. User provides values
   └─ Via values.yaml file or --set flags
   └─ Merges with chart defaults

3. Helm renders templates
   └─ Processes Jinja2-like templating language
   └─ Substitutes values into YAML
   └─ Produces: deployment.yaml, service.yaml, etc

4. Submit to Kubernetes
   └─ Helm sends rendered YAML to API Server
   └─ kubectl apply -f rendered-manifests.yaml

5. Track release
   └─ Helm stores release metadata in Kubernetes ConfigMap/Secret
   └─ Enables: rollback, history, upgrades

Example:
  Chart: mysql-5.7.0
  Values: replication.enabled=true, replication.replicas=3
  Release: "prod-mysql" deployed with 3 replicas
  
  Later: Replication issues, value changed to replicas=2
  $ helm upgrade prod-mysql mysql --set replication.replicas=2
  
  Still broken, need to rollback
  $ helm rollback prod-mysql 1  (back to first release)
  Release reverts to replicas=3
```

---

## 2. Why Helm is Used

### Problem Without Helm

**Scenario**: Deploy multi-tier application (Web + API + Database + Cache)

```yaml
# deployment-web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-web
  labels:
    app: app
spec:
  replicas: 3  # Hardcoded
  
# deployment-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-api
  labels:
    app: app
spec:
  replicas: 5  # Hardcoded
  
# service-web.yaml
apiVersion: v1
kind: Service
metadata:
  name: app-web
spec:
  ports:
  - port: 80

# service-api.yaml
...
# (Many more YAML files)
```

**Issues**:
- Deploying to dev vs staging vs prod = copy-paste entire files
- Different image tags per environment = manual editing
- Replicas different per environment = maintain N copies
- Secrets handling = unsafe
- Version tracking = who has which version?
- Rollback = manual kubectl operations
- Dependencies between resources = implicit

### Solution: Helm

```
helm install app ./chart \
  --namespace prod \
  -f values-prod.yaml

Helm handles:
1. ✅ Templating (values → YAML)
2. ✅ Environment separation (values-prod.yaml vs values-staging.yaml)
3. ✅ Version tracking (every install/upgrade tracked)
4. ✅ Atomic operations (install all or nothing)
5. ✅ Rollback capability (rollback to previous version instantly)
6. ✅ Dependency ordering (install in correct order)
```

### When Helm Makes Sense

**✅ Use Helm When:**
- Multiple apps to deploy (reduces YAML duplication)
- Environment differences (dev, staging, prod settings)
- Frequent deployments/rollbacks needed
- Team collaboration (chart as source of truth)
- Package reuse (share charts across teams/orgs)

**❌ Avoid Helm When:**
- Single simple deployment (kustomize or raw kubectl better)
- Complex business logic (Helm not a config tool)
- Real-time configuration changes (Helm static at deploy time)

### Helm vs Alternatives

| Tool | Strength | Weakness |
|------|----------|----------|
| **Helm** | Packaging, versioning, repos | Steeper learning curve |
| **Kustomize** | Simple overlays, no templating | Limited features |
| **kubectl + envsubst** | Full control, simple | Manual management, error-prone |
| **GitOps (ArgoCD)** | Declarative, continuous sync | Extra layer of complexity |
| **Terraform** | IaC, cloud resources | Not Kubernetes-specific |

---

## 3. Internal Working (Charts, Templates, Rendering)

### Chart Structure Deep Dive

```
my-app/
├── Chart.yaml
│   └─ apiVersion: v2  # Helm 3 format
│      name: my-app
│      version: 1.2.3  # Chart version (not app version)
│      appVersion: 2.5.0  # Application version
│      description: "My awesome app"
│      maintainers:
│      - name: jane
│        email: jane@example.com
│
├── values.yaml
│   └─ # Default values (can be overridden)
│      # Every value here can be: {{ .Values.key }}
│      replicaCount: 3
│      image:
│        repository: nginx
│        tag: "latest"
│      service:
│        type: ClusterIP
│        port: 80
│
├── templates/
│   ├── deployment.yaml
│   │  └─ Processing:
│   │     Input (template):  replicas: {{ .Values.replicaCount }}
│   │     After rendering:   replicas: 3
│   │
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml  # If sensitive data
│   └── _helpers.tpl  # Reusable template functions
│
├── charts/
│   └─ # Dependency charts (managed by Chart.lock)
│      mysql-8.0.3/
│      redis-6.5.0/
│
├── .helmignore  # Files to exclude (like .gitignore)
│
└── Chart.lock  # Lock file (pinned dependency versions)
   └─ Ensures reproducible builds
      (similar to package-lock.json)
```

### Rendering Process (Detailed)

**Input**: Template file + Values object

```yaml
# templates/deployment.yaml (template)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app      # Helm variable
spec:
  replicas: {{ .Values.replicaCount }}  # From values.yaml
  template:
    spec:
      containers:
      - name: app
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        ports:
        - containerPort: {{ .Values.service.port }}
```

**Values object (what Helm builds internally)**:

```yaml
.Release:
  Name: "my-app"           # From helm install my-app ...
  Namespace: "default"
  IsUpgrade: false
  IsInstall: true
  Revision: 1

.Values:
  replicaCount: 3
  image:
    repository: "nginx"
    tag: "latest"
  service:
    port: 80

.Chart:
  Name: "my-app"
  Version: "1.2.3"

.Files:               # Access to other files
  # Config files, scripts, etc

.Capabilities:        # Cluster capabilities
  KubeVersion: "1.28.0"
  APIVersions: [...]
```

**Rendering engine** (Sprig functions):

```yaml
# Template functions available:
{{ upper .Release.Name }}           # Output: MY-APP
{{ default "3" .Values.replicas }}  # Use default if empty
{{ range .Values.items }}           # Loop through items
  - name: {{ . }}
{{ end }}
{{ if .Values.enabled }}            # Conditional
  enabled: true
{{ end }}
```

**Output**: Pure YAML (no template syntax left)

```yaml
# After rendering
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: nginx:latest
        ports:
        - containerPort: 80
```

This YAML is then applied to cluster.

### Dependency Resolution

```yaml
# Chart.yaml
dependencies:
- name: mysql
  version: "8.0.3"
  repository: "https://charts.bitnami.com/bitnami"
  condition: mysql.enabled  # Only install if true
  alias: database           # Rename to "database" in values

- name: redis
  version: "6.5.0"
  repository: "https://charts.bitnami.com/bitnami"
  tags: ["cache"]           # Install if tag matches

# values.yaml
mysql:
  enabled: true
  auth:
    rootPassword: "secret"
  
redis:
  enabled: true
  replica:
    replicaCount: 2

# Build process:
$ helm dependency update
  ├─ Fetches mysql-8.0.3 from repo
  ├─ Fetches redis-6.5.0 from repo
  ├─ Creates Chart.lock (locks versions)
  ├─ Extracts to charts/
  └─ Updates dependencies

When helm install runs:
  1. Render parent chart templates
  2. Render mysql chart (with database values)
  3. Render redis chart (with redis values)
  4. Merge all YAML
  5. Install all resources
  
Rollback:
  $ helm rollback my-app 1
  └─ Rolls back parent + all dependencies to previous versions
```

---

## 4. Real-World Usage

### Scenario 1: Multi-Environment Deployment

**Setup**:
```
charts/
├── my-app/              # Main chart
│   ├── Chart.yaml
│   ├── values.yaml      # Default values
│   └── templates/
│
├── values-dev.yaml      # Dev overrides
├── values-staging.yaml  # Staging overrides
└── values-prod.yaml     # Prod overrides

# values.yaml (defaults)
replicaCount: 1
image:
  repository: myapp
  tag: latest
database:
  host: localhost
  port: 5432
resources:
  requests:
    cpu: 100m
    memory: 128Mi

# values-prod.yaml
replicaCount: 5         # More replicas in prod
image:
  tag: v1.2.3          # Specific version (not latest!)
database:
  host: prod-db.internal.service
  port: 5432
  sslMode: require      # Enforce SSL
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 1000m
    memory: 2Gi
```

**Deployment commands**:
```bash
# Dev
helm install myapp ./charts/my-app \
  -n dev --create-namespace \
  -f values-dev.yaml

# Staging
helm install myapp ./charts/my-app \
  -n staging --create-namespace \
  -f values-staging.yaml

# Prod
helm install myapp ./charts/my-app \
  -n prod --create-namespace \
  -f values-prod.yaml

# Each environment gets different settings
# Same chart, different values = DRY principle
```

### Scenario 2: Application Upgrade with Rollback

```bash
# Release v1 deployed
$ helm install myapp ./charts/my-app \
  --set image.tag=v1.0.0
Release: myapp
Namespace: default
Revision: 1

# Users: Everything working

# New version released
$ helm upgrade myapp ./charts/my-app \
  --set image.tag=v1.1.0
Release: myapp
Namespace: default
Revision: 2

# New pods rolling out (old → new)
# Users: Still being served by old pods during transition

# Check deployment
$ helm status myapp
Release: myapp
Status: deployed
Revision: 2

# Monitor new pods
$ kubectl get pods -w
# All pods become ready with v1.1.0

# BUT: Bug discovered in v1.1.0!
# Error logs showing issues

# ROLLBACK instantly
$ helm rollback myapp 1
Release: myapp
Namespace: default
Revision: 3 (new revision, but pointing to v1 image)

# Pods revert to v1.0.0
# Users: Service restored, zero manual work!

# Check history
$ helm history myapp
REVISION  UPDATED              STATUS      CHART   APP VERSION   DESCRIPTION
1         2024-04-10 10:00:00  SUPERSEDED  my-app-1.0.0  v1.0.0  Install complete
2         2024-04-10 11:00:00  SUPERSEDED  my-app-1.0.0  v1.1.0  Upgrade complete
3         2024-04-10 11:05:00  DEPLOYED    my-app-1.0.0  v1.0.0  Rollback to 1

# Compare what changed
$ helm diff revision myapp 1 2
# Shows differences between revisions
```

### Scenario 3: Helm + GitOps Pipeline

```
Repository:
├── helm/
│   └── my-app/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│
└── values/
    ├── values-dev.yaml
    ├── values-staging.yaml
    └── values-prod.yaml

CI Pipeline (GitHub Actions):
  1. Developer pushes PR with Chart changes
  
  2. Validate syntax
     $ helm lint ./helm/my-app
     
  3. Dry-run install (preview)
     $ helm install myapp ./helm/my-app \
       --dry-run --debug \
       -f values/values-dev.yaml
     
  4. Package chart
     $ helm package ./helm/my-app
     └─ Creates: my-app-1.0.0.tgz
     
  5. Push to chart registry (like DockerHub for charts)
     $ helm push my-app-1.0.0.tgz oci://registry.example.com
     
  6. Merge to main branch
  
CD Pipeline (ArgoCD or Flux):
  1. Watch Git repository
  
  2. Detect new chart version in values file
  
  3. Automatically upgrade
     $ helm upgrade myapp my-app \
       --version 1.0.0 \
       -f values/values-prod.yaml
  
  4. If upgrade fails
     └─ ArgoCD detects unhealthy pods
     └─ Automatically rollback
     └─ Notifies team
        
  5. If successful
     └─ ArgoCD confirms healthy
     └─ Notifies team
     └─ Git becomes source of truth
```

---

## 5. Common Mistakes

### Mistake 1: Using image.tag: "latest" in Production

```yaml
# ❌ WRONG
image:
  repository: myapp
  tag: latest     # Unpredictable!

Problems:
1. Docker builder always pulls new image
2. Helm cache might use old "latest"
3. Rollback doesn't work (rollback to "latest" = different image!)
4. Hard to debug ("which version is running?")

# ✅ CORRECT
image:
  repository: myapp
  tag: v1.2.3     # Specific version

Chart versioning:
- App version: 1.2.3 (Docker image tag)
- Chart version: 1.0.0 (Helm chart version)
- Each combination tracked separately
```

### Mistake 2: Storing Secrets in values.yaml

```yaml
# ❌ WRONG
values.yaml
---
database:
  password: "super-secret-password-123"
  apiKey: "sk_live_abcd1234"

Problems:
1. Checked in to Git (plaintext visible!)
2. Helm values merged into ConfigMap (readable in cluster)
3. Anyone with chart access has secrets
4. Audit trail shows secrets in helm history

# ✅ CORRECT
values.yaml
---
database:
  passwordSecret:
    name: db-credentials
    key: password

# Store actual secret separately
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
data:
  password: c3VwZXItc2VjcmV0LXBhc3N3b3JkLTEyMw== # Base64

# Or: Use external secret management
# Helm template references: {{ .Values.externalSecret.name }}
# ExternalSecret controller fetches from Vault/AWS Secrets Manager
```

### Mistake 3: Over-templating (Too Many {{.Values.}})

```yaml
# ❌ WRONG (Over-templated)
templates/deployment.yaml
---
spec:
  replicas: {{ .Values.deployment.replicas }}
  selector:
    matchLabels:
      app: {{ .Values.deployment.selector.matchLabels.app }}
      version: {{ .Values.deployment.selector.matchLabels.version }}
  template:
    metadata:
      labels:
        app: {{ .Values.deployment.selector.matchLabels.app }}
        version: {{ .Values.deployment.selector.matchLabels.version }}

values.yaml has 50+ values!
Too complex to understand what changes what

# ✅ CORRECT (Right-sized templating)
templates/deployment.yaml
---
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
      version: {{ .Chart.Version }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
        version: {{ .Chart.Version }}

values.yaml:
---
replicaCount: 3

Rule of thumb:
- Template what changes per environment
- Hardcode what's always the same
- Use helpers for complex logic
```

### Mistake 4: Not Testing Before Upgrade

```yaml
# ❌ WRONG
$ helm upgrade myapp ./chart \
  --set image.tag=v2.0.0
# Applied directly to prod!

# Immediate outage

# ✅ CORRECT
$ helm upgrade myapp ./chart \
  --set image.tag=v2.0.0 \
  --dry-run --debug
# Preview what will change

$ helm diff upgrade myapp ./chart \
  --values values-prod.yaml
# Shows exact changes before applying

# Test in dev first
$ helm upgrade myapp-dev ./chart \
  --values values-dev.yaml
  -f values/v2.0.0-test.yaml

# Verify in dev
$ kubectl get pods -n dev
$ kubectl logs -n dev -l app=myapp

# Only then upgrade prod
$ helm upgrade myapp ./chart \
  --values values-prod.yaml
```

---

## 6. Debugging Helm Issues

### Scenario 1: Helm Install Fails with "Error: template: deployment.yaml"

```bash
# Error message
Error: template: deployment.yaml:45:2: executing "deployment.yaml" as template:
  error calling dict: unsupported type

# Debug Step 1: Check template syntax
$ helm lint ./charts/my-app
==> Linting charts/my-app

1. Template syntax error

# Debug Step 2: Render template locally
$ helm template myapp ./charts/my-app \
  -f values.yaml
# Shows exactly what template produces
# Look for: {{ }} syntax errors

# Debug Step 3: Check line 45 of deployment.yaml
$ sed -n '40,50p' ./charts/my-app/templates/deployment.yaml

Line 45 likely has:
{{ .Values.something }}
But .Values.something doesn't exist!

# Fix: Check values.yaml
$ grep "something:" ./charts/my-app/values.yaml
# Missing value!

# Solution: Add to values.yaml or fix template
option A: values.yaml
  something: value

option B: template
  {{ default "default-value" .Values.something }}
```

### Scenario 2: Pods Not Starting After Helm Upgrade

```bash
# Check release status
$ helm status myapp
Status: FAILED

# See what went wrong
$ helm describe myapp
Error: template had a blank error

# Check pod status
$ kubectl get pods -l app=myapp
NAME          READY   STATUS       RESTARTS   AGE
myapp-xxxx    0/1     CrashLoop    5          2m

# Compare previous vs current
$ helm diff upgrade myapp ./chart
# Shows all changes

# Rollback to known-good
$ helm rollback myapp
Release "myapp" has been rolled back to previous release

# Debug the new version
$ helm template myapp ./chart > new-manifest.yaml
$ diff old-manifest.yaml new-manifest.yaml
# Spot the breaking change

# Common issues:
1. API version changed (e.g., policy/v1beta1 → policy/v1)
2. Resource moved to different namespace
3. Label selector changed (pods orphaned)
4. Storage class not available
```

### Scenario 3: Multiple Helm Charts Conflicting

```bash
# Error: "Duplicate key in ConfigMap"
# Happens when two releases try to create same resource

# Check what's installed
$ helm list -A
NAME     NAMESPACE   STATUS    CHART
app1     default     DEPLOYED  my-app-1.0.0
app2     default     DEPLOYED  my-app-1.0.0

# Both installed in default namespace with same resource names

# Debug Step 1: Examine templates
$ helm template app1 ./chart
metadata:
  name: configmap-data  # No suffix!

$ helm template app2 ./chart
metadata:
  name: configmap-data  # Same name!

# Both try to create: configmap-data
# Conflict!

# Fix: Use release name in resource names
template:
metadata:
  name: {{ .Release.Name }}-configmap-data

# Now app1 creates: app1-configmap-data
#     app2 creates: app2-configmap-data

# Re-install with fix
$ helm uninstall app2
$ helm install app2 ./chart-fixed
```

---

## 7. Interview Questions (15+ Scenario-Based)

### Level 1: Fundamentals

**Q1**: What's the difference between Chart version and appVersion?

**Answer**:
- **appVersion**: Version of the application (e.g., nginx v1.24.0)
  - What users care about
  - Specified in Chart.yaml like: appVersion: 1.24.0

- **Chart version**: Version of the Helm chart itself (e.g., my-nginx-chart v2.1.0)
  - What Helm maintainers version
  - When chart templates change, version bumps
  - Example: myapp chart v1.0.0 → v1.0.1 (bug fix in template)

Example:
```
Chart: my-app v1.2.0 (chart version)
appVersion: 2.5.1 (app version)

Later bug fix in template:
Chart: my-app v1.2.1 (bumped)
appVersion: 2.5.1 (same app)

App upgrade:
Chart: my-app v1.2.1
appVersion: 2.6.0 (bumped)
```

---

**Q2**: You have 3 environments (dev, staging, prod). How would you structure Helm charts?

**Answer**:
Option A: Single chart + multiple values files
```bash
helm install myapp ./chart -f values-prod.yaml
helm install myapp ./chart -f values-staging.yaml
helm install myapp ./chart -f values-dev.yaml

Pros: Single chart maintained, clear separation
Cons: Need 3 release installs
```

Option B: Umbrella chart with sub-charts
```
umbrella-chart/
├── Chart.yaml
├── requirements.yaml  (dependencies)
├── values/
│   ├── values-dev.yaml
│   ├── values-staging.yaml
│   └── values-prod.yaml
└── charts/
    ├── my-app/
    ├── database/
    └── cache/

helm install myapp ./umbrella-chart -f values/values-prod.yaml
```

Option C: Separate repos per environment
```bash
helm repo add myapp-dev https://...
helm repo add myapp-prod https://...

helm install myapp myapp-prod/myapp --version 1.2.3
```

Best practice: **Option A for small projects, Option B for microservices**

---

### Level 2: Advanced Usage

**Q3**: Explain Helm vs Kustomize. When would you use each?

**Answer**:

**Helm**:
- Full templating engine
- Package manager features
- Versioning, repos, releases
- Learning curve: Moderate
- Use when: Multi-team, need packaging

**Kustomize**:
- Overlay-based patching
- No templating engine
- Built into kubectl
- Learning curve: Low
- Use when: Simple overlays, no templating needed

Example:
```yaml
# Helm approach (templating)
deployment.yaml template:
replicas: {{ .Values.replicas }}

values-prod.yaml:
replicas: 5

# Kustomize approach (patching)
base/deployment.yaml:
replicas: 1

overlays/prod/kustomization.yaml:
patches:
- target:
    kind: Deployment
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
```

Choose: Helm if complex templating, Kustomize if simple overlays

---

### Level 3: Production

**Q4**: Design a Helm release strategy for canary deployments (5% → 25% → 50% → 100% traffic).

**Answer**:
Helm + Service Mesh (Istio approach):

```yaml
# Helm chart values for canary
canary:
  enabled: true
  weight:
    new: 5        # 5% traffic to new version
    stable: 95    # 95% to current version

# Template: virtualservice.yaml (Istio)
spec:
  hosts:
  - myapp
  http:
  - match:
    - uri:
        prefix: "/"
    route:
    - destination:
        host: myapp-v1
        subset: v1
      weight: {{ .Values.canary.weight.stable }}
    - destination:
        host: myapp-v2
        subset: v2
      weight: {{ .Values.canary.weight.new }}

# Helm upgrade sequence
Step 1: Deploy v2 pods (but receive 0% traffic)
$ helm upgrade myapp ./chart \
  --set image.tag=v2.0.0 \
  --set canary.weight.new=0

Step 2: Gradually increase traffic
$ helm upgrade myapp ./chart \
  --set image.tag=v2.0.0 \
  --set canary.weight.new=5  # 5%
$ # Monitor errors, latency

$ helm upgrade myapp ./chart \
  --set canary.weight.new=25  # 25%

$ helm upgrade myapp ./chart \
  --set canary.weight.new=100  # 100%

Step 3: Complete cutover
$ helm upgrade myapp ./chart \
  --set image.tag=v2.0.0
  # Remove v1 pods

Automation: Use flagger to auto-canary (progressive delivery)
```

---

## 8. Advanced Insights (Versioning, CI/CD Integration)

### Semantic Versioning for Charts

```yaml
# Chart.yaml
version: 1.2.3

Rules (Semantic Versioning):
1.2.3
│││
││└─ PATCH: Bug fixes in chart template
│└──MINOR: New features (backwards compatible)
└───MAJOR: Breaking changes (incompatible)

Example:
v1.0.0: Initial release
v1.0.1: Typo in template fixed (PATCH)
v1.1.0: Added new feature: monitoring (MINOR, backwards compatible)
  Old values still work, new values optional
v2.0.0: Changed all template fields (MAJOR, breaking)
  Old values won't work!

Version promotion:
production: v1.2.3 (stable)
staging: v1.3.0 (release candidate)
dev: v1.4.0-beta.1 (in development)

Only bump MINOR/MAJOR in stable branches
```

### Helm + GitOps CI/CD Pipeline

```
Repository Structure:
├── helm/
│   └── my-app/
│       ├── Chart.yaml (version: 1.0.0)
│       ├── values.yaml
│       └── templates/
│
├── deployment/
│   ├── helmfile.yaml
│   └── values/
│       ├── dev.yaml
│       ├── staging.yaml
│       └── prod.yaml
│
└── .github/workflows/
    ├── helm-lint.yaml
    ├── helm-release.yaml
    └── helm-deploy.yaml

Workflow:
Developer creates PR:
  └─ Modifies: templates/deployment.yaml
  └─ PR triggers: helm-lint.yaml
     ├─ $ helm lint ./helm/my-app
     ├─ $ helm template myapp ./helm/my-app > /tmp/manifest.yaml
     ├─ $ kubeval /tmp/manifest.yaml (schema validation)
     └─ Results posted to PR

Approve + Merge to main:
  └─ Triggers: helm-release.yaml
     ├─ $ helm package ./helm/my-app
     ├─ Bumps version: 1.0.0 → 1.0.1
     ├─ $ helm push my-app-1.0.1.tgz oci://registry
     ├─ Creates GitHub release with chart
     └─ Git tag: v1.0.1

ArgoCD watches Git + Helm repo:
  └─ Detects new chart: my-app-1.0.1
  └─ Updates helmfile.yaml
  └─ Automatically upgrades:
     $ helm upgrade myapp my-app \
       --version 1.0.1 \
       -f values/prod.yaml
  └─ Monitors upgrade, auto-rolls back if fails

Example CI/CD Actions:

helm-lint.yaml:
---
on: [pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: azure/setup-helm@v3
    - run: helm lint ./helm/my-app
    - run: helm template test ./helm/my-app

helm-release.yaml:
---
on:
  push:
    branches: [main]
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: azure/setup-helm@v3
    - run: helm package ./helm/my-app
    - run: helm push my-app-*.tgz oci://...
```

### Chart Repository Management

```yaml
#1. Public Repository (Helm Hub)
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm repo update
$ helm install mysql bitnami/mysql

#2. Private Repository (Enterprise)
# Option A: Git-based (GitHub releases)
$ helm repo add mycompany https://github.com/mycompany/helm-charts/releases/download/

# Option B: OCI Registry (Docker registry)
$ helm repo add mycompany oci://ghcr.io/mycompany/charts

# Option C: Artifactory, ChartMuseum
$ helm repo add mycompany https://artifactory.example.com/artifactory/helm-repo

#3. Chart Dependencies
Chart.yaml:
---
dependencies:
- name: mysql
  version: "8.0.0"
  repository: "https://charts.bitnami.com/bitnami"

# Fetch dependencies
$ helm dependency update
$ helm dependency list

# Lock file (like package-lock.json)
Chart.lock keeps exact versions
Ensures reproducible deployments

#4. Sharing Charts Across Teams
Internal Helm Repository (private):
├─ webapp/1.0.0
├─ api/2.1.0
├─ database/5.0.0
├─ monitoring/1.5.0

Teams pull from internal repo:
$ helm install api mycompany/api --version 2.1.0
```

---

## Conclusion

Helm fundamentals rest on three pillars:
1. **Packaging**: Charts package Kubernetes manifests
2. **Templating**: Values render dynamic YAML
3. **Release Management**: Track, upgrade, rollback

Master these three and you can:
- Deploy complex applications reliably
- Manage multiple environments with clarity
- Collaborate across teams with versioned packages
- Automate deployments through CI/CD

Helm is the **kubectl apply** for the modern Kubernetes world!
