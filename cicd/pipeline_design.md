# CI/CD Pipeline Design: Architecture, Patterns, and Best Practices

## 1. Core Concept

A production CI/CD pipeline automates:

1. **Source Control**: Git commits trigger workflows
2. **Build**: Compile code, run build tools
3. **Test**: Unit, integration, security tests
4. **Quality Gates**: Coverage, linting, scanning
5. **Deployment**: Move to staging/production
6. **Verification**: Health checks, smoke tests
7. **Rollback**: Revert if issues detected
8. **Notification**: Alert teams of status

### Why Pipeline Design Matters

```
Without CI/CD (Manual):
├─ Developer runs tests locally (maybe)
├─ Commits to git
├─ Someone manually deploys
├─ Deployment fails?
├─ Manual rollback
├─ Slow: 1-2 hours per deployment

With Well-Designed CI/CD:
├─ Commit to git
├─ Automated tests run
├─ Automated quality checks
├─ Staging deployment (auto)
├─ Approval gate (human)
├─ Production deployment (auto)
├─ Automated verification
├─ Auto-rollback if issues
└─ Fast: 5-10 mins per deployment!
```

---

## 2. Pipeline Architecture

### Modern Pipeline Stages

```
┌────────────────────────────────────────────────────────┐
│ TRIGGER (git push, PR, schedule)                       │
└────────────────────────┬───────────────────────────────┘
                         ↓
┌────────────────────────────────────────────────────────┐
│ COMMIT STAGE (Validate code quality)                   │
├─ Lint & format check                                  │
├─ Dependency audit                                      │
├─ SAST (Static analysis)                               │
└─────────────────────────┬──────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────┐
│ BUILD STAGE (Create deployable artifact)               │
├─ Compile code                                          │
├─ Run unit tests (40% of tests)                         │
├─ Generate coverage report                              │
├─ Build container image                                 │
├─ Push to registry                                      │
├─ Generate SBOM (Software Bill of Materials)            │
└─────────────────────────┬──────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────┐
│ TEST STAGE (Integration & security testing)            │
├─ Integration tests (30% of tests)                      │
├─ API tests                                             │
├─ Container image scan (Trivy, Snyk)                    │
├─ DAST (Dynamic security scan)                          │
└─────────────────────────┬──────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────┐
│ QUALITY GATES (Block if thresholds not met)            │
├─ Coverage > 80%                                        │
├─ No high/critical vulnerabilities                      │
├─ Build time < 10 mins                                  │
└─────────────────────────┬──────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────┐
│ STAGING DEPLOY (Deploy to staging environment)         │
├─ Blue-green deployment logic                           │
├─ Dynamic replica sizing                               │
├─ E2E tests (30% of tests)                              │
└─────────────────────────┬──────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────┐
│ APPROVAL GATE (Human review & approval)                │
├─ Review deployment changes                             │
├─ Check staging test results                            │
├─ Approval timeout (e.g., 24 hours)                     │
└─────────────────────────┬──────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────┐
│ PRODUCTION DEPLOY                                      │
├─ Canary deployment (5% of traffic)                     │
├─ Monitor metrics (30 seconds)                          │
├─ Gradual rollout (10% → 50% → 100%)                   │
├─ Smoke tests                                           │
├─ Verify all replicas healthy                           │
└─────────────────────────┬──────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────┐
│ MONITORING & ROLLBACK                                  │
├─ Track error rates vs baseline                         │
├─ Watch response times                                  │
├─ Auto-rollback if degradation detected                 │
├─ Alert teams                                           │
└────────────────────────────────────────────────────────┘
```

### Test Pyramid (Distribution)

```
            /\
           /  \          END-TO-END (10%)
          /────\        ├── Full workflow tests
         /      \       ├── Slow (1-5 mins each)
        /        \      └── ~5-10 tests
       /──────────\
      /            \    INTEGRATION (30%)
     /              \   ├── Component interaction
    /────────────────\  ├── Database, APIs
   /                  \ └── ~50-100 tests
  /                    \
 /──────────────────────\ UNIT (60%)
/________________________\ ├── Individual functions
                          ├── Fast (<100ms each)
                          └── ~1000+ tests

Benefits:
├─ Fast feedback (quick unit tests)
├─ Broad coverage (many unit tests)
├─ Few expensive tests (E2E)
└─ Realistic scenarios (integration)
```

