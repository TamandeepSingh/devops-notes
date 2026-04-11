# Helm Templating: Deep Dive into the Template Language

## 1. Core Concept

Helm templating enables **declarative infrastructure-as-code** by:

1. **Dynamic YAML Generation**: Convert template + values → pure YAML
2. **Reusability**: Single template works for all environments
3. **Logic**: Conditional rendering, loops, functions
4. **Maintainability**: DRY principle (Don't Repeat Yourself)

### The Templating Engine

Helm uses **Go's text/template** library (similar to Jinja2):

```
Input:
├─ templates/deployment.yaml (Go template syntax)
└─ values.yaml (configuration data)

Helm Template Engine:
└─ Processes: {{ .Values.key }}, {{ if }}, {{ range }}
└─ Renders to:

Output:
└─ deployment.yaml (pure YAML, no template syntax)
```

### Simple Example

```yaml
# Template: templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}

# Values: values.yaml
replicaCount: 3

# Rendered Output (after helm template)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-app
  namespace: default
  labels:
    app: my-app
spec:
  replicas: 3
```

---

## 2. Why Helm Templating is Used

### Problem Without Templating

**Before Helm** (manual YAML management):

```
For 3 environments = 3 complete YAML files
```

```yaml
# deployment-dev.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-dev
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest

# deployment-staging.yaml  (almost identical!)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-staging
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest

# deployment-prod.yaml  (almost identical!)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-prod
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.2.3   # Different from dev!

# Copy-paste errors!
# Update one, forget to update others
# 95% duplication
```

### Solution: Helm Templating

```yaml
# Single template: templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: app
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}

# Three values files:
# values-dev.yaml
replicaCount: 1
image:
  repository: myapp
  tag: latest

# values-prod.yaml
replicaCount: 5
image:
  repository: myapp
  tag: v1.2.3
```

**Benefits**:
- ✅ Single source of truth (one template)
- ✅ Environment-specific overrides (values files)
- ✅ DRY principle (no duplication)
- ✅ Easy to maintain (change template once)

---

## 3. Internal Working (Template Rendering)

### Template Syntax Deep Dive

#### 1. Variable Substitution

```yaml
# Basic variable
{{ .Release.Name }}        # Output: my-app

# Nested values
{{ .Values.database.host }} # Output: localhost

# With default
{{ .Values.something | default "fallback" }}
# If .Values.something empty, use "fallback"

# Accessing array index
{{ index .Values.servers 0 }}  # First server

Built-in variables available:
.Release.Name          # Release name (e.g., "my-app")
.Release.Namespace     # Namespace (e.g., "default")
.Release.IsUpgrade     # Boolean: is this upgrade?
.Release.IsInstall     # Boolean: is this install?
.Release.Revision      # Revision number (1, 2, 3...)

.Chart.Name            # Chart name
.Chart.Version         # Chart version
.Chart.AppVersion      # Application version

.Values.*              # User-provided values
.Files.*               # Access to files
.Capabilities.*        # Cluster capabilities
```

#### 2. String Functions (Sprig Library)

```yaml
# String manipulation
{{ .Release.Name | upper }}           # Output: MY-APP
{{ .Values.appName | lower }}         # Output: myapp
{{ .Values.title | quote }}           # Output: "mytitle"
{{ .Values.description | trunc 10 }}  # Truncate to 10 chars

# String operations
{{ printf "%s-%s" .Release.Name "suffix" }}  # Format string

# Joining/splitting
{{ join "," .Values.tags }}           # Join array with comma
{{ split "," .Values.items }}         # Split string by comma

# Regular expressions
{{ .Values.email | regexMatch ".*@example.com" }}  # Boolean

# Encoding
{{ .Values.password | b64enc }}       # Base64 encode
{{ .Values.encodedPassword | b64dec }}  # Base64 decode
```

#### 3. Conditional Logic

```yaml
# if/else/else if
{{ if .Values.enabled }}
  name: {{ .Release.Name }}
{{ else if .Values.standby }}
  name: {{ .Release.Name }}-standby
{{ else }}
  name: {{ .Release.Name }}-disabled
{{ end }}

# Multiple conditions
{{ if and .Values.a .Values.b }}
  Both are true
{{ end }}

{{ if or .Values.a .Values.b }}
  At least one is true
{{ end }}

{{ if not .Values.disabled }}
  Not disabled
{{ end }}

# Practical example
{{ if eq .Release.Namespace "prod" }}
  replicas: 5
{{ else }}
  replicas: 1
{{ end }}
```

#### 4. Loops (Range)

```yaml
# Loop through array
ports:
{{ range .Values.ports }}
  - name: {{ .name }}
    port: {{ .port }}
    protocol: {{ .protocol | default "TCP" }}
{{ end }}

Values:
---
ports:
  - name: http
    port: 80
  - name: https
    port: 443
    protocol: TCP

Output:
---
ports:
  - name: http
    port: 80
    protocol: TCP
  - name: https
    port: 443
    protocol: TCP

# Loop through map (dictionary)
{{ range $key, $value := .Values.config }}
  {{ $key }}: {{ $value }}
{{ end }}

# Loop with index
{{ range $index, $item := .Values.items }}
  Item {{ $index }}: {{ $item }}
{{ end }}

# Nested loops
{{ range .Values.services }}
  service: {{ .name }}
  {{ range .ports }}
    - port: {{ .port }}
  {{ end }}
{{ end }}
```

#### 5. Template Functions (with examples)

```yaml
# Include template
{{ include "mychart.labels" . }}
# Includes _helpers.tpl content with current context

# Named value
{{ with .Values.database }}
  host: {{ .host }}
  port: {{ .port }}
  # Short form: .host instead of .Values.database.host
{{ end }}

# Accessing current key in context
{{ range $key, $value := .Values }}
  Current key: {{ $key }}
{{ end }}

# Safe template execution (don't error on nil)
{{ .Values.maynotexist | default "" }}

# Type checking
{{ if typeIs "map[string]interface {}" .Values.config }}
  It's a map!
{{ end }}
```

#### 6. Advanced: Custom Helpers

```yaml
# _helpers.tpl (reusable templates)
{{/*
Expand the name of the chart.
*/}}
{{- define "mychart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create labels
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
{{ include "mychart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

# Usage in templates/deployment.yaml
metadata:
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
```

### Rendering Flow

```
Step 1: Load all files
├─ Chart.yaml
├─ values.yaml (defaults)
├─ templates/_helpers.tpl (custom functions)
├─ templates/deployment.yaml
├─ templates/service.yaml
└─ User-provided values (--set, -f values-prod.yaml)

Step 2: Merge values (deep merge)
Default values: {replicaCount: 1, image.tag: latest}
User values:    {replicaCount: 3}
Result:         {replicaCount: 3, image.tag: latest}  ← User overrides

Step 3: Build context object
.Release = {Name: "my-app", Namespace: "default", ...}
.Chart = {Name: "mychart", Version: "1.0.0", ...}
.Values = {replicaCount: 3, image.tag: latest, ...}
.Files = {file1: content1, ...}

Step 4: Render each template
For each file in templates/:
  └─ Process Go template syntax
  └─ Replace {{ }} with values
  └─ Output pure YAML

Step 5: Validate output
└─ Check YAML syntax
└─ Validate against Kubernetes schema (optional)

Step 6: Output manifests
Deployment.yaml
Service.yaml
ConfigMap.yaml
(all pure YAML, no template syntax)
```

---

## 4. Real-World Templating Scenarios

### Scenario 1: Environment-Specific Resource Limits

```yaml
# templates/deployment.yaml
resources:
  requests:
    cpu: {{ .Values.resources.requests.cpu }}
    memory: {{ .Values.resources.requests.memory }}
  limits:
    cpu: {{ .Values.resources.limits.cpu }}
    memory: {{ .Values.resources.limits.memory }}

# values-dev.yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi

# values-prod.yaml
resources:
  requests:
    cpu: 1000m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 2Gi

# Rendering:
# Dev:  100m → 200m CPU, 128Mi → 256Mi Memory
# Prod: 1000m → 2000m CPU, 1Gi → 2Gi Memory
```

### Scenario 2: Conditional Module Inclusion

```yaml
# templates/deployment.yaml
spec:
  containers:
  - name: app
    image: {{ .Values.image }}

{{ if .Values.sidecar.enabled }}
  - name: sidecar
    image: {{ .Values.sidecar.image }}
    volumeMounts:
    - name: sidecar-config
      mountPath: /etc/sidecar
{{ end }}

{{ if .Values.logging.enabled }}
  - name: logging-agent
    image: {{ .Values.logging.driver }}
{{ end }}

volumes:
{{ if .Values.sidecar.enabled }}
- name: sidecar-config
  configMap:
    name: {{ .Release.Name }}-sidecar
{{ end }}

# values.yaml
sidecar:
  enabled: true
  image: sidecar:v1

logging:
  enabled: false
  driver: filebeat:7.0

# Rendering: Sidecar included, logging excluded
```

### Scenario 3: Dynamic Port Configuration

```yaml
# templates/service.yaml
ports:
{{ range .Values.service.ports }}
  - name: {{ .name }}
    port: {{ .port }}
    targetPort: {{ .targetPort | default .port }}
    protocol: {{ .protocol | default "TCP" }}
{{ end }}

# values.yaml
service:
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
      protocol: TCP
    - name: metrics
      port: 9090

# Rendering (nindent handles indentation):
ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  - name: metrics
    port: 9090
    targetPort: 9090
    protocol: TCP
```

### Scenario 4: Multi-Replica StatefulSet

```yaml
# templates/statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  serviceName: {{ include "mychart.fullname" . }}
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      {{ if .Values.affinity }}
      affinity:
        {{- toYaml .Values.affinity | nindent 8 }}
      {{ end }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: {{ .Values.storage }}

# values.yaml
replicas: 3
image:
  repository: mysql
  tag: "8.0"
storage: 10Gi
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - mysql
      topologyKey: kubernetes.io/hostname
```

---

## 5. Common Templating Mistakes

### Mistake 1: Incorrect Indentation

```yaml
# ❌ WRONG
spec:
  containers:
  - name: app
{{ .Values.resources }}

# Renders as:
spec:
  containers:
  - name: app
resources:              # Wrong indentation!
  requests:
    cpu: 100m

# ✅ CORRECT
spec:
  containers:
  - name: app
    {{- toYaml .Values.resources | nindent 4 }}

# Renders as:
spec:
  containers:
  - name: app
    resources:          # Correct indentation
      requests:
        cpu: 100m

# Indentation helpers:
{{ toYaml .Values.x | nindent N }}   # New indented
{{ toYaml .Values.x | indent N }}    # Keep existing indent
```

### Mistake 2: Loop Context Loss

```yaml
# ❌ WRONG
labels:
{{ range .Values.tags }}
  app: {{ .Release.Name }}  # ERROR: .Release not available in loop!
  tag: {{ . }}
{{ end }}

# ✅ CORRECT (using context variable)
labels:
{{ range .Values.tags }}
  app: {{ $.Release.Name }}  # $ refers to root context
  tag: {{ . }}               # . is current item
{{ end }}

# ✅ BETTER (name the context)
labels:
{{ range $key, $value := .Values.config }}
  {{ $key }}: {{ $value }}
  app: {{ $.Release.Name }}
{{ end }}
```

### Mistake 3: Missing Quote Handling

```yaml
# ❌ WRONG
environment:
  - name: DB_URL
    value: {{ .Values.database.url }}
    
# If value is: "localhost:5432" (with spaces), renders as:
    value: localhost:5432  # YAML parse error!

# ✅ CORRECT
environment:
  - name: DB_URL
    value: {{ .Values.database.url | quote }}   # Adds quotes
    # Or quote in values.yaml:
    # database.url: "localhost:5432"
```

### Mistake 4: Empty Value Confusion

```yaml
# ❌ WRONG
{{ if .Values.replicaCount }}
  replicas: {{ .Values.replicaCount }}
{{ end }}

# If replicaCount: 0, condition is false!
# Zero is falsy in Go templates

# ✅ CORRECT
{{ if ne .Values.replicaCount nil }}
  replicas: {{ .Values.replicaCount }}
{{ end }}

# Or use explicit comparison
{{ if .Values.replicaCount }}  # Works for 1, 2, 3...
  replicas: {{ .Values.replicaCount }}
{{ else }}
  replicas: 1  # Default
{{ end }}
```

### Mistake 5: Whitespace/Control Issues

```yaml
# ❌ WRONG (extra blank lines in output)
apiVersion: v1
kind: ConfigMap
{{ if .Values.data }}
config: |
  {{ .Values.data }}
{{ end }}

# Renders with:
  - blank lines
  - extra whitespace

# ✅ CORRECT (control whitespace with {{- and -}})
apiVersion: v1
kind: ConfigMap
{{- if .Values.data }}
config: |
  {{ .Values.data }}
{{- end }}

# {{- removes trailing whitespace
# -}} removes leading whitespace
# Prevents unwanted blank lines
```

---

## 6. Debugging Templating Issues

### Scenario 1: Template Renders But YAML Is Invalid

```bash
# Error: "YAML parse error on line 45"

# Step 1: Render locally and examine
$ helm template myapp ./chart > manifest.yaml
$ cat manifest.yaml | grep -n "line 40-50"

# Step 2: Check indentation on line 45
sed -n '40,50p' manifest.yaml

# Step 3: Look for original template
$ grep -n "replicas" ./chart/templates/*.yaml

# Common causes:
1. {{ toYaml }} without nindent
2. Unquoted values with special chars
3. Loop indentation errors

# Fix: Re-check toYaml usage
# Before: {{ toYaml .Values.resources }}
# After:  {{ toYaml .Values.resources | nindent 2 }}
```

### Scenario 2: Template Variables Not Substituting

```bash
# Problem: Output contains {{ .Values.something }} literally

# Step 1: Check template file
$ cat ./chart/templates/deployment.yaml | grep "something"

# Step 2: Check if it's in quotes (template evaluation happens outside quotes)
# ❌ WRONG: value: "{{ .Values.something }}"  (quotes prevent eval)
# ✅ CORRECT: value: {{ .Values.something | quote }}

# Step 3: Check if variable exists
$ helm template myapp ./chart --debug | grep -A 5 "something"

# Step 4: Check values merge
$ helm values myapp ./chart -f values.yaml | grep "something"

# If "something" not in output, add it to values.yaml
```

### Scenario 3: Loop Not Iterating

```bash
# Problem: Loop body not appearing in output

# Example:
{{ range .Values.items }}
  - item: {{ . }}
{{ end }}

# Outputs nothing!

# Debug Step 1: Check if items exist
$ helm template myapp ./chart --debug
...
.Values = map[string]interface{}{
  "items": nil,    # ← items is nil!
  ...
}

# Debug Step 2: Check values file
$ cat values.yaml | grep -A 3 "items"
# items: not defined!

# Fix: Add items to values.yaml
items:
  - item1
  - item2
  - item3

# Re-render:
$ helm template myapp ./chart
outputs:
  - item: item1
  - item: item2
  - item: item3
```

### Scenario 4: Context Variables Undefined

```bash
# Error: value of key .Values.database.host is "<nil>"

# Template:
{{ .Values.database.host }}

# Debug:
$ helm template myapp ./chart --debug | grep -i database
# database not in values!

# Check values file:
$ cat values.yaml | grep database
# Not there!

# Fix 1: Add to values.yaml
database:
  host: localhost
  port: 5432

# Fix 2: Or use default in template
{{ .Values.database.host | default "localhost" }}

# Testing:
$ helm template myapp ./chart -f values.yaml
# Should show correct value
```

---

## 7. Interview Questions (15+ Scenario-Based)

### Level 1: Templating Basics

**Q1**: What's the difference between `{{ }}` and `{{- -}}`?

**Answer**:
- `{{ }}`: Evaluate template, keep whitespace
  ```
  Output:
    - item1     ← blank line before
    - item2
  ```

- `{{- }}`: Remove whitespace before/after (trim)
  ```
  {{- range .Values.items }}
    - {{ . }}
  {{- end }}
  
  Output:
  - item1       ← no blank line
  - item2
  ```

Usage: Use `{{- -}}` to control YAML indentation and prevent unwanted blank lines.

---

**Q2**: How do you pass values to templates?

**Answer**:
1. **Default values**: values.yaml in chart
   ```yaml
   replicaCount: 3
   ```

2. **File override**: -f flag
   ```bash
   helm install myapp ./chart -f values-prod.yaml
   ```

3. **CLI override**: --set flag
   ```bash
   helm install myapp ./chart \
     --set replicaCount=5 \
     --set image.tag=v1.2.3
   ```

4. **Merge order** (precedence):
   - Chart defaults (lowest)
   - -f values files (middle)
   - --set flags (highest)

---

### Level 2: Advanced Templating

**Q3**: Explain the difference between `include` and `template`.

**Answer**:
- `include`: Captures output (can pipe)
  ```yaml
  {{ include "mychart.labels" . | nindent 4 }}
  # Output piped through nindent
  ```

- `template`: Renders directly (no piping)
  ```yaml
  {{ template "mychart.labels" . }}
  # Output rendered as-is
  ```

**Practical difference**:
```yaml
# Using include (recommended)
metadata:
  labels:
    {{- include "mychart.labels" . | nindent 4 }}

# Output properly indented:
metadata:
  labels:
    app: myapp
    version: 1.0.0

# Using template (flat output)
metadata:
{{- template "mychart.labels" . }}

# Output not properly indented:
metadata:
app: myapp
version: 1.0.0
```

Use `include` + piping for better output control.

---

**Q4**: Design a reusable template helper for common labels.

**Answer**:
```yaml
# _helpers.tpl
{{- define "mychart.labels" -}}
app: {{ include "mychart.name" . }}
version: {{ .Chart.Version }}
release: {{ .Release.Name }}
managed-by: Helm
{{- end }}

{{- define "mychart.selectorLabels" -}}
app: {{ include "mychart.name" . }}
instance: {{ .Release.Name }}
{{- end }}

# Usage in templates/deployment.yaml
metadata:
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  
spec:
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}

# Output:
metadata:
  labels:
    app: my-app
    version: 1.0.0
    release: myapp
    managed-by: Helm
spec:
  selector:
    matchLabels:
      app: my-app
      instance: myapp
```

---

## 8. Advanced Insights (Template Patterns in CI/CD)

### Template Pattern 1: Progressive Delivery (Canary)

```yaml
# _helpers.tpl
{{- define "mychart.canaryWeight" -}}
  {{- if .Values.canary.enabled }}
  {{- .Values.canary.weight }}
  {{- else }}
  0
  {{- end }}
{{- end }}

# templates/virtualservice.yaml (Istio)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: {{ .Release.Name }}
spec:
  hosts:
  - {{ .Release.Name }}.example.com
  http:
  - match:
    - uri:
        prefix: "/"
    route:
    - destination:
        host: {{ .Release.Name }}-v1
      weight: {{ sub 100 (include "mychart.canaryWeight" .) }}
    - destination:
        host: {{ .Release.Name }}-v2
      weight: {{ include "mychart.canaryWeight" . }}

# values.yaml
canary:
  enabled: true
  weight: 10  # 10% traffic to v2

# Helm upgrade sequence:
helm upgrade myapp ./chart --set canary.weight=10  # 10% v2
helm upgrade myapp ./chart --set canary.weight=50  # 50% v2
helm upgrade myapp ./chart --set canary.weight=0 canary.enabled=false # 100% v2
```

### Template Pattern 2: Multi-Region Deployment

```yaml
# _helpers.tpl
{{- define "mychart.region" -}}
  {{- .Values.global.region | lower }}
{{- end }}

{{- define "mychart.registry" -}}
  {{- if eq (include "mychart.region" .) "us" }}
    us-docker.pkg.dev/myproject
  {{- else if eq (include "mychart.region" .) "eu" }}
    eu-docker.pkg.dev/myproject
  {{- else }}
    asia-docker.pkg.dev/myproject
  {{- end }}
{{- end }}

# templates/deployment.yaml
image: {{ include "mychart.registry" . }}/myapp:{{ .Values.image.tag }}

# Renders differently per region:
# US:   us-docker.pkg.dev/myproject/myapp:v1.0.0
# EU:   eu-docker.pkg.dev/myproject/myapp:v1.0.0
# Asia: asia-docker.pkg.dev/myproject/myapp:v1.0.0
```

### Template Pattern 3: Helm Dependency Templating

```yaml
# Chart.yaml
dependencies:
- name: prometheus
  version: "15.0.0"
  repository: "https://prometheus-community.github.io/helm-charts"
  condition: prometheus.enabled
  
- name: grafana
  version: "6.43.0"
  repository: "https://grafana.github.io/helm-charts"
  condition: grafana.enabled

# values.yaml
prometheus:
  enabled: true
  prometheusSpec:
    retention: 30d
    
grafana:
  enabled: true
  adminPassword: "secret"

# When rendered, Helm processes:
# 1. Parent chart templates
# 2. + prometheus dependency (with prometheus values)
# 3. + grafana dependency (with grafana values)
# 4. All merged into single deployment

# Advantage: Single helm install deploys entire stack!
$ helm install monitoring ./chart
# Deploys: prometheus + grafana + myapp in one command
```

### CI/CD Template Testing Strategy

```bash
# 1. Lint templates (static analysis)
$ helm lint ./chart
==> Linting chart/

1. values.yaml is valid YAML
2. Chart.yaml is valid
3. requirements.yaml is valid (if present)
4. Templates parse correctly

# 2. Template rendering (dry-run)
$ helm template myapp ./chart -f values.yaml > /tmp/rendered.yaml
# Generates pure manifests

# 3. Schema validation
$ kubeval /tmp/rendered.yaml
# Validates against Kubernetes OpenAPI schema

# 4. Policy validation (OPA/Kyverno)
$ conftest test /tmp/rendered.yaml -p policies/
# Custom policies: require resources limits, restrict privileged, etc

# 5. Diff comparison (before upgrade)
$ helm diff upgrade myapp ./chart -f values.yaml
# Shows what will change

# Complete pipeline:
$ helm lint ./chart && \
  helm template myapp ./chart -f values.yaml > /tmp/manifest.yaml && \
  kubeval /tmp/manifest.yaml && \
  conftest test /tmp/manifest.yaml && \
  helm upgrade --dry-run myapp ./chart -f values.yaml

# Only if all pass:
$ helm upgrade myapp ./chart -f values.yaml
```

---

## Conclusion

Helm templating mastery requires:
1. **Understanding Go templates**: `{{ }}` syntax, pipes, functions
2. **Managing complexity**: Conditionals, loops, context
3. **Debugging skills**: Render → validate → fix
4. **Best practices**: Helpers, indentation, whitespace control

Templates are where Helm's power lies! Master them and you can:
- Deploy any Kubernetes application
- Manage multiple environments seamlessly
- Create reusable charts for your organization
- Integrate with CI/CD pipelines

**Key takeaway**: Templates separate concerns (deployment logic) from configuration (values), making infrastructure-as-code maintainable at scale!
