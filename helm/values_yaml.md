# Values.yaml: Configuration Management Deep Dive

## 1. Core Concept

Values.yaml is the **configuration contract** between chart and user:

1. **Defines chart behavior** (what can be configured)
2. **Provides defaults** (fallbacks if not overridden)
3. **Documents options** (comments explaining each value)
4. **Enables reusability** (different configs = different deployments)

### The Configuration Hierarchy

```
chart/
├── values.yaml              # Chart DEFAULT values
└── templates/               # Use: {{ .Values.key }}

User provides:
├── -f values-prod.yaml      # Override values
├── --set key=value          # CLI override
└── Merge order: defaults < files < CLI (CLI wins)

Result in cluster:
├── ConfigMap: release-values (stores release values)
├── Secret: release-data (stores secrets)
└── Deployed with merged values
```

### Simple Values File

```yaml
# values.yaml
replicaCount: 3              # How many pod replicas

image:
  repository: nginx          # Container image repo
  tag: "1.24.0"             # Image version
  pullPolicy: IfNotPresent  # When to pull image

service:
  type: ClusterIP            # Service type
  port: 80                  # Exposed port

resources:
  requests:                 # Minimum resources
    cpu: 100m
    memory: 128Mi
  limits:                   # Maximum resources
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false            # Enable HPA?
  minReplicas: 1
  maxReplicas: 10

# Usage in template
# {{ .Values.replicaCount }} → 3
# {{ .Values.image.repository }} → nginx
# {{ .Values.service.port }} → 80
```

---

## 2. Why Values.yaml is Used

### Problem Without Values.yaml

**Hardcoded values in templates**:

```yaml
# deployment.yaml (no templating)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 5           # Hardcoded!
  template:
    spec:
      containers:
      - name: api
        image: myapp:v1.2.3  # Hardcoded!
        resources:
          requests:
            cpu: 1000m    # Hardcoded!
            memory: 1Gi   # Hardcoded!

# To change for staging environment:
# 1. Copy file
# 2. Edit replicas: 2
# 3. Edit image: v1.2.2
# 4. Edit resources: 500m/512Mi

# Maintenance nightmare!
# - Duplicate files
# - Easy to miss updates
# - Error-prone
```

### Solution: Externalize to Values.yaml

```yaml
# Single template + multiple values files
# templates/deployment.yaml
replicas: {{ .Values.replicaCount }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
cpu: {{ .Values.resources.requests.cpu }}
memory: {{ .Values.resources.requests.memory }}

# values.yaml (defaults)
replicaCount: 1
image:
  repository: myapp
  tag: latest
resources:
  requests:
    cpu: 100m
    memory: 128Mi

# values-staging.yaml (override)
replicaCount: 2
image:
  tag: v1.2.2
resources:
  requests:
    cpu: 500m
    memory: 512Mi

# Deployment:
$ helm install api ./chart -f values-staging.yaml
# Uses template + values-staging.yaml
# No duplication!
```

**Benefits**:
- ✅ Single source of truth (one template)
- ✅ Multiple configurations (values files)
- ✅ Easy to override (no file copying)
- ✅ Git-friendly (track values separately)

---

## 3. Internal Working (Values Merging, Defaults, Resolution)

### Values Merging Process

```
Input:
├── Chart values.yaml (built-in)
├── User -f values-prod.yaml
└── User --set key=val

Merge order (highest priority wins):
         ↑ --set flags (highest priority)
         │ (overrides everything)
         ├─ -f user files (middle priority)
         │ (overrides defaults)
         └─ Chart defaults (lowest priority)
         
Example:
Chart defaults:
  replicaCount: 1
  image:
    tag: latest

User file (-f values-prod.yaml):
  replicaCount: 5

User CLI (--set):
  image.tag: v1.2.3

Final merged values:
  replicaCount: 5          ← From user file
  image:
    tag: v1.2.3            ← From --set
```

### Deep Merge Behavior

```yaml
# Chart: values.yaml
database:
  host: localhost
  port: 5432
  sslMode: disable
  timeoutSeconds: 30

# User override (-f values.yaml)
database:
  host: prod-db.example.com
  sslMode: require

# Result (deep merge, not replace):
database:
  host: prod-db.example.com    ← Overridden
  port: 5432                   ← Kept from default
  sslMode: require             ← Overridden
  timeoutSeconds: 30           ← Kept from default

# Problem: Merging lists (NOT deep merged)
# Chart: environment: ["LOG_LEVEL=debug", "MODE=dev"]
# User:  environment: ["LOG_LEVEL=info"]
# Result: environment: ["LOG_LEVEL=info"]  ← List REPLACED, not merged!
```

