# CI/CD - Continuous Integration & Deployment

## 1. Core Concept

**CI/CD** is the practice of **automating the pipeline from code to production**: continuous integration (frequent code merges with automated tests) and continuous deployment/delivery (automated testing, approval, and deployment).

**Value Proposition:**
- **Speed:** Deploy from commit to production in minutes
- **Reliability:** Automated tests catch bugs early
- **Safety:** Automated rollback on failure
- **Visibility:** Audit trail of what changed and why
- **Team Velocity:** Developers don't wait for manual deployment

---

## 2. Pipeline Architecture

### Typical CI/CD Pipeline

```
┌────────────┐    ┌───────────┐    ┌──────────┐    ┌────────┐    ┌──────────┐
│   Push     │───>│ Unit      │───>│ Build    │───>│ Test   │───>│ Deploy   │
│   Code     │    │ Tests     │    │ Docker   │    │ (E2E)  │    │ (Dev)    │
└────────────┘    │ Lint      │    │ image    │    │ Sec    │    └──────────┘
                  │ Format    │    │ Scan     │    │ Scan   │           │
                  └───────────┘    └──────────┘    └────────┘           │
                                                                          ↓
┌─────────────────────────────────────────────────────────────────────────────
│                                                                              
├──> ✅ Manual Approval (Staging)  
│     ↓
├──> Deploy to Staging
│     ↓
├──> Integration Tests
│     ↓
├──> Manual Approval (Production)
│     ↓
└──> Deploy to Production
```

### Real CI/CD Flow with Git

```
1. Developer: git push to feature branch
   ↓ (Webhook)

2. CI system detects push
   - Pulls code
   - Runs: linting, formatting, unit tests
   - Builds Docker image
   - Pushes to registry
   ↓

3. CI posts status on PR
   - ✅ All checks passed
   - 🔗 Link to build artifacts
   ↓

4. Team: Code review (human check)
   - Approves PR
   ↓

5. CI system: Merge to main triggered
   - Runs full test suite
   - Builds production image
   - Tags with commit hash
   ↓

6. CD system: Deployment
   - Option A: Auto-deploy to prod (GitOps)
   - Option B: Manual approval then deploy
   ↓

7. Health checks
   - Verify app is healthy
   - Automated rollback if unhealthy
```

---

## 3. Infrastructure-as-Code in CI/CD

### Pattern: Terraform Automation

```yaml
# .github/workflows/deploy-infrastructure.yaml
name: Terraform Deploy

on:
  push:
    branches: [main]
    paths: [terraform/**, .github/workflows/deploy-infrastructure.yaml]
  pull_request:
    branches: [main]
    paths: [terraform/**]

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    steps:
      # Authenticate to Google Cloud
      - uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      
      # Setup Terraform
      - uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      # Check code quality
      - name: Terraform Format
        run: terraform fmt -check -recursive terraform/
      
      - name: Terraform Validate
        working-directory: terraform
        run: terraform validate
      
      - name: TFLint
        uses: terraform-linters/setup-tflint@v3
        with:
          tflint_version: latest
      - run: |
          cd terraform
          tflint --recursive
          tflint -f json > tflint-report.json
      
      # Check for security issues
      - name: TFSec Security Scan
        uses: aquasecurity/tfsec-action@v1
        with:
          working_directory: terraform
      
      # Plan changes
      - name: Terraform Plan
        working-directory: terraform
        run: |
          terraform init -backend-config=backend.gcs
          terraform plan -out=tfplan -lock-timeout=5m
          terraform show tfplan > tfplan.txt
      
      # Post plan to PR comment
      - name: Comment on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('terraform/tfplan.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '```\n' + plan + '\n```'
            });
      
      # Apply after merge
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        working-directory: terraform
        run: terraform apply tfplan -lock-timeout=5m
      
      # Post-deployment validation
      - name: Validate Infrastructure
        run: |
          gcloud compute instances list --filter="environment:prod" --format=json
          # Verify infrastructure is as expected
      
      # Notify Slack
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1.24.0
        with:
          payload: |
            {
              "text": "Infrastructure deployed: ${{ job.status }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Terraform Deploy*\nStatus: ${{ job.status }}\nCommit: ${{ github.sha }}"
                  }
                }
              ]
            }
        if: always()
```

---

## 4. Application Deployment CI/CD

### Pattern: Kubernetes Deployment

```yaml
# .github/workflows/deploy-app.yaml
name: Build & Deploy Application

on:
  push:
    branches: [main]
    paths: [src/**, Dockerfile, k8s/**, .github/workflows/deploy-app.yaml]

env:
  REGISTRY: gcr.io
  IMAGE_NAME: ${{ secrets.GCP_PROJECT }}/myapp
  CLUSTER_NAME: prod-gke
  CLUSTER_REGION: us-central1

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
      # Code checkout
      - uses: actions/checkout@v3
      
      # Setup build environment
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      # Run tests
      - name: Unit Tests
        run: npm test
      
      - name: Build
        run: npm run build
      
      # Lint and format
      - name: Lint
        run: npm run lint
      
      - name: Format Check
        run: npm run format:check
      
      # Security checks
      - name: Dependency Check
        run: |
          npm audit --audit-level=high
          # Fails if high-severity vulnerabilities found
      
      - name: SAST Scan
        uses: github/super-linter@v4
      
      # Build Docker image
      - name: Build Docker Image
        run: |
          docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} .
          docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest .
      
      # Container image scanning
      - name: Scan Image for Vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Scan Results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
  
  push-image:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    steps:
      - uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      
      - uses: google-github-actions/setup-gcloud@v1
      
      - name: Configure Docker for GCR
        run: gcloud auth configure-docker
      
      - name: Push to Registry
        run: |
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
  
  deploy:
    needs: push-image
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      
      - uses: google-github-actions/setup-gcloud@v1
      
      # Connect to GKE cluster
      - name: Get GKE Credentials
        run: |
          gcloud container clusters get-credentials ${{ env.CLUSTER_NAME }} \
            --region ${{ env.CLUSTER_REGION }}
      
      # Update deployment
      - name: Update Deployment
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n production
      
      # Verify rollout
      - name: Wait for Rollout
        run: |
          kubectl rollout status deployment/myapp -n production --timeout=5m
      
      # Health checks
      - name: Health Check
        run: |
          kubectl get pods -l app=myapp -n production
          kubectl describe deployment myapp -n production
      
      # Automated rollback on failure
      - name: Rollback on Failure
        if: failure()
        run: |
          kubectl rollout undo deployment/myapp -n production
          kubectl rollout status deployment/myapp -n production
      
      # Notify
      - name: Deployment Notification
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "App deployed to production",
              "blocks": [{
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "✅ Deployment successful\nVersion: ${{ github.sha }}\nStatus: Healthy"
                }
              }]
            }
```

