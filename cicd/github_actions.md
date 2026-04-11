# GitHub Actions: Enterprise CI/CD Platform

## 1. Core Concept

GitHub Actions is GitHub's native CI/CD platform that:

1. **Triggers on events**: Push, PR, schedule, webhook, manual
2. **Runs jobs in workflows**: Sequential or parallel execution
3. **Executes steps**: Tasks like build, test, deploy
4. **Uses actions**: Reusable components (build, deploy, notify)
5. **Stores artifacts**: Build outputs, test reports, logs
6. **Manages secrets**: Encrypted credentials, tokens, keys

### Why GitHub Actions for Enterprise

```
Traditional CI/CD (Jenkins, GitLab CI):
├─ Separate infrastructure
├─ Authentication overhead
├─ Webhook configuration complexity
├─ Ops team to maintain

GitHub Actions:
├─ Native to repository
├─ Same permissions as Git
├─ Triggered automatically
├─ GitHub maintains infrastructure
└─ Zero setup!
```

### GitHub Actions Architecture

```
Event (push, PR, schedule)
    ↓
Webhook triggers GitHub
    ↓
Parse .github/workflows/
    ↓
Queue job in runner pool
    ↓
Runner executes:
  Step 1 → Step 2 → Step 3
    ↓
Store artifacts & logs
    ↓
Report status to Git
    ↓
Notify (Slack, email, etc)
```

---

## 2. Pipeline Architecture

### GitHub Actions Structure

```yaml
name: CI/CD Pipeline

# When to trigger
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # 2 AM daily
  workflow_dispatch:      # Manual trigger

# Environment variables (available to all jobs)
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: myapp

# Jobs (parallel by default)
jobs:
  # Job 1: Lint code
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run linter
        run: npm run lint

  # Job 2: Run tests
  test:
    runs-on: ubuntu-latest
    needs: lint  # Wait for lint job
    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        run: npm test

  # Job 3: Build & push
  build:
    runs-on: ubuntu-latest
    needs: test  # Wait for test job
    steps:
      - uses: actions/checkout@v3
      - name: Build image
        run: docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest .
      - name: Push to registry
        run: docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

  # Job 4: Deploy to staging
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    environment: staging  # Approval gate
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to staging
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest \
            -n staging

  # Job 5: Deploy to production
  deploy-prod:
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment: production  # Requires approval
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to production
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest \
            -n production
```

### Execution Flow

```
Workflow triggers
    ↓
All independent jobs start:
├─ Job: lint (5 mins)
├─ Job: build (platform check) 
└─ Job: security-scan (3 mins)
    ↓
When lint completes → test starts (depends: lint)
    ↓
When test completes → build starts (depends: test)
    ↓
When build completes → deploy-staging starts
    ↓
Human approval required (environment: production)
    ↓
deploy-prod starts on approval
    ↓
Workflow complete!

Total time: ~15-20 mins (vs sequential: 40+ mins)
```

### Runners: Self-Hosted vs GitHub-Hosted

```
GitHub-Hosted (Easiest):
├─ ubuntu-latest
├─ windows-latest
├─ macos-latest
├─ Pre-installed tools
└─ Free 2,000 mins/month (private repos)

Self-Hosted (For enterprise):
├─ On-premise servers
├─ Full control
├─ No rate limits
├─ Custom tools/hardware
└─ Your ops team manages

Hybrid approach:
├─ Linting & testing → GitHub-hosted (fast, free!)
├─ Deployment → Self-hosted (security, control)
└─ Best of both!
```

---

## 3. Real-World CI/CD Workflows

### Workflow 1: Node.js App with PR Reviews

```yaml
name: Node.js CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test-and-lint:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
      
      - name: Comment PR with coverage
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const coverage = JSON.parse(fs.readFileSync('./coverage/coverage-final.json', 'utf8'));
            const lines = (coverage.total.lines.pct).toFixed(2);
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `📊 Coverage: ${lines}%`
            });

  build-and-push:
    runs-on: ubuntu-latest
    needs: test-and-lint
    if: github.ref == 'refs/heads/main'
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Docker buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix={{branch}}-
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

# Result: 
# - Tests run on multiple Node versions
# - Coverage reported on PRs
# - Docker image built & pushed on main branch
```

### Workflow 2: Infrastructure Testing & Deployment