### Type Coercion in Values

```yaml
# values.yaml
# All treated as STRINGS initially

# String
name: myapp          → "myapp"

# Number (quote to force string)
port: 8080          → 8080 (number)
port: "8080"        → "8080" (string)

# Boolean
enabled: true       → true (boolean)
enabled: "true"     → "true" (string, NOT boolean!)

# Template interpretation
{{ .Values.port }}              → 8080
{{ .Values.port | quote }}      → "8080"
{{ .Values.name }}              → myapp
{{ .Values.enabled }}           → true
{{ if .Values.enabled }} ...    → true (condition works)

# Common mistake
{{ .Values.str | toInt }}       → Convert string to int
{{ .Values.num | toString }}    → Convert number to string
```

### Accessing Values in Templates

```yaml
# Scalar values
{{ .Values.name }}                    # myapp
{{ .Values.port }}                    # 8080

# Nested values (dot notation)
{{ .Values.database.host }}           # localhost
{{ .Values.image.repository }}        # nginx

# Array access
{{ index .Values.servers 0 }}         # First server
{{ (index .Values.servers 0).host }}  # With nested field

# Map iteration
{{ range $key, $value := .Values.labels }}
  {{ $key }}: {{ $value }}
{{ end }}

# With dotted keys (from --set)
# --set database.replicas=3
# {{ .Values.database.replicas }} → 3

# Complex --set
# --set "env[0].name=LOG_LEVEL" --set "env[0].value=debug"
# {{ index .Values.env 0 }} → {name: LOG_LEVEL, value: debug}
```

---

## 4. Real-World Values Scenarios

### Scenario 1: Multi-Environment Configuration

```yaml
# File structure
helm/
├── my-app/
│   ├── Chart.yaml
│   ├── values.yaml              # DEFAULTS
│   └── templates/
│
└── environments/
    ├── values-dev.yaml
    ├── values-staging.yaml
    ├── values-prod.yaml
    └── values-prod-replica.yaml  # Prod multi-region

# values.yaml (defaults)
replicaCount: 1
image:
  repository: myapp
  tag: latest
resources:
  requests:
    cpu: 100m
    memory: 128Mi
database:
  host: localhost
  port: 5432

# environments/values-dev.yaml
# Most values use defaults, just override some
image:
  tag: latest  # Always latest in dev
database:
  host: localhost
  ## port: 5432 (use default)

# environments/values-prod.yaml
replicaCount: 5
image:
  tag: v1.2.3  # Locked version
resources:
  requests:
    cpu: 1000m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 2Gi
database:
  host: prod-db.internal
  sslMode: require
  connectionPool: 100

# Deployment commands
$ helm install myapp ./my-app                    # Uses defaults
$ helm install myapp ./my-app -f environments/values-dev.yaml
$ helm install myapp ./my-app -f environments/values-prod.yaml
```

### Scenario 2: Feature Flags via Values

```yaml
# values.yaml
features:
  authentication:
    enabled: true
    oauth2:
      enabled: false
      provider: google
    
  logging:
    enabled: true
    level: info
    outputs:
      - stdout
      - file
    
  monitoring:
    enabled: true
    prometheus: true
    jaegar: false

# templates/deployment.yaml
spec:
  containers:
  - name: app
    image: {{ .Values.image }}
    
    {{- if .Values.features.authentication.enabled }}
    env:
    - name: AUTH_ENABLED
      value: "true"
    {{- if .Values.features.authentication.oauth2.enabled }}
    - name: OAUTH2_ENABLED
      value: "true"
    - name: OAUTH2_PROVIDER
      value: {{ .Values.features.authentication.oauth2.provider }}
    {{- end }}
    {{- end }}

# environments/values-prod.yaml
features:
  authentication:
    enabled: true
    oauth2:
      enabled: true  # Enable OAuth in prod!
      provider: oauth2.example.com
  
  monitoring:
    jaegar: true    # Enable distributed tracing in prod
```

### Scenario 3: Values Schema Validation

