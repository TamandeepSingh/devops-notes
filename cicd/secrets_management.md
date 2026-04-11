# Secrets Management in CI/CD: Security, Rotation, and Production Practices

## 1. Core Concept

**Secrets** are sensitive credentials that enable access to infrastructure:

1. **Database passwords**: Connect to prod database
2. **API tokens**: Deploy to cloud, notify services
3. **SSH keys**: Deploy to servers
4. **Cloud credentials**: AWS/GCP/Azure credentials
5. **TLS certificates**: Enable HTTPS
6. **OAuth tokens**: Access third-party services
7. **Encryption keys**: Decrypt sensitive data

### Why Secrets Management Critical

```
Security failures:
├─ Secrets committed to Git
│  └─ Anyone with repo access sees passwords!
│
├─ Secrets in environment variables
│  └─ Visible in process listings, logs, containers!
│
├─ Hardcoded in config files
│  └─ Copied in PR diffs, vulnerability scans!
│
├─ Static secrets (never rotated)
│  └─ Compromise = forever access!
│
└─ Unencrypted secrets
   └─ Breach exposes everything!

With proper management:
├─ ✅ Encrypted at rest
├─ ✅ Encrypted in transit
├─ ✅ Rotated regularly
├─ ✅ Audit trail of access
├─ ✅ Limited to specific jobs
└─ ✅ Automatically revoked
```

---

## 2. Pipeline Architecture for Secrets

### Secret Flow

```
Developer needs API token
    ↓
Requests access (via ops ticket)
    ↓
Ops reviews access (policy check)
    ├─ What service needs it?
    ├─ How long needed?
    ├─ What permissions?
    └─ What's the risk?
    ↓
Ops creates token with expiration
    ↓
Stores in secrets manager (encrypted)
    ├─ GitHub Secrets
    ├─ HashiCorp Vault
    ├─ AWS Secrets Manager
    └─ Azure Key Vault
    ↓
CI/CD retrieves at runtime
    ├─ Decrypts
    ├─ Injects into job environment
    └─ Clears after job completes
    ↓
Secrets never visible in:
├─ Git commits
├─ Logs
├─ Process listings
└─ Container images
```

### Access Control

```
Developer (principal)
    ↓
Requests secret (database password)
    ↓
Policy engine checks:
├─ Does developer have permission?
├─ Is secret eligible for this role?
├─ Has MFA been used?
└─ Is access from approved location?
    ↓
If approved:
├─ Log access (audit trail)
├─ Return secret
├─ Start expiration timer
└─ Monitor for suspicious usage
    ↓
If denied:
├─ Log denial
├─ Alert security team
└─ Block access
```

---

## 3. Real-World Secrets Management Workflows

### Workflow 1: GitHub Secrets (Simple)

```yaml
# GitHub Actions workflow using secrets
name: Deploy with GitHub Secrets

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v3
      
      # Secret automatically injected by GitHub
      # Not visible in logs (GitHub masks it)
      - name: Deploy to production
        env:
          DATABASE_PASSWORD: ${{ secrets.DATABASE_PASSWORD }}
          AWS_ACCESS_KEY: ${{ secrets.AWS_ACCESS_KEY }}
          DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
        run: |
          # GitHub automatically masks these in output:
          # echo "Connecting to DB with password" 
          # Output: Connecting to DB with password ****
          
          docker login -u myuser -p $DOCKERHUB_TOKEN
          docker push myimage:latest
          
          # Access AWS
          aws s3 ls
```

#### GitHub Secrets Setup

```bash
# Store secrets:
# Repository Settings → Secrets and variables → Actions → New repository secret

# Or via CLI:
$ gh secret set DATABASE_PASSWORD --body "secret_value"
$ gh secret set AWS_ACCESS_KEY_ID --body "AKIAIOSFODNN7EXAMPLE"

# List secrets:
$ gh secret list

# Use in workflow:
- env:
    SECRET_NAME: ${{ secrets.SECRET_NAME }}
  run: deploy.sh

# Important: Secrets are NOT available in PR workflows from forks!
# (Security: prevent malicious forks from accessing secrets)
```