```yaml
name: Infrastructure Pipeline

on:
  push:
    paths:
      - 'terraform/**'
      - 'ansible/**'
  pull_request:
    paths:
      - 'terraform/**'
      - 'ansible/**'

jobs:
  terraform-plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Terraform format check
        run: terraform fmt -check -recursive terraform/
      
      - name: Terraform validate
        run: terraform validate terraform/
      
      - name: Terraform plan
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          cd terraform/
          terraform init
          terraform plan -out=tfplan
      
      - name: Upload plan
        uses: actions/upload-artifact@v3
        with:
          name: tfplan
          path: terraform/tfplan

  ansible-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install ansible-lint
        run: pip install ansible-lint
      
      - name: Lint playbooks
        run: ansible-lint ansible/

  terraform-apply:
    runs-on: ubuntu-latest
    needs: terraform-plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Download plan
        uses: actions/download-artifact@v3
        with:
          name: tfplan
          path: terraform/
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
      
      - name: Terraform apply
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          cd terraform/
          terraform init
          terraform apply -auto-approve tfplan
      
      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "Terraform applied",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Terraform Apply Complete*\n${{ job.status }}"
                  }
                }
              ]
            }
```

### Workflow 3: Scheduled Maintenance & Health Checks

```yaml
name: Scheduled Operations

on:
  schedule:
    - cron: '0 2 * * *'        # Daily at 2 AM
    - cron: '0 0 * * 0'        # Weekly Sunday
  workflow_dispatch:            # Manual trigger

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Alert if high vulnerabilities
        run: |
          if grep '"CRITICAL"' trivy-results.sarif; then
            echo "🚨 CRITICAL VULNERABILITIES FOUND"
            exit 1
          fi

  dependency-updates:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Check for outdated dependencies
        run: npm outdated || true
      
      - name: Create PR for updates
        uses: peter-evans/create-pull-request@v5
        with:
          title: 'chore: update dependencies'
          body: 'Automated dependency update'
          branch: 'auto/dependency-update'

  health-check:
    runs-on: ubuntu-latest
    steps:
      - name: Health check production
        run: |
          curl -f https://api.example.com/health || exit 1
      
      - name: Health check staging
        run: |
          curl -f https://staging-api.example.com/health || exit 1
      
      - name: Alert if unhealthy
        if: failure()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "🚨 Health check failed",
              "color": "danger"
            }

  backup-verification:
    runs-on: ubuntu-latest
    steps:
      - name: Verify recent backups
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          # Check last backup age
          LAST_BACKUP=$(aws s3 ls s3://backups/ --recursive | sort | tail -n 1 | awk '{print $1" "$2}')
          BACKUP_TIME=$(date -d "$LAST_BACKUP" +%s)
          CURRENT_TIME=$(date +%s)
          DIFF=$((($CURRENT_TIME - $BACKUP_TIME) / 3600))
          
          if [ $DIFF -gt 24 ]; then
            echo "⚠️ Backup is $DIFF hours old!"
            exit 1
          fi
```

---

## 4. Common Mistakes

### Mistake 1: No Status Checks on PRs

```yaml
# ❌ WRONG
- name: Deploy on any branch
  run: deploy.sh
# Code deploys even on feature branches!

# ✅ CORRECT
- name: Deploy only on main
  if: github.ref == 'refs/heads/main'
  run: deploy.sh

# Or require status checks:
# Repository Settings → Branches → Require status checks
# Select: test, lint, build jobs
# PRs can't merge unless all pass!
```

### Mistake 2: No Timeout Protection

```yaml
# ❌ WRONG
- name: Long running test
  run: npm test
# Hangs forever if test crashes = workflow stuck

# ✅ CORRECT
- name: Long running test
  run: npm test
  timeout-minutes: 30  # Kill if exceeds 30 mins

# Or for job:
jobs:
  test:
    timeout-minutes: 30  # Kill entire job
    runs-on: ubuntu-latest
```

### Mistake 3: Exposing Secrets in Logs

```yaml
# ❌ WRONG
- name: Login to Docker
  run: |
    echo "Password: ${{ secrets.DOCKER_PASSWORD }}"
    docker login -u ${{ secrets.DOCKER_USER }} \
      -p ${{ secrets.DOCKER_PASSWORD }}
# Password exposed in logs!

# ✅ CORRECT
- name: Login to Docker
  run: |
    docker login -u ${{ secrets.DOCKER_USER }} \
      -p ${{ secrets.DOCKER_PASSWORD }}
  # GitHub automatically masks secrets in output

# Or use dedicated action:
- uses: docker/login-action@v2
  with:
    username: ${{ secrets.DOCKER_USER }}
    password: ${{ secrets.DOCKER_PASSWORD }}
```