```yaml
# chart-schema.json (Helm 3.0+)
# Validates values against JSON Schema

{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100,
      "description": "Number of replicas"
    },
    "image": {
      "type": "object",
      "properties": {
        "repository": {
          "type": "string",
          "pattern": "^[a-z0-9]+(/[a-z0-9]+)*$"
        },
        "tag": {
          "type": "string",
          "description": "Image tag (use specific version, not 'latest')"
        }
      },
      "required": ["repository", "tag"]
    }
  },
  "required": ["replicaCount", "image"]
}

# Validation when user provides values
$ helm install myapp ./chart -f values-invalid.yaml
# Error: replicaCount must be between 1-100
# Error: image.tag is required

$ helm install myapp ./chart \
  --set replicaCount=5 \
  --set image.repository=myapp \
  --set image.tag=v1.2.3
# Success: All validations pass
```

### Scenario 4: Values Templating (Advanced)

```yaml
# values.yaml with templates
appName: myapp
appVersion: 1.2.3

# Computed values (using Helm magic)
labels:
  app: {{ .Chart.Name }}        # References chart name
  version: {{ .Chart.Version }} # References chart version

# Don't do this! Values are not templated!
# {{ }} in values.yaml is just a string!

# Correct approach: Put logic in template
# values.yaml
appName: myapp

# template/deployment.yaml
metadata:
  labels:
    app: {{ include "mychart.labels" . }}
    version: {{ .Chart.Version }}
```

---

## 5. Common Values.yaml Mistakes

### Mistake 1: Using `latest` Tag

```yaml
# ❌ WRONG
image:
  repository: myapp
  tag: latest         # Unpredictable version!

# Problems:
1. Each pod pull might get different image
2. Helm upgrade with no image.tag change = different image
3. Impossible to debug which version is running
4. Rollback doesn't guarantee same image

# ✅ CORRECT
image:
  repository: myapp
  tag: v1.2.3         # Specific version

# For dev only:
image:
  tag: develop        # At least a meaningful tag
```

### Mistake 2: Storing Secrets in values.yaml

```yaml
# ❌ WRONG - values.yaml
database:
  username: admin
  password: super-secret-123  # PLAINTEXT!
  
api:
  apiKey: sk_live_abcd1234     # EXPOSED!

# Problems:
1. Stored in Git (visible to all!)
2. Visible in helm history
3. Visible in ConfigMap in cluster
4. Security audit nightmare

# ✅ CORRECT - Use external secret management
# values.yaml
database:
  credentialsSecret:
    name: db-credentials
    key: connection-string

# Create secret separately:
$ kubectl create secret generic db-credentials \
  --from-literal=connection-string="user:pass@host:5432"

# Or use ExternalSecrets operator:
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  secretStoreRef:
    name: vault
    kind: SecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
  - secretKey: connection-string
    remoteRef:
      key: prod/database/credentials
```

### Mistake 3: Over-Templating Values

```yaml
# ❌ WRONG - values.yaml (values are NOT templated!)
appName: myapp
replicaCount: 3

# This LOOKS like it should work but doesn't:
name: {{ .Release.Name }}-app    # {{ }} NOT processed in values!
labels:
  app: {{ .appName }}             # NOT processed!

# Output: name: "{{ .Release.Name }}-app"  (literal string!)

# ✅ CORRECT - Move logic to template
# values.yaml
appName: myapp
replicaCount: 3

# template/deployment.yaml
metadata:
  name: {{ .Release.Name }}-app  # Processed here!
  labels:
    app: {{ .Values.appName }}   # Processed here!
```

### Mistake 4: Deep Nesting Makes Values Hard to Use

```yaml
# ❌ WRONG - Too deeply nested
config:
  application:
    logging:
      handlers:
        console:
          format: json
          level: info
        file:
          path: /var/log/app.log
          level: debug

# Access: {{ .Values.config.application.logging.handlers.console.format }}
# Hard to remember, easy to typo

# ✅ CORRECT - Flatten where possible
logging:
  console_format: json
  console_level: info
  file_path: /var/log/app.log
  file_level: debug

# OR use semantic grouping
app:
  logging:
    console:
      format: json
      level: info
    file:
      path: /var/log/app.log
      level: debug

# Access: {{ .Values.app.logging.console.format }}
# Clear and reasonable depth
```

### Mistake 5: Inconsistent Value Types

```yaml
# ❌ WRONG - Inconsistent handling
environment: "production"    # String
debug: "false"               # String (should be boolean!)
port: 8080                   # Number
timeout: "30"                # String (should be number!)

# In template (confusing!)
{{ if .Values.debug }}       # Works? (string "false" is truthy!)
  debug: true
{{ end }}

# ✅ CORRECT - Consistent types
environment: production      # String
debug: false                 # Boolean
port: 8080                   # Number
timeout: 30                  # Number

# In template (clear)
{{ if .Values.debug }}       # Only true if boolean true
  debug: true
{{ end }}
```

