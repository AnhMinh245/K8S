# 11.3. CI/CD Integration với Kubernetes

> Automated deployment pipeline từ code đến production

---

## 🎯 Mục Tiêu

- ✅ Hiểu GitOps workflow
- ✅ Setup CI pipeline (build + test)
- ✅ Setup CD pipeline (deploy)
- ✅ Implement security scanning
- ✅ Automated rollback

---

## 🔄 GitOps Workflow

### GitOps là gì?

**GitOps** = Git as single source of truth cho infrastructure và applications

```
┌──────────────────────────────────────────────────┐
│              GITOPS WORKFLOW                     │
├──────────────────────────────────────────────────┤
│                                                  │
│  1. Developer pushes code                       │
│     Git Repository (GitHub/GitLab)              │
│           ↓                                      │
│  2. CI Pipeline triggers                        │
│     • Build Docker image                        │
│     • Run tests                                 │
│     • Security scan                             │
│     • Push to registry                          │
│           ↓                                      │
│  3. Update manifests                            │
│     • New image tag trong YAML                  │
│     • Commit to GitOps repo                     │
│           ↓                                      │
│  4. ArgoCD/Flux detects change                  │
│     • Sync với cluster                          │
│     • Apply manifests                           │
│     • Monitor health                            │
│           ↓                                      │
│  5. Application deployed!                       │
│                                                  │
└──────────────────────────────────────────────────┘
```

---

## 🏗️ Repository Structure

### Recommended: Separate Repos

**1. Application Repository (app-repo)**
```
app-repo/
├── src/                    # Source code
├── tests/                  # Unit & integration tests
├── Dockerfile              # Container definition
├── .gitlab-ci.yml         # CI pipeline
└── README.md
```

**2. GitOps Repository (gitops-repo)**
```
gitops-repo/
├── base/                   # Base manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/               # Dev environment
│   │   ├── kustomization.yaml
│   │   └── patch.yaml
│   ├── staging/           # Staging environment
│   │   ├── kustomization.yaml
│   │   └── patch.yaml
│   └── prod/              # Production environment
│       ├── kustomization.yaml
│       └── patch.yaml
└── README.md
```

---

## 🔨 CI Pipeline (GitLab CI Example)

### .gitlab-ci.yml

```yaml
stages:
  - build
  - test
  - security
  - publish
  - deploy

variables:
  DOCKER_IMAGE: gcr.io/${GCP_PROJECT_ID}/myapp
  DOCKER_TAG: ${CI_COMMIT_SHORT_SHA}

# Build Docker image
build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
    - docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest
  only:
    - main
    - develop

# Run unit tests
test:unit:
  stage: test
  image: node:18
  script:
    - npm install
    - npm run test:unit
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

# Run integration tests
test:integration:
  stage: test
  image: node:18
  services:
    - postgres:14
  variables:
    POSTGRES_DB: test_db
    POSTGRES_USER: test_user
    POSTGRES_PASSWORD: test_pass
  script:
    - npm install
    - npm run test:integration

# Security: Trivy container scanning
security:trivy:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL ${DOCKER_IMAGE}:${DOCKER_TAG}
  allow_failure: false

# Security: SAST (Static Application Security Testing)
security:sast:
  stage: security
  image: returntocorp/semgrep
  script:
    - semgrep --config=auto --json --output=sast-report.json .
  artifacts:
    reports:
      sast: sast-report.json

# Security: Dependency scanning
security:dependencies:
  stage: security
  image: node:18
  script:
    - npm audit --audit-level=high
  allow_failure: false

# Publish image to registry
publish:
  stage: publish
  image: google/cloud-sdk:alpine
  script:
    # Authenticate with GCR
    - echo ${GCP_SERVICE_KEY} | base64 -d > ${HOME}/gcp-key.json
    - gcloud auth activate-service-account --key-file ${HOME}/gcp-key.json
    - gcloud auth configure-docker
    
    # Push images
    - docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
    - docker push ${DOCKER_IMAGE}:latest
  only:
    - main

# Update GitOps repo
deploy:update-manifest:
  stage: deploy
  image: alpine/git
  script:
    # Clone GitOps repo
    - git clone https://${GITLAB_TOKEN}@gitlab.com/yourorg/gitops-repo.git
    - cd gitops-repo
    
    # Update image tag
    - |
      if [ "${CI_COMMIT_BRANCH}" == "main" ]; then
        OVERLAY="prod"
      else
        OVERLAY="dev"
      fi
    
    - cd overlays/${OVERLAY}
    - kustomize edit set image ${DOCKER_IMAGE}:${DOCKER_TAG}
    
    # Commit and push
    - git config user.email "ci@example.com"
    - git config user.name "GitLab CI"
    - git add .
    - git commit -m "Update ${OVERLAY} image to ${DOCKER_TAG}"
    - git push origin main
  only:
    - main
    - develop
```