### Workflow 2: HashiCorp Vault (Enterprise)

```yaml
name: Deploy with Vault

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v3
      
      # Authenticate to Vault using GitHub token
      - name: Get secrets from Vault
        uses: HashiCorp/vault-action@v2
        with:
          url: https://vault.example.com
          method: jwt
          role: github-actions
          jwtGithubAudience: https://vault.example.com
          secrets: |
            secret/data/prod/database database_password ;
            secret/data/prod/aws access_key | AWS_ACCESS_KEY ;
            secret/data/prod/docker docker_token | DOCKER_TOKEN
      
      # Secrets now in environment variables
      - name: Deploy
        run: |
          docker login -u myuser -p $DOCKER_TOKEN
          aws s3 ls
      
      # Vault secrets auto-rotated in background!
```

#### Vault Setup

```bash
# Enable JWT auth for GitHub
$ vault auth enable jwt
$ vault write auth/jwt/config \
    oidc_discovery_url="https://token.actions.githubusercontent.com" \
    oidc_discovery_ca_pem=@ca.pem \
    bound_audiences="https://vault.example.com"

# Create role for GitHub Actions
$ vault write auth/jwt/role/github-actions \
    bound_audiences="https://vault.example.com" \
    allowed_redirect_uris="https://github.com/*" \
    user_claim="actor"

# Create secret
$ vault kv put secret/prod/database \
    password="$(openssl rand -base64 32)"

# Rotate secret (Vault handles this!)
$ vault lease renew secret/prod/database
```

### Workflow 3: AWS Secrets Manager

```yaml
name: Deploy with AWS Secrets

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    permissions:
      id-token: write
      contents: read
    
    steps:
      - uses: actions/checkout@v3
      
      # Use OIDC to get temporary AWS credentials
      # (no long-lived keys!)
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::123456789000:role/GitHubActionsRole
          aws-region: us-east-1
      
      # Get secrets from AWS Secrets Manager
      - name: Get secrets
        run: |
          # Retrieve single secret
          DB_PASSWORD=$(aws secretsmanager get-secret-value \
            --secret-id prod/database/password \
            --query SecretString \
            --output text)
          export DATABASE_PASSWORD=$DB_PASSWORD
          
          # Retrieve JSON secret
          AWS_CREDS=$(aws secretsmanager get-secret-value \
            --secret-id prod/aws/credentials \
            --query SecretString \
            --output text)
          
          # Deploy using secrets
          ./deploy.sh
      
      # Secrets marked SecureString are encrypted!
```

#### AWS Setup

```bash
# Store secret
$ aws secretsmanager create-secret \
    --name prod/database/password \
    --secret-string 'complex-password-123'

# Retrieve secret
$ aws secretsmanager get-secret-value \
    --secret-id prod/database/password

# Rotate automatically
$ aws secretsmanager rotate-secret \
    --secret-id prod/database/password \
    --rotation-rules AutomaticallyAfterDays=30
```

### Workflow 4: Multi-Secret Management