---

## 6. Debugging Values Issues

### Scenario 1: Values Not Showing in Deployment

```bash
# Problem: Expected value {{ .Values.something }} not in manifest

# Step 1: Check rendered manifest
$ helm template myapp ./chart -f values.yaml > /tmp/manifest.yaml
$ grep "something" /tmp/manifest.yaml

# Not found!

# Step 2: Check values file
$ grep "something" values.yaml
# Not defined!

# Step 3: Check template
$ grep "something" ./chart/templates/*.yaml
# {{ .Values.something }}

# Step 4: Check if value is optional
If optional: Add to values.yaml
If required: Add to values.yaml and Chart.yaml schema

# Fix:
values.yaml:
something: value

# Re-test:
$ helm template myapp ./chart | grep "something"
# Now appears!
```

### Scenario 2: Override Not Working

```bash
# Problem: $ helm install myapp ./chart --set xyz=abc
# But xyz not in final deployment

# Step 1: Check original set command
--set xyz=abc

# Step 2: Render with debug
$ helm install myapp ./chart \
  --set xyz=abc \
  --dry-run --debug | grep "xyz"

# xyz not in values!

# Step 3: Check template usage
$ grep "xyz" ./chart/templates/*.yaml
# Not used anywhere!

# Step 4: Check if key is nested
--set .Values.section.xyz
# Should be:
--set section.xyz=abc

# Step 5: Test with actual template
$ helm template myapp ./chart \
  --set section.xyz=abc | grep "abc"

# If still not working, verify template exists
# and references .Values.section.xyz
```

### Scenario 3: Values Conflict Between Releases

```bash
# Problem: Two releases with same chart, both modifying shared resource

$ helm list
NAME     NAMESPACE   CHART
app1     default     my-app
app2     default     my-app

# Both create: configmap named "app-config"
# Conflict!

# Step 1: Check values for both
$ helm get values app1
configMapName: app-config        # Hardcoded!

$ helm get values app2
configMapName: app-config        # Same name!

# Step 2: Use release name to differentiate
values.yaml:
configMapName: {{ .Release.Name }}-config

# Step 3: Upgrade releases
$ helm upgrade app1 ./chart  # Creates: app1-config
$ helm upgrade app2 ./chart  # Creates: app2-config

# No more conflict!
```

---

## 7. Interview Questions (15+ Scenario-Based)

### Level 1: Values Basics

**Q1**: What's the order of precedence when merging values?

**Answer**:
```
Highest priority (wins):
┌─ --set flags         (CLI overrides)
├─ -f values files     (User provided files)
└─ values.yaml         (Chart defaults)
Lowest priority (ignored):
```

Example:
```yaml
# Chart defaults: replicaCount: 1
# User file -f values-prod.yaml: replicaCount: 5
# CLI: --set replicaCount: 10

# Final: replicaCount: 10  (--set wins)
```

---

**Q2**: How would you structure values for a multi-database application?

**Answer**:
```yaml
# Option 1: Flat structure
database_primary_host: db-primary.exemple.com
database_primary_port: 5432
database_replicas_host: db-replica.example.com
database_replicas_port: 5432
cache_host: cache.example.com
cache_port: 6379

# Option 2: Semantic grouping (recommended)
database:
  primary:
    host: db-primary.example.com
    port: 5432
    sslMode: require
  replicas:
    host: db-replica.example.com
    port: 5432
    sslMode: require

cache:
  host: cache.example.com
  port: 6379
  ttl: 3600

# Usage in template:
{{ .Values.database.primary.host }}
{{ .Values.cache.host }}
```

---

### Level 2: Production Values

**Q3**: Design a values structure for a multi-tenant SaaS application.

**Answer**:
```yaml
# values.yaml (multi-tenant SaaS)
global:
  domain: example.com
  
tenants:
  - name: acme-corp
    tier: enterprise
    replicas: 5
    resources:
      cpu: 2000m
      memory: 4Gi
    features:
      sso: true
      customBranding: true
      analytics: true
      
  - name: startup-inc
    tier: starter
    replicas: 1
    resources:
      cpu: 500m
      memory: 1Gi
    features:
      sso: false
      customBranding: false
      analytics: false

# Template: templates/deployment.yaml
{{ range .Values.tenants }}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-{{ .name }}
spec:
  replicas: {{ .replicas }}
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: {{ .resources.cpu }}
            memory: {{ .resources.memory }}
        env:
        - name: TENANT_NAME
          value: {{ .name }}
        - name: SSO_ENABLED
          value: {{ .features.sso | quote }}
{{ end }}

# Result: 2 deployments
# - app-acme-corp (5 replicas, 2000m CPU)
# - app-startup-inc (1 replica, 500m CPU)
```