---

## 5. Common Issues & Solutions

### Issue 1: Long-running Pipeline

❌ **Problem:** Pipeline takes 45 minutes, blocks deployments

✅ **Solutions:**
```yaml
# Parallelize
jobs:
  test-unit:
    runs-on: ubuntu-latest
  test-integration:
    runs-on: ubuntu-latest
  test-e2e:
    runs-on: ubuntu-latest

# All run in parallel, total time = longest job

# Cache dependencies
- uses: actions/cache@v3
  with:
    path: node_modules
    key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}

# Only run necessary tests on PR, full suite on merge
- name: Run Tests
  run: |
    if [ "${{ github.event_name }}" == "pull_request" ]; then
      npm test -- --changed  # Only changed files
    else
      npm test               # Full suite
    fi
```

### Issue 2: Secret Management

❌ **Wrong:**
```yaml
- name: Deploy
  env:
    DB_PASSWORD: "prod-db-password-123"  # In plaintext!
  run: deploy
```

✅ **Right:**
```yaml
- name: Deploy
  env:
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
  run: deploy

# Secrets stored encrypted in GitHub
# Only decrypted during job execution
```

### Issue 3: Deployment Rollback

❌ **No rollback plan:**
```
Deploy fails
Application down
Manual rollback needed
```

✅ **Automated rollback:**
```yaml
# Verify health after deploy
- name: Health Check
  run: curl -f http://app/health

# If fails, automatically rollback
- name: Rollback
  if: failure()
  run: helm rollback myrelease
  # Or: kubectl rollout undo deployment/app
```

---

## 6. Interview Questions

### Q1: Pipeline Design

**Interviewer:** "Design CI/CD pipeline to safely deploy infrastructure changes to production."

**Good Answer:**

1. **Merge to main triggers pipeline**
2. **Terraform plan** - Show proposed changes
3. **Automatically post plan to PR** - Visibility
4. **Require manual approval** - 2-person rule
5. **Apply approved changes** - Automated deployment
6. **Run health checks** - Verify success
7. **Auto-rollback on failure** - Safety mechanism
8. **Notify team** - Transparency

### Q2: Handling Secrets

**Interviewer:** "How do you manage secrets (API keys, passwords) in CI/CD?"

**Good Answer:**
- Store in secret manager (GitHub Secrets, vault)
- Never commit to Git
- Inject as environment variables during execution
- Rotate regularly
- Audit access
- Use short-lived credentials when possible

---

## 7. GitOps Pattern

### Modern: GitOps (Git as Source of Truth)

```
┌─────────────┐
│ Git Repo    │  ← Source of Truth (infrastructure + app configs)
│ main branch │
└──────┬──────┘
       │ (webhook)
       ↓
┌─────────────────────┐
│ GitOps Controller   │  ← Runs in cluster
│ (Flux/ArgoCD)       │
└──────┬──────────────┘
       │ watches
       ↓
┌─────────────────────────┐
│ Kubernetes Cluster      │
│ (actual state)          │
└─────────────────────────┘

Workflow:
1. Developer: Update app version in Git
   - git commit infrastructure/app-deploy.yaml
   - image: myapp:v1.2.0 → v2.0.0

2. Git webhook: Triggers GitOps controller

3. Flux/ArgoCD:
   - Detects difference (Git vs Cluster)
   - Applies changes: kubectl apply
   - App automatically updated

4. If manual change made in cluster:
   - Flux detects drift
   - Reverts to Git state (source of truth)

Benefit: Git is single source of truth
```

---

## Key Takeaways

1. **Automate everything** - From build to production
2. **Fail fast** - Test early, catch issues immediately
3. **Deploy frequently** - Reduces blast radius of changes
4. **GitOps** - Git as source of truth enables safe deployments
5. **Rollback capability** - Always be able to revert
6. **Security scanning** - Catch vulnerabilities before production
7. **Notifications** - Visibility for governance/compliance
8. **Health checks** - Automated rollback on failure

---

## Production CI/CD Checklist

- [ ] **Automated testing** - Unit, integration, E2E
- [ ] **Code quality** - Linting, formatting, security scans
- [ ] **Container scanning** - Vulnerabilities in Docker images
- [ ] **Infrastructure-as-code testing** - Terraform validate, tflint, tfsec
- [ ] **Manual approval** - For production deployments
- [ ] **Automated rollout** - Rolling updates, canary deploys
- [ ] **Health checks** - Automated rollback on failure
- [ ] **Monitoring/logging** - See what's happening post-deploy
- [ ] **Secrets management** - No plaintext credentials
- [ ] **Audit trail** - Who changed what and when
- [ ] **Documentation** - How to operate pipelines