```yaml
name: Complex Secrets Management

on:
  push:
    branches: [main]

env:
  # Public values (not secrets!)
  LOG_LEVEL: INFO
  APP_ENV: production
  AWS_REGION: us-east-1

jobs:
  setup-secrets:
    runs-on: ubuntu-latest
    outputs:
      vault_token: ${{ steps.vault.outputs.vault_token }}
    
    steps:
      - uses: HashiCorp/vault-action@v2
        id: vault
        with:
          url: https://vault.example.com
          method: jwt
          role: github-actions
          jwtGithubAudience: https://vault.example.com
          secrets: |
            secret/data/prod/master master_key | MASTER_KEY ;
            secret/data/prod/db db_host | DB_HOST ;
            secret/data/prod/db db_user | DB_USER ;
            secret/data/prod/db db_pass | DB_PASS

  build:
    needs: setup-secrets
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Build with secrets injected
        env:
          MASTER_KEY: ${{ secrets.MASTER_KEY }}
          DB_HOST: ${{ secrets.DB_HOST }}
          DB_USER: ${{ secrets.DB_USER }}
          DB_PASS: ${{ secrets.DB_PASS }}
        run: |
          # Secrets available during build
          npm run build
          
          # After step completes, secrets cleared!

  # Secrets NOT automatically available in next job!
  # Must explicitly pass between jobs if needed
  deploy:
    needs: [setup-secrets, build]
    runs-on: ubuntu-latest
    
    steps:
      # Must authenticate to Vault again for deploy job!
      - uses: HashiCorp/vault-action@v2
        with:
          url: https://vault.example.com
          method: jwt
          role: github-actions
          jwtGithubAudience: https://vault.example.com
          secrets: |
            secret/data/prod/deploy deploy_key | DEPLOY_KEY
      
      - name: Deploy
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
        run: deploy.sh
```

---

## 4. Common Mistakes

### Mistake 1: Hardcoding Secrets

```yaml
# ❌ WRONG
- name: Deploy
  run: |
    docker login -u myuser -p "my_password_123"
    # Password visible in workflow file!
    # Visible in Git history!
    # Anyone with repo access sees it!

# ✅ CORRECT
- name: Deploy
  env:
    DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
  run: |
    docker login -u myuser -p $DOCKER_PASSWORD
    # Password only in secrets manager
    # Not in repo
    # Not in logs (GitHub masks it)
```

### Mistake 2: Committing .env Files

```bash
# ❌ WRONG
.env  # Contains: DB_PASSWORD=secret
# Committed to Git history!
# Recoverable even after delete!
# Visible to anyone with repo access!

# ✅ CORRECT
# .gitignore
.env
.env.local
.env.*.local

# Templates instead:
# .env.example
DB_PASSWORD=<set-via-ci>
```

### Mistake 3: Secrets in Container Images

```yaml
# ❌ WRONG
FROM node:18
ENV DATABASE_PASSWORD=secret123
RUN npm install
# Image contains secret!
# Anyone with image access sees it!

# ✅ CORRECT
FROM node:18
# NO hardcoded secrets!
RUN npm install

# Pass at runtime:
$ docker run \
    -e DATABASE_PASSWORD=$SECRET \
    myimage
```

### Mistake 4: No Secret Rotation

```bash
# ❌ WRONG
# Secret created once, never changed
# If compromised, attacker has permanent access

# ✅ CORRECT
# Rotate every 30-90 days
# Use automatic rotation (AWS, Vault)

# Vault example:
$ vault lease renew -increment=86400 \
    auth/approle/login/my-role

# AWS example:
aws secretsmanager rotate-secret \
  --secret-id prod/database \
  --rotation-rules AutomaticallyAfterDays=30
```

### Mistake 5: Secrets in Logs

```bash
# ❌ WRONG
- name: Deploy
  env:
    API_KEY: ${{ secrets.API_KEY }}
  run: |
    echo "Starting deployment with key: $API_KEY"
    # Logs show: "Starting deployment with key: sk-1234567890"
    # Log viewer can see secret!

# ✅ CORRECT
- name: Deploy
  env:
    API_KEY: ${{ secrets.API_KEY }}
  run: |
    echo "Starting deployment"
    # GitHub automatically masks secrets in output
    # But be careful NOT to echo/print secrets!
    
    # Use `--silent` or redirect to /dev/null
    curl --silent -H "Authorization: Bearer $API_KEY" ...
```

---

## 5. Debugging Secret Issues

### Problem: "Secret Not Found" Error