---

## 🚀 CD Pipeline với ArgoCD

### Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### ArgoCD Application Manifest

**app-production.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
spec:
  # Source: GitOps repository
  source:
    repoURL: https://github.com/yourorg/gitops-repo.git
    targetRevision: main
    path: overlays/prod
  
  # Destination: Production cluster
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  # Sync policy
  syncPolicy:
    automated:
      prune: true      # Delete resources không còn trong Git
      selfHeal: true   # Auto-sync nếu cluster drift
      allowEmpty: false
    
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  # Health check
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas  # Ignore replicas (HPA manages this)
```

**Deploy Application:**
```bash
kubectl apply -f app-production.yaml -n argocd
```

---

## 🔍 Health Checks trong CI/CD

### Pre-deployment Checks

**check-deployment.sh:**
```bash
#!/bin/bash
set -e

NAMESPACE=$1
DEPLOYMENT=$2
TIMEOUT=300  # 5 minutes

echo "Checking deployment: ${DEPLOYMENT} in namespace: ${NAMESPACE}"

# Wait for deployment rollout
kubectl rollout status deployment/${DEPLOYMENT} -n ${NAMESPACE} --timeout=${TIMEOUT}s

# Check pod status
PODS=$(kubectl get pods -n ${NAMESPACE} -l app=${DEPLOYMENT} -o jsonpath='{.items[*].metadata.name}')