### Mistake 4: Expensive Matrix Without Filtering

```yaml
# ❌ WRONG
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node-version: [16.x, 18.x, 20.x]
    # 3 × 3 = 9 jobs!
    # Wastes CI minutes

# ✅ CORRECT
strategy:
  matrix:
    include:
      - os: ubuntu-latest
        node-version: 16.x  # Old LTS
      - os: ubuntu-latest
        node-version: 20.x  # Latest LTS
      - os: windows-latest
        node-version: 20.x  # Test on Windows
      - os: macos-latest
        node-version: 20.x  # Test on macOS
    # 4 jobs instead of 9
    # Still good coverage!
```

### Mistake 5: Not Cleaning Up Old Artifacts

```yaml
# ❌ WRONG
- uses: actions/upload-artifact@v3
  with:
    name: build
    path: dist/
# Artifacts accumulate forever = storage costs!

# ✅ CORRECT
- uses: actions/upload-artifact@v3
  with:
    name: build
    path: dist/
    retention-days: 7  # Auto-delete after 7 days

# Or manually clean:
- uses: geekyeggo/delete-artifact@v2
  with:
    name: build
    failOnError: false
```

---

## 5. Debugging Pipelines

### Problem: Workflow Stuck in Queued State

```bash
# Signs:
# - "Queued" status never changes to "Running"
# - No logs appear

# Debug:
1. Check runner availability
   Settings → Actions → Runners → See available runners

2. Check if runner offline
   - Self-hosted runner: Restart runner application
   - GitHub-hosted: Usually available

3. Check runner labels match
   runs-on: [ubuntu-latest]  # ← GitHub label
   vs runs-on: self-hosted    # ← Missing 'self-hosted' label?

# Fix:
runs-on: self-hosted
# or
runs-on: [self-hosted, ubuntu]
```

### Problem: "Permission denied" When Accessing Secrets

```yaml
# ❌ WRONG
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy
        run: kubectl apply --kubeconfig=${{ secrets.KUBECONFIG }}
        # Error: Permission denied!

# Reasons:
# 1. Secret doesn't exist
# 2. Secret name typo
# 3. Secret not accessible from this job
# 4. Workflow not in branch where secret exists

# Debug:
# 1. Check Settings → Secrets and variables → Actions
# 2. Verify secret exists
# 3. Verify workflow name correct (case-sensitive!)
# 4. For org secrets: Settings → Secrets and variables → Actions

# Fix:
- name: Deploy with kubeconfig
  env:
    KUBECONFIG_CONTENT: ${{ secrets.KUBECONFIG }}
  run: |
    echo "$KUBECONFIG_CONTENT" > ~/.kube/config
    kubectl apply -f deployment.yaml
```

### Problem: Git Checkout Fails

```bash
# Error: "fatal: repository not found"

# Debug:
1. Check if pushing to right repository
   $ git remote -v
   
2. Check if SSH key configured
   $ ssh -T git@github.com
   
3. Check workflow permissions
   Settings → Actions → General → Workflow permissions
   Should be: "Read and write permissions"

# Fix in workflow:
jobs:
  deploy:
    permissions:
      contents: write      # Need to push
      pull-requests: write # Need to create PRs
    steps:
      - uses: actions/checkout@v3
        with:
          token: ${{ secrets.GITHUB_TOKEN }}  # Explicit token
```

---

## 6. Interview Questions (Scenario-Based)

### **Q1**: Design a zero-downtime GitHub Actions deployment pipeline for a Node.js microservice.