```bash
# Error: "Undefined secret MY_SECRET"

# Debug:
1. Check secret name (case-sensitive!)
   $ gh secret list | grep -i my_secret

2. Check secret exists in right place
   # Repository secrets? Organization secrets? Environment secrets?
   Settings → Secrets → Which scope?

3. Check workflow can access secret
   # For organization secrets: must be in same org
   # For environment secrets: must match environment name

4. Check job permissions
   # Some job types can't access secrets
   if: env.MY_SECRET != '' might fail
   # Better: directly use secret in run step

# Fix:
# Ensure secret name matches exactly:
MY_SECRET: ${{ secrets.MY_SECRET }}  # Case matters!
```

### Problem: Secret Masked in Output But Shouldn't Be

```bash
# GitHub sometimes over-masks!
# Output shows: ***secret value***
# But I need to see it for debugging!

# Solution:
# 1. GitHub logs are secure (only team can view)
# 2. Download and decrypt locally
# 3. Or use workflow debugging

# Run workflow with debug:
$ gh secret set ACTIONS_STEP_DEBUG --body "true"

# Or set in workflow:
- name: Debug
  env:
    RUNNER_DEBUG: 1
  run: bash -x deploy.sh
```

---

## 6. Interview Questions (Security-Focused)

### **Q1**: Design a secrets management system for 50 engineers deploying to 3 cloud providers daily, with audit compliance requirements.

**Answer**:
```yaml
---
Architecture:

# Layer 1: Secret Storage (Centralized)
├─ HashiCorp Vault (self-managed)
│  ├─ Database credentials
│  ├─ API tokens
│  └─ Encryption keys
│
├─ AWS Secrets Manager (for AWS resources)
│  ├─ RDS passwords
│  └─ Application secrets
│
├─ GitHub Secrets (for CI/CD)
│  ├─ Docker tokens
│  └─ Non-sensitive API keys

# Layer 2: Authentication (No long-lived credentials)
├─ OIDC (Open ID Connect)
│  ├─ GitHub Actions → Vault
│  ├─ GitHub Actions → AWS
│  └─ GitHub Actions → GCP
│
└─ Result: Temporary credentials only!

# Layer 3: Authorization (Role-based)
├─ Role: Backend-Deploy
│  ├─ Access: prod/backend/* secrets
│  ├─ Duration: 1 hour (auto-revoke)
│  └─ Audit log: All access tracked
│
├─ Role: DevOps-Team
│  ├─ Access: All secrets
│  ├─ Requires: 2FA + approval
│  └─ Audit log: Every access logged
│
└─ Role: Junior-Developer
   ├─ Access: staging/* secrets only
   ├─ Duration: Explicit approval per access
   └─ Audit log: Flagged for review

# Layer 4: Rotation (Automatic expiration)
├─ Database passwords: Every 30 days
├─ API tokens: Every 90 days
├─ SSH keys: Every 180 days
└─ Vault automatically handles rotation!

# Layer 5: Audit (Compliance)
├─ All secret access logged
├─ Who: Engineer ID
├─ What: Secret name
├─ When: Timestamp
├─ Where: Job ID
├─ Retention: 7 years (compliance)
└─ Exported to: CloudTrail, Datadog, SIEM

# Implementation:

.github/workflows/deploy.yml:

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    permissions:
      id-token: write  # OIDC token
      contents: read
    
    steps:
      # Authenticate to Vault using OIDC
      # No long-lived credentials!
      - uses: hashicorp/vault-action@v2
        with:
          url: https://vault.example.com
          method: jwt
          role: github-actions-backend  # Role-based access!
          jwtGithubAudience: https://vault.example.com
          secrets: |
            secret/data/prod/backend db_password ;
            secret/data/prod/backend api_token
      
      - name: Deploy
        env:
          DB_PASSWORD: ${{ env.db_password }}
          API_TOKEN: ${{ env.api_token }}
        run: |
          # Vault rotates secrets in background
          # Our job always gets fresh credentials
          ./deploy.sh

# Benefits:
✅ No hardcoded secrets
✅ Automatic rotation
✅ Audit compliance
✅ Role-based access
✅ Temporary credentials
✅ Zero-trust architecture
```

---

## 7. Security Checklist