for POD in ${PODS}; do
  echo "Checking pod: ${POD}"
  
  # Check if pod is Running
  STATUS=$(kubectl get pod ${POD} -n ${NAMESPACE} -o jsonpath='{.status.phase}')
  if [ "${STATUS}" != "Running" ]; then
    echo "Pod ${POD} is not Running (status: ${STATUS})"
    exit 1
  fi
  
  # Check if containers are ready
  READY=$(kubectl get pod ${POD} -n ${NAMESPACE} -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')
  if [ "${READY}" != "True" ]; then
    echo "Pod ${POD} is not Ready"
    kubectl describe pod ${POD} -n ${NAMESPACE}
    exit 1
  fi
done

echo "✅ Deployment ${DEPLOYMENT} is healthy!"
```

---

### Smoke Tests

**smoke-test.sh:**
```bash
#!/bin/bash
set -e

SERVICE_URL=$1

echo "Running smoke tests against: ${SERVICE_URL}"

# Health check endpoint
echo "1. Testing /health endpoint"
HEALTH=$(curl -s -o /dev/null -w "%{http_code}" ${SERVICE_URL}/health)
if [ ${HEALTH} -ne 200 ]; then
  echo "❌ Health check failed (HTTP ${HEALTH})"
  exit 1
fi
echo "✅ Health check passed"

# API endpoint
echo "2. Testing /api/status endpoint"
STATUS=$(curl -s -o /dev/null -w "%{http_code}" ${SERVICE_URL}/api/status)
if [ ${STATUS} -ne 200 ]; then
  echo "❌ Status endpoint failed (HTTP ${STATUS})"
  exit 1
fi
echo "✅ Status endpoint passed"

# Response time check
echo "3. Testing response time"
RESPONSE_TIME=$(curl -s -o /dev/null -w "%{time_total}" ${SERVICE_URL}/api/status)
if (( $(echo "${RESPONSE_TIME} > 1.0" | bc -l) )); then
  echo "⚠️  Warning: Response time is ${RESPONSE_TIME}s (> 1s)"
else
  echo "✅ Response time OK (${RESPONSE_TIME}s)"
fi

echo "✅ All smoke tests passed!"
```

---

## 🔄 Progressive Delivery với Flagger

### Canary Deployment

**flagger-canary.yaml:**
```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
  namespace: production
spec:
  # Target deployment
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  
  # Service configuration
  service:
    port: 80
    targetPort: 8080
    gateways:
    - public-gateway
    hosts:
    - myapp.example.com
  
  # Canary analysis
  analysis:
    # Check interval
    interval: 1m
    
    # Max traffic percentage
    threshold: 5
    
    # Max weight for canary
    maxWeight: 50
    
    # Increment step
    stepWeight: 5
    
    # Metrics for analysis
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m
    
    # Webhooks for custom checks
    webhooks:
    - name: acceptance-test
      type: pre-rollout
      url: http://flagger-loadtester.test/
      timeout: 30s
      metadata:
        type: bash
        cmd: "curl -sd 'test' http://myapp-canary:80/token | grep token"
    
    - name: load-test
      url: http://flagger-loadtester.test/
      timeout: 5s
      metadata:
        type: cmd
        cmd: "hey -z 1m -q 10 -c 2 http://myapp-canary.production/"
```

**Canary Workflow:**
```
1. Deploy new version
   ↓
2. Route 5% traffic to canary
   ↓
3. Run analysis (1 minute)
   ├─ Success rate > 99%?
   ├─ Response time < 500ms?
   └─ Custom checks pass?
   ↓
4. If OK → Increase to 10%
   If NOT OK → Rollback
   ↓
5. Repeat until 50% traffic
   ↓
6. Promote canary to primary
   ↓
7. Done!
```

---

## 🛡️ Security trong CI/CD

### Image Signing

**Cosign (Sigstore):**
```bash
# Generate key pair
cosign generate-key-pair

# Sign image
cosign sign --key cosign.key ${DOCKER_IMAGE}:${TAG}

# Verify signature
cosign verify --key cosign.pub ${DOCKER_IMAGE}:${TAG}
```

**Policy Enforcement:**
```yaml
# admission-policy.yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
  - glob: "gcr.io/myproject/*"
  authorities:
  - key:
      data: |
        -----BEGIN PUBLIC KEY-----
        ...
        -----END PUBLIC KEY-----
```

---

### Secrets Management

**External Secrets Operator:**

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  
  secretStoreRef:
    name: gcpsm-secret-store
    kind: SecretStore
  
  target:
    name: app-secrets
    creationPolicy: Owner
  
  data:
  - secretKey: database-password
    remoteRef:
      key: prod-db-password
  
  - secretKey: api-key
    remoteRef:
      key: prod-api-key
```

**SecretStore (Google Secret Manager):**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcpsm-secret-store
  namespace: production
spec:
  provider:
    gcpsm:
      projectID: "my-project"
      auth:
        workloadIdentity:
          clusterLocation: us-central1
          clusterName: production-cluster
          serviceAccountRef:
            name: external-secrets-sa
```

---

## 📊 Monitoring CI/CD Pipeline

### Metrics to Track

**Build Metrics:**
```
✓ Build success rate
✓ Build duration
✓ Time to detect failures
✓ Flaky test rate
```

**Deployment Metrics:**
```
✓ Deployment frequency
✓ Lead time (commit → production)
✓ Mean time to recovery (MTTR)
✓ Change failure rate
```

**Grafana Dashboard Query (Prometheus):**
```promql
# Deployment frequency (per day)
sum(increase(argocd_app_sync_total{phase="Succeeded"}[24h]))

# Average deployment duration
avg(argocd_app_reconcile_duration_seconds)

# Deployment success rate
sum(rate(argocd_app_sync_total{phase="Succeeded"}[1h]))
/
sum(rate(argocd_app_sync_total[1h]))
```

---

## 🎯 Best Practices

### DO ✅

**1. Separate CI và CD:**
```
CI (Build & Test) → Fast feedback
CD (Deploy) → GitOps-based, declarative
```

**2. Immutable images:**
```
✓ Tag images với commit SHA
✓ Không dùng :latest tag
✓ Scan images for vulnerabilities
```

**3. Test pyramid:**
```
Unit tests (nhiều nhất)
    ↑
Integration tests
    ↑
E2E tests (ít nhất, chậm nhất)
```

**4. Progressive delivery:**
```
✓ Canary deployments
✓ Feature flags
✓ Automated rollback
```

**5. Security:**
```
✓ Scan code (SAST)
✓ Scan dependencies
✓ Scan container images
✓ Sign images
✓ Policy enforcement
```

---

### DON'T ❌

```
✗ Deploy directly to production
✗ Skip tests để deploy nhanh
✗ Use :latest tag
✗ Store secrets trong Git
✗ Manual deployments
✗ No rollback plan
✗ Ignore security scans
✗ Deploy straight to 100% traffic
```

---

## 🎓 Key Takeaways

**1. GitOps:**
- Git as single source of truth
- Declarative infrastructure
- Automated sync

**2. CI Pipeline:**
- Build → Test → Scan → Publish
- Fast feedback loop
- Security built-in

**3. CD Pipeline:**
- ArgoCD/Flux for GitOps
- Automated deployments
- Health checks và rollback

**4. Progressive Delivery:**
- Canary deployments
- Gradual traffic shifting
- Automated rollback

**5. Security:**
- Image scanning
- Secrets management
- Policy enforcement

---

[⬅️ 11.2. Deployment Strategies](./02-deployment-strategies.md) | [➡️ 11.4. Security](./04-security-hardening.md) | [🏠 Mục Lục](../README.md)