---

## 3. Real-World CI/CD Workflows

### Workflow 1: Enterprise Microservices Pipeline

```yaml
# Full production pipeline with all stages
name: Production Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: myapp

jobs:
  # Stage 1: Commit validation
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # Full history for diff
      
      - name: Lint code
        run: npm run lint
      
      - name: Audit dependencies
        run: npm audit
      
      - name: Check commit messages
        run: |
          # Verify conventional commits
          npx commitlint --from=origin/main
      
      - name: Check for secrets
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: origin/main
          extra_args: --debug

  # Stage 2: Build
  build:
    needs: validate
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.meta.outputs.tags }}
      digest: ${{ steps.build.outputs.digest }}
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '20.x'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      # Unit tests (most of testing)
      - name: Run unit tests
        run: npm run test:unit -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
      
      - name: Build application
        run: npm run build
      
      - name: Setup Docker buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha
      
      - id: build
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # Stage 3: Test
  test:
    needs: build
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '20.x'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Integration tests
        env:
          DATABASE_URL: postgres://postgres:testpass@localhost:5432/test
        run: npm run test:integration
      
      - name: API tests
        run: npm run test:api

  # Stage 4: Security scanning
  security-scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Pull image for scanning
        run: docker pull ${{ needs.build.outputs.image }}
      
      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ needs.build.outputs.image }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Fail if vulnerabilities
        run: |
          if [ -s trivy-results.sarif ]; then
            exit 1
          fi

  # Quality Gate (block if thresholds not met)
  quality-gate:
    needs: [build, test, security-scan]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Check build status
        if: needs.build.result != 'success'
        run: exit 1
      
      - name: Check test status
        if: needs.test.result != 'success'
        run: exit 1
      
      - name: Check security
        if: needs.security-scan.result != 'success'
        run: exit 1
      
      - name: All gates passed
        run: echo "✅ Ready for deployment"

  # Stage 5: Deploy to staging
  deploy-staging:
    needs: [quality-gate, build]
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure kubectl
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG_STAGING }}" | base64 -d > ~/.kube/config
      
      - name: Deploy to staging
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ needs.build.outputs.image }} \
            -n staging
      
      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/myapp \
            -n staging \
            --timeout=5m
      
      - name: E2E tests on staging
        run: npm run test:e2e
        env:
          BASE_URL: https://staging.example.com

  # Stage 6: Approval gate
  approval:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production-approval
    
    steps:
      - name: Waiting for approval
        run: echo "⏳ Awaiting approval for production deployment"

  # Stage 7: Deploy to production
  deploy-prod:
    needs: [approval, build]
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure kubectl
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG_PROD }}" | base64 -d > ~/.kube/config
      
      # Blue-green deployment
      - name: Determine inactive slot
        run: |
          ACTIVE=$(kubectl get svc myapp-lb -o jsonpath='{.spec.selector.slot}')
          INACTIVE=$([ "$ACTIVE" = "blue" ] && echo "green" || echo "blue")
          echo "INACTIVE=$INACTIVE" >> $GITHUB_ENV
      
      - name: Deploy to ${{ env.INACTIVE }}
        run: |
          kubectl set image deployment/myapp-${{ env.INACTIVE }} \
            myapp=${{ needs.build.outputs.image }} \
            -n production
      
      - name: Wait for readiness
        run: |
          kubectl wait --for=condition=ready pod \
            -l app=myapp,slot=${{ env.INACTIVE }} \
            -n production \
            --timeout=5m
      
      - name: Health checks
        run: |
          for i in {1..30}; do
            if curl -f https://prod.example.com/health; then
              echo "✅ Health check passed"
              exit 0
            fi
            sleep 10
          done
          exit 1
      
      - name: Switch traffic
        run: |
          kubectl patch svc myapp-lb \
            -p '{"spec":{"selector":{"slot":"${{ env.INACTIVE }}"}}}'
      
      - name: Monitor for 5 mins
        run: |
          for i in {1..30}; do
            ERROR_RATE=$(curl -s https://metrics.example.com/error_rate)
            if (( $(echo "$ERROR_RATE > 1" | bc -l) )); then
              echo "❌ Error rate spiked: $ERROR_RATE%"
              exit 1
            fi
            sleep 10
          done

  # Notifications
  notify:
    needs: [deploy-prod]
    runs-on: ubuntu-latest
    if: always()
    
    steps:
      - name: Notify Slack
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
                    "text": "*Deployment ${{ job.status | upper }}*\n${{ github.event.head_commit.message }}"
                  }
                }
              ]
            }
        if: always()
```