```yaml
Secret Management Security:

✅ Storage
  ├─ [ ] Secrets encrypted at rest
  ├─ [ ] Encryption at transit (TLS)
  ├─ [ ] Access control (RBAC)
  └─ [ ] Separate storage per environment

✅ Generation
  ├─ [ ] Cryptographically secure (openssl, not random )
  ├─ [ ] Appropriate length (32+ chars for passwords)
  ├─ [ ] Complex characters (upper, lower, number, symbol)
  └─ [ ] Never expose during generation

✅ Rotation
  ├─ [ ] Automatic rotation enabled
  ├─ [ ] Rotation frequency documented (30-90 days)
  ├─ [ ] Old secrets still valid during transition
  └─ [ ] Verification after rotation

✅ Access
  ├─ [ ] OIDC or temporary credentials (never long-lived)
  ├─ [ ] RBAC implemented (roles, not users)
  ├─ [ ] MFA required for manual access
  ├─ [ ] Principle of least privilege (minimum access)
  └─ [ ] Just-in-time (JIT) access approval

✅ Audit
  ├─ [ ] All access logged
  ├─ [ ] Logs immutable (S3 versioning, append-only)
  ├─ [ ] Logs secured (separate from secrets!)
  ├─ [ ] Retention policy (7+ years for compliance)
  └─ [ ] Alerting for unusual access

✅ Incident Response
  ├─ [ ] Secret compromise procedure documented
  ├─ [ ] Immediate revocation possible
  ├─ [ ] Rotation on-demand available
  ├─ [ ] Blast radius (dependent services) identified
  └─ [ ] Communication plan for security team

✅ Development
  ├─ [ ] Developers never see production secrets
  ├─ [ ] .env files in .gitignore
  ├─ [ ] Templates for configuration
  ├─ [ ] Different secrets per environment
  └─ [ ] Secret scanning in CI/CD
```

---

## 8. Advanced Insights

### Secret Scanning

```yaml
# GitHub Secret Scanning
# Automatically detect secrets committed to repo

name: Secret Scanning

on: [push]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      # Find secrets in code
      - uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: origin/main
          extra_args: --json

      # If secrets found, fail workflow
      - name: Check for secrets
        run: |
          if trufflehog filesystem . --json | grep -q "Verified"; then
            echo "❌ SECRETS FOUND - commit blocked!"
            exit 1
          fi
```

### Encryption in Flight

```bash
# Always use TLS for secret transmission
# GitHub → Vault: HTTPS (TLS 1.3)
# Vault → Application: TLS + mTLS

# Example: Vault client authentication
$ curl -H "X-Vault-Token: $VAULT_TOKEN" \
    https://vault.example.com/v1/secret/data/prod \
    --cacert /vault/ca.pem \
    --cert /vault/client.crt \
    --key /vault/client.key
```

### Secret Sprawl Prevention

```bash
# Problem: Secrets scattered across systems
# ├─ GitHub Actions secrets
# ├─ Dockerfile ENV
# ├─ Docker Compose .env
# ├─ Kubernetes secrets
# ├─ Application config files
# └─ Developer laptops!

# Solution: Single source of truth

# All secrets from Vault:
- GitHub Actions → Vault (via JWT auth)
- Application startup → Vault (via role)
- Kubernetes → Vault (ServiceAccount auth)
- Developers → Vault (via CLI with MFA)

# Result: Centralized, audited, rotated!
```

---

## Conclusion

Secrets management is the **foundation of security**:

✅ **Store securely**: Encrypted at rest and in transit  
✅ **Access safely**: OIDC, RBAC, MFA  
✅ **Rotate regularly**: Automatic expiration  
✅ **Audit thoroughly**: Full access logs  
✅ **Respond quickly**: Immediate revocation  

Master secrets management and you:
- ✅ Prevent credential leaks
- ✅ Meet compliance requirements
- ✅ Detect compromises early
- ✅ Limit blast radius
- ✅ Enable zero-trust architecture
- ✅ Sleep at night knowing secrets are safe!