---

**Q4**: How would you version values files to track changes?

**Answer**:
```yaml
# Git approach (recommended for production)
.
├── helm/
│   └── my-app/
│       ├── Chart.yaml (version: 1.2.3)
│       └── values.yaml
│
└── helm-values/
    ├── prod/
    │   ├── base-values.yaml    # Committed
    │   ├── secrets.encrypted   # Git-crypt encrypted
    │   └── CHANGELOG.md        # Track changes
    │
    ├── staging/
    │   ├── values.yaml
    │   └── CHANGELOG.md
    │
    └── dev/
        └── values.yaml

# CHANGELOG.md example:
## 2024-04-10
- Increased prod replicas: 3 → 5
- Added memory limit: 2Gi
- Enabled new feature flag

## 2024-04-05
- Updated image tag: v1.1.0
- Modified database connection pool: 50 → 100

# Usage:
$ helm upgrade myapp ./helm/my-app \
  -f helm-values/prod/base-values.yaml

# Version tracking:
$ git log -p helm-values/prod/base-values.yaml
# Shows all value changes with dates/authors

# For secrets:
$ git-crypt unlock                              # Decrypt
$ helm upgrade myapp ./helm/my-app \
  -f helm-values/prod/base-values.yaml \
  -f helm-values/prod/secrets.encrypted        # Encrypted secrets
```

---

## 8. Advanced Insights (Values in CI/CD)

### Pattern 1: Values Override Matrix (Test All Combos)

```bash
#!/bin/bash
# Test all environment + feature combinations

environments=("dev" "staging" "prod")
features=("basic" "premium" "enterprise")

for env in "${environments[@]}"; do
  for feature in "${features[@]}"; do
    echo "Testing: $env + $feature"
    
    helm template myapp ./chart \
      -f values/values-${env}.yaml \
      -f values/values-${feature}-features.yaml \
      > /tmp/manifest-${env}-${feature}.yaml
    
    kubeval /tmp/manifest-${env}-${feature}.yaml
    [ $? -ne 0 ] && echo "FAILED" && exit 1
  done
done

echo "All combinations validated!"
```

### Pattern 2: Secrets Management Pipeline

```bash
# Separate values from secrets

# Committed to Git:
values-prod.yaml
---
database:
  host: prod-db.internal
  port: 5432
  ssl: true
  # credentialsSecret: filled at deploy time

# Not committed (secrets only):
# Using ArgoCD + sealed-secrets
secrets-prod.yaml
---
database:
  credentialsSecret:
    name: prod-db-credentials
    key: connection-string

# Deployment:
$ helm upgrade myapp ./chart \
  -f values/values-prod.yaml \
  -f values/secrets-prod.yaml   # Decrypted by CI/CD
```

### Pattern 3: Helm Values in GitOps (ArgoCD)

```yaml
# ArgoCD ApplicationSet manages multiple values files
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: deploy-all-environments
spec:
  generators:
  - list:
      elements:
      - name: dev
        values: values-dev.yaml
      - name: staging
        values: values-staging.yaml
      - name: prod
        values: values-prod.yaml
        
  template:
    metadata:
      name: 'myapp-{{name}}'
    spec:
      source:
        repoURL: https://github.com/myorg/helm-charts
        path: helm/my-app
        helm:
          valueFiles:
          - 'values/{{values}}'
          
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{name}}'

# Result: 3 applications deployed
# - myapp-dev (using values-dev.yaml)
# - myapp-staging (using values-staging.yaml)
# - myapp-prod (using values-prod.yaml)
```

---

## Conclusion

Values.yaml is the **configuration contract** between chart and user:

**Key principles**:
1. **Externalize**: Move all configurable values to values.yaml
2. **Document**: Comment what each value does
3. **Separate**: Secrets ≠ configuration
4. **Override**: Support file-based and CLI overrides
5. **Version**: Track value changes in Git/changelog

Master values management and you have:
- ✅ Reusable charts across environments
- ✅ Version-controlled configuration
- ✅ Clear audit trails
- ✅ Production-ready deployments

**Best practice**: Use values.yaml as your "infrastructure documentation" - anyone reading it should understand what the application can be configured to do!