### Workflow 2: Infrastructure as Code Pipeline

```yaml
name: IaC Pipeline

on:
  pull_request:
    paths:
      - 'terraform/**'
      - 'helmchart/**'
  push:
    branches: [main]
    paths:
      - 'terraform/**'
      - 'helmchart/**'

jobs:
  terraform-validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Terraform fmt
        run: terraform fmt -check -recursive .
      
      - name: Terraform init
        run: terraform init -backend=false
      
      - name: Terraform validate
        run: terraform validate
      
      - name: TFLint
        uses: terraform-linters/setup-tflint@v3
        with:
          tflint_version: latest
      
      - name: Run TFLint
        run: |
          tflint --init
          tflint --format compact

  helm-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: azure/setup-helm@v3
      
      - name: Helm lint
        run: helm lint helmchart/
      
      - name: Helm template
        run: helm template myapp helmchart/ > rendered.yaml
      
      - name: Validate YAML
        run: kubectl apply -f rendered.yaml --dry-run=client

  plan:
    needs: [terraform-validate, helm-lint]
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: hashicorp/setup-terraform@v2
      
      - name: Terraform plan
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          terraform plan -out=tfplan
      
      - name: Generate plan summary
        run: terraform show -json tfplan > plan.json
      
      - name: Comment on PR with plan
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const plan = JSON.parse(fs.readFileSync('plan.json'));
            const changes = plan.resource_changes.length;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `📋 Terraform Plan:\n${changes} resources to change`
            });

  apply:
    needs: [plan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: infrastructure
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: hashicorp/setup-terraform@v2
      
      - name: Terraform apply
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: terraform apply -auto-approve tfplan
      
      - name: Store outputs
        run: terraform output -json > outputs.json
      
      - name: Commit outputs
        run: |
          git config user.email "ci@example.com"
          git config user.name "CI"
          git add outputs.json
          git commit -m "Update Terraform outputs"
          git push
```

---

## 4. Common Mistakes

### Mistake 1: All Tests Run Sequentially

```yaml
# ❌ WRONG
jobs:
  test1:
    runs-on: ubuntu-latest
    steps:
      - run: npm test -- --testNamePattern="Feature1"
  
  test2:
    needs: test1  # ← Wait for test1!
    steps:
      - run: npm test -- --testNamePattern="Feature2"
  
  test3:
    needs: test2  # ← Wait for test2!
    steps:
      - run: npm test -- --testNamePattern="Feature3"

# Total time: 30 mins (10+10+10)

# ✅ CORRECT (parallel)
jobs:
  test1:
    runs-on: ubuntu-latest
    steps:
      - run: npm test -- --testNamePattern="Feature1"
  
  test2:
    runs-on: ubuntu-latest
    # No dependency!
    steps:
      - run: npm test -- --testNamePattern="Feature2"
  
  test3:
    runs-on: ubuntu-latest
    steps:
      - run: npm test -- --testNamePattern="Feature3"

# Total time: ~10 mins (all parallel!)
```

### Mistake 2: No Approval Gates for Production

```yaml
# ❌ WRONG
deploy-prod:
  runs-on: ubuntu-latest
  if: github.ref == 'refs/heads/main'
  steps:
    - run: kubectl apply -f deployment.yaml
# Deploys automatically without review!

# ✅ CORRECT
deploy-prod:
  needs: deploy-staging
  runs-on: ubuntu-latest
  environment: production  # ← Requires approval!
  if: github.ref == 'refs/heads/main'
  steps:
    - run: kubectl apply -f deployment.yaml

# Repository settings require approval before this step
```

### Mistake 3: No Timeout on Long Operations

```yaml
# ❌ WRONG
- name: Run tests
  run: npm test
# Hangs forever if tests crash

# ✅ CORRECT
jobs:
  test:
    timeout-minutes: 30  # Kill job if exceeds 30 mins
    steps:
      - run: npm test
        timeout-minutes: 15  # Kill step if exceeds 15 mins
```