**Answer**:
```yaml
name: Zero-Downtime Deployment

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci && npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v3
      - uses: docker/setup-buildx-action@v2
      - uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v4
        with:
          images: ghcr.io/${{ github.repository }}
      - uses: docker/build-push-action@v4
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}

  deploy-blue-green:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v3
      
      # Determine which is active (blue or green)
      - name: Get active slot
        run: |
          ACTIVE=$(curl -s https://api.example.com/deployment/active)
          INACTIVE=$([ "$ACTIVE" = "blue" ] && echo "green" || echo "blue")
          echo "ACTIVE=$ACTIVE" >> $GITHUB_ENV
          echo "INACTIVE=$INACTIVE" >> $GITHUB_ENV
      
      # Deploy to inactive slot (green if blue is active)
      - name: Deploy to ${{ env.INACTIVE }}
        run: |
          kubectl set image deployment/app-${{ env.INACTIVE }} \
            app=${{ needs.build.outputs.image }} \
            -n production
      
      # Wait for readiness
      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/app-${{ env.INACTIVE }} \
            -n production \
            --timeout=5m
      
      # Health check
      - name: Health check
        run: |
          for i in {1..30}; do
            if curl -f http://app-${{ env.INACTIVE }}:8080/health; then
              echo "✅ Health check passed"
              exit 0
            fi
            sleep 10
          done
          exit 1
      
      # Switch traffic
      - name: Switch traffic
        run: |
          curl -X POST https://api.example.com/deployment/switch \
            -d "target=${{ env.INACTIVE }}" \
            -H "Authorization: Bearer ${{ secrets.API_TOKEN }}"
      
      # Verify old version still running (for instant rollback)
      - name: Verify previous version ready
        run: |
          curl -f http://app-${{ env.ACTIVE }}:8080/health
      
      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "Deployment: ${{ job.status }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "Switched from ${{ env.ACTIVE }} to ${{ env.INACTIVE }}"
                  }
                }
              ]
            }
```

---

## 7. Deployment Strategies

### Blue-Green Deployment

```yaml
# Same setup as above - traffic switches instantly
# Pros:
#   - Instant rollback (just switch back)
#   - No downtime
#   - Easy to test before switching
# 
# Cons:
#   - Need 2x resources (2 full environments)
#   - Database migrations tricky
```

### Canary Deployment

```yaml
jobs:
  deploy-canary:
    environment: production
    steps:
      - name: Deploy to 5% of servers
        run: |
          kubectl set image deployment/app \
            app=new-image \
            -n production \
            --replicas=1  # 1 out of 20 servers

      - name: Monitor metrics
        run: |
          for i in {1..60}; do
            ERROR_RATE=$(curl https://metrics.example.com/error_rate?pod=canary)
            if (( $(echo "$ERROR_RATE > 5" | bc -l) )); then
              echo "❌ Error rate too high: $ERROR_RATE%"
              exit 1
            fi
            sleep 10
          done
      
      - name: Gradual rollout if healthy
        run: |
          kubectl set replicas deployment/app \
            --replicas=20 \
            -n production
```

---

## 8. Advanced Insights

### Reusable Workflows

```yaml
# .github/workflows/node-test.yml
name: Node Test Workflow

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
      npm-script:
        required: true
        type: string

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ inputs.node-version }}
      - run: npm ci
      - run: npm run ${{ inputs.npm-script }}

# main.yml (uses the reusable workflow)
on: [push]

jobs:
  call-workflow:
    uses: ./.github/workflows/node-test.yml
    with:
      node-version: '20.x'
      npm-script: 'test'
```

### Composite Actions

```yaml
# .github/actions/deploy-to-k8s/action.yml
name: Deploy to Kubernetes
description: Deploy image to K8s cluster

inputs:
  image:
    description: Container image
    required: true
  environment:
    description: Target environment
    required: true
  kubeconfig:
    description: Kubeconfig content
    required: true

runs:
  using: composite
  steps:
    - name: Setup kubeconfig
      run: |
        mkdir -p ~/.kube
        echo "${{ inputs.kubeconfig }}" > ~/.kube/config
      shell: bash
    
    - name: Deploy
      run: |
        kubectl set image deployment/app \
          app=${{ inputs.image }} \
          -n ${{ inputs.environment }}
      shell: bash

# Usage in workflow:
- uses: ./.github/actions/deploy-to-k8s
  with:
    image: ghcr.io/app:latest
    environment: production
    kubeconfig: ${{ secrets.KUBECONFIG }}
```

---

## Conclusion

GitHub Actions provides:

✅ **Native integration** with repositories  
✅ **Easy setup** (no external infrastructure)  
✅ **Powerful features** (workflows, matrix builds, environments)  
✅ **Security** (secret management, OIDC, approval gates)  
✅ **Scalability** (self-hosted runners for enterprise)  

Master GitHub Actions and you can:
- Deploy safely with approval gates
- Test across multiple platforms
- Integrate with any cloud platform
- Automate all development workflows
- Maintain audit trails for compliance