---

## 5. Debugging Pipeline Issues

### Pipeline Slow (5+ minutes per stage)

```bash
# Debug steps:

1. Check parallelization
   - Are jobs running sequentially when they could be parallel?
   - Use: needs: only when truly necessary

2. Check cache hits
   - npm cache: --cache, actions/setup-node cache
   - Docker buildx cache: type=gha

3. Check runner availability
   - GitHub-hosted: Usually fast
   - Self-hosted: May be overloaded

4. Check step duration
   - npm install slow? Use npm ci
   - Docker build slow? Multi-stage builds, caching
   - Tests slow? Parallel test runs, split into chunks

# Fix: Parallel tests across multiple jobs
strategy:
  matrix:
    test-suite: [unit, integration, e2e]
jobs:
  test:
    strategy:
      matrix:
        test-suite: [unit, integration, e2e]
    steps:
      - run: npm run test:${{ matrix.test-suite }}
```

---

## 6. Interview Questions

### **Q1**: Design a CI/CD pipeline for a 50-engineer team deploying 20 microservices daily to production.

**Answer**:
```yaml
# Architecture:
#
# 1. Monorepo with multiple services
# 2. Service discovery via git paths
# 3. Shared validation stage
# 4. Independent build/test per service
# 5. Staged deployment (dev → staging → prod)
# 6. Cross-service integration tests
# 7. Rollback per service

name: Multi-Service Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # Discover changed services
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.detect.outputs.services }}
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - id: detect
        run: |
          CHANGED=$(git diff --name-only ${{ github.event.before }} | \
            grep -oE '^services/[^/]+' | sort -u)
          echo "services=$(echo $CHANGED | jq -R 'splits(" ") | select(. != "")')" >> $GITHUB_OUTPUT

  # Build all changed services
  build:
    needs: detect-changes
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.services) }}
    steps:
      - uses: actions/checkout@v3
      - name: Build ${{ matrix.service }}
        run: |
          cd services/${{ matrix.service }}
          docker build -t myregistry/${{ matrix.service }}:${{ github.sha }} .
          docker push myregistry/${{ matrix.service }}:${{ github.sha }}

  # Test each service independently
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Integration tests across services
        run: |
          docker-compose -f docker-compose.test.yml up --abort-on-container-exit

  # Deploy to staging
  deploy-staging:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Deploy all services
        run: helm upgrade --install myapp ./helm -f staging-values.yaml

  # Approval for prod
  approval:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: prod-approval
    steps:
      - run: echo "Ready for production"

  # Canary deploy to prod
  deploy-prod-canary:
    needs: approval
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to 5% canary
        run: ./scripts/deploy-canary.sh

  # Monitor and gradual rollout
  deploy-prod-full:
    needs: deploy-prod-canary
    runs-on: ubuntu-latest
    steps:
      - name: Monitor canary (5 mins)
        run: ./scripts/monitor-metrics.sh
      
      - name: Gradual rollout
        run: ./scripts/rollout-gradual.sh
```

---

## 7. Deployment Strategies

### Progressive Delivery

```
Day 1 (Canary): 5% traffic
Day 2 (Canary+): 25% traffic
Day 3 (Beta): 50% traffic
Day 4 (Full): 100% traffic

Benefits: Catch issues early, gradual rollback
```

---

## 8. Advanced Insights

### Cost Optimization

```yaml
# Reduce CI minutes:
# 1. Cache everything (npm, maven, docker)
# 2. Parallel jobs
# 3. Skip jobs on certain paths
# 4. Use self-hosted for expensive operations
# 5. Matrix builds on fewer platforms

# Reduce cloud costs:
# 1. Auto-cleanup old containers
# 2. Staging uses fewer replicas
# 3. Scheduled cleanup jobs
# 4. Reserved instances for runners
```

---

## Conclusion

A well-designed CI/CD pipeline:

✅ **Fast**: 5-15 minutes total  
✅ **Safe**: Multiple quality gates  
✅ **Reliable**: Automated recovery  
✅ **Scalable**: Hundreds of deployments/day  
✅ **Auditable**: Full change history  

Master pipeline design and you enable:
- ✅ Rapid, safe deployments
- ✅ Team confidence in code quality
- ✅ Quick incident recovery
- ✅ Compliance audit trails
- ✅ Cost optimization
