# 3.4. Labels & Selectors

> Query và organize resources hiệu quả với Labels

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **Labels và Selectors** là gì
- ✅ Sử dụng **Labels để organize** resources
- ✅ **Query resources** với Selectors
- ✅ Phân biệt **Labels vs Annotations**
- ✅ **Best practices** cho labeling strategy

---

## 🏷️ Labels Là Gì?

### Định Nghĩa

**Labels** = Key-value pairs gắn vào K8s objects để identify và organize.

### Giải Thích Bằng Ví Dụ

**Labels giống như tags/stickers trên hồ sơ:**

```
📂 HỒ SƠ NHÂN VIÊN (Pod)

Name: John Doe
Labels/Tags:
├── 🏷️ department: engineering
├── 🏷️ role: backend-developer
├── 🏷️ level: senior
├── 🏷️ team: payments
├── 🏷️ location: hanoi
└── 🏷️ project: e-commerce

Use cases:
✓ Find all engineers: department=engineering
✓ Find senior devs: level=senior
✓ Find Hanoi team: location=hanoi
✓ Find payments team: team=payments
```

### Labels trong K8s

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-frontend-1
  labels:
    app: webapp              # Application name
    tier: frontend           # Architecture tier
    environment: production  # Environment
    version: v2.1            # Version
    owner: team-alpha        # Team ownership
```

---

## 🎯 TẠI SAO Cần Labels?

### Vấn Đề Không Có Labels

```bash
# 100 Pods trong cluster
$ kubectl get pods

NAME                     READY   STATUS    RESTARTS   AGE
webapp-frontend-1        1/1     Running   0          5d
webapp-frontend-2        1/1     Running   0          5d
webapp-backend-1         1/1     Running   0          5d
webapp-backend-2         1/1     Running   0          5d
database-primary         1/1     Running   0          10d
database-replica-1       1/1     Running   0          10d
...95 more pods...

Questions:
❓ Làm sao tìm tất cả frontend Pods?
❓ Làm sao tìm production Pods?
❓ Làm sao tìm Pods của team-alpha?
❓ Làm sao update chỉ v2.1 Pods?

→ Khó! Phải dựa vào naming convention (dễ sai)
```

### Giải Pháp: Labels + Selectors

```bash
# Tất cả frontend Pods
$ kubectl get pods -l tier=frontend

# Production Pods
$ kubectl get pods -l environment=production

# Team alpha's Pods
$ kubectl get pods -l owner=team-alpha

# v2.1 Pods
$ kubectl get pods -l version=v2.1

# Multiple labels (AND)
$ kubectl get pods -l tier=frontend,environment=production

✓ Clean, flexible, powerful!
```

---

## 📝 Label Syntax

### Key Rules

```yaml
# Valid label format
labels:
  # Simple
  app: webapp
  tier: frontend
  
  # With prefix (recommended for organization)
  example.com/app: webapp
  team.company.io/owner: team-alpha
  
  # Can contain
  valid-label_123: value-ABC_xyz

# Key constraints:
# - Prefix (optional): DNS subdomain (max 253 chars)
# - Name (required): max 63 chars
# - Can contain: a-z, A-Z, 0-9, -, _, .
# - Must start/end with alphanumeric

# Value constraints:
# - Max 63 chars
# - Can be empty
# - Same character rules as key
```

### Common Label Keys

```yaml
# Recommended labels (k8s.io maintained)
app.kubernetes.io/name: wordpress
app.kubernetes.io/instance: wordpress-abcxzy
app.kubernetes.io/version: "4.9.6"
app.kubernetes.io/component: database
app.kubernetes.io/part-of: wordpress
app.kubernetes.io/managed-by: helm

# Custom organization labels
environment: production
tier: frontend
team: team-alpha
cost-center: engineering
business-unit: ecommerce
```

---

## 🔍 Selectors

### Equality-Based Selectors

**Format:** `key=value` or `key!=value`

```bash
# Exact match
kubectl get pods -l environment=production
kubectl get pods -l tier=frontend

# Not equal
kubectl get pods -l environment!=production

# Multiple (AND logic)
kubectl get pods -l environment=production,tier=frontend
# Returns Pods with BOTH labels
```

### Set-Based Selectors

**Format:** `key in (value1,value2)` or `key notin (value1,value2)`

```bash
# In set
kubectl get pods -l 'environment in (production,staging)'
# Pods với environment=production OR staging

# Not in set
kubectl get pods -l 'tier notin (frontend,backend)'
# Pods không có tier=frontend và không có tier=backend

# Key exists
kubectl get pods -l environment
# Pods có label key 'environment' (bất kỳ value nào)

# Key does not exist
kubectl get pods -l '!environment'
# Pods KHÔNG có label key 'environment'
```

### Complex Queries

```bash
# Combine equality và set-based
kubectl get pods -l 'environment=production,tier in (frontend,backend)'

# Multiple conditions
kubectl get pods -l 'environment=production,tier=frontend,version notin (v1.0,v1.1)'
```

---

## 🎯 Labels trong Actions

### Add Labels

```bash
# Add label to existing Pod
kubectl label pods nginx-pod tier=frontend

# Add label to Node
kubectl label nodes worker-1 disktype=ssd

# Add multiple labels
kubectl label pods nginx-pod tier=frontend environment=dev

# Verify
kubectl get pods --show-labels
```

### Update Labels

```bash
# Update existing label (use --overwrite)
kubectl label pods nginx-pod tier=backend --overwrite

# Without --overwrite → Error if exists
kubectl label pods nginx-pod tier=backend
# Error: 'tier' already has a value (frontend)
```

### Remove Labels

```bash
# Remove label (use minus sign)
kubectl label pods nginx-pod tier-

# Verify
kubectl get pods nginx-pod --show-labels
```

---

## 🎭 Labels trong Resource Specs

### Pod với Labels

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-pod
  labels:
    app: webapp
    tier: frontend
    environment: production
    version: v2.1
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

### Deployment với Labels

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp              # Deployment's labels
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp            # Select Pods với label này
      tier: frontend
  template:
    metadata:
      labels:
        app: webapp          # Labels cho Pods tạo ra
        tier: frontend
        version: v2.1
    spec:
      containers:
      - name: nginx
        image: nginx:latest
```

**Important:** `spec.selector.matchLabels` MUST match `spec.template.metadata.labels`!

---

## 🔗 Selectors trong Services

### Service → Pods

```yaml
# Pods với labels
apiVersion: v1
kind: Pod
metadata:
  name: webapp-1
  labels:
    app: webapp
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80

---
# Service selects Pods
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp        # Select ALL Pods với app=webapp
    tier: frontend     # AND tier=frontend
  ports:
  - port: 80
    targetPort: 80
```

**How it works:**

```
Service: webapp-service
Selector: app=webapp, tier=frontend
    ↓
Looks for Pods với matching labels
    ↓
Finds:
├── webapp-1 (app=webapp, tier=frontend) ✓
├── webapp-2 (app=webapp, tier=frontend) ✓
├── webapp-3 (app=webapp, tier=backend) ✗ (tier mismatch)
└── api-1 (app=api, tier=frontend) ✗ (app mismatch)
    ↓
Service load balances to: webapp-1, webapp-2
```

---

## 📊 Labels vs Annotations

### Khác Biệt

| Feature | Labels | Annotations |
|---------|--------|-------------|
| **Purpose** | Identify, select, organize | Non-identifying metadata |
| **Used by** | Selectors, queries | Humans, tools |
| **Size** | Small (63 chars) | Larger (256 KB) |
| **Structure** | Simple key-value | Can be complex (JSON, etc.) |
| **Query** | Yes (kubectl -l) | No |

### When to Use What

**Use Labels for:**
```yaml
labels:
  app: webapp                # Selectable
  environment: production    # Queryable
  tier: frontend            # Organize
  version: v2.1             # Filter
```

**Use Annotations for:**
```yaml
annotations:
  # Build/deployment info
  build-timestamp: "2024-01-01T10:00:00Z"
  git-commit: "abc123def456"
  deployed-by: "jenkins-pipeline"
  
  # Tool-specific metadata
  prometheus.io/scrape: "true"
  prometheus.io/port: "9090"
  
  # Documentation
  description: "Frontend web application"
  contact: "team-alpha@company.com"
  
  # Complex data (JSON)
  config: |
    {
      "feature-flags": ["new-ui", "beta-feature"],
      "rate-limits": {"rpm": 1000}
    }
```

---

## 🎨 Labeling Strategies

### Strategy 1: Multi-Dimensional

```yaml
# Good: Multiple dimensions
labels:
  app: webapp
  tier: frontend
  environment: production
  version: v2.1
  team: team-alpha
  cost-center: engineering

Allows queries:
├── All webapp Pods
├── All frontend Pods
├── All production Pods
├── All v2.1 Pods
├── All team-alpha Pods
└── Combinations: production frontend webapp Pods
```

### Strategy 2: Hierarchical

```yaml
# Organize by hierarchy
labels:
  organization: company
  business-unit: ecommerce
  product: webapp
  component: frontend
  sub-component: user-authentication

Query by level:
├── All company resources
├── All ecommerce resources
├── All webapp resources
└── Specific components
```

### Strategy 3: Standardized

```yaml
# Use standard keys across org
labels:
  # Application identity
  app.kubernetes.io/name: wordpress
  app.kubernetes.io/instance: wordpress-abc123
  app.kubernetes.io/version: "5.0"
  app.kubernetes.io/component: database
  app.kubernetes.io/part-of: wordpress
  
  # Management
  app.kubernetes.io/managed-by: helm
  
  # Custom org standards
  company.com/environment: production
  company.com/team: platform
  company.com/cost-center: "12345"
```

---

## 💡 Best Practices

### DO ✅

```yaml
✅ Use meaningful, consistent names
labels:
  environment: production    # Good: Clear
  tier: frontend            # Good: Standard term

✅ Use multiple dimensions
labels:
  app: webapp
  tier: frontend
  environment: production
  # Multiple query dimensions

✅ Use prefixes for organization
labels:
  company.com/team: alpha
  company.com/cost-center: eng

✅ Document labeling standards
# Create org-wide label guidelines

✅ Use labels for selection
# Labels for kubectl queries, Service selectors

✅ Use annotations for metadata
annotations:
  build-timestamp: "2024-01-01"
  # Non-selectable metadata
```

### DON'T ❌

```yaml
❌ Don't use overly long values
labels:
  description: "This is a very long description..."  # Use annotation!

❌ Don't use special characters
labels:
  team@company: alpha  # Invalid! Use team.company

❌ Don't use labels for large data
labels:
  config: "{json: 'data'...}"  # Use ConfigMap or annotation!

❌ Don't use inconsistent naming
labels:
  env: prod        # Inconsistent
  environment: dev # Pick one standard!

❌ Don't put sensitive data
labels:
  password: secret123  # NEVER! Use Secret!
```

---

## 🎓 Kiểm Tra Hiểu Biết

**1. Selector này select Pods nào?**
```bash
kubectl get pods -l 'environment in (prod,staging),tier=frontend'
```

<details>
<summary>Xem đáp án</summary>

Pods phải thỏa TẤT CẢ:
1. `environment` label = "prod" HOẶC "staging"
2. `tier` label = "frontend"

Examples match:
- {environment: prod, tier: frontend} ✓
- {environment: staging, tier: frontend} ✓

Examples don't match:
- {environment: dev, tier: frontend} ✗ (env not in set)
- {environment: prod, tier: backend} ✗ (tier mismatch)
- {environment: prod} ✗ (missing tier label)
</details>

**2. Labels vs Annotations - khi nào dùng gì?**

<details>
<summary>Xem đáp án</summary>

**Labels:**
- Dùng để: Select, query, organize
- Size: Small (63 chars)
- Examples: app=webapp, tier=frontend

**Annotations:**
- Dùng để: Metadata, tool configs, documentation
- Size: Larger (256 KB)
- Examples: build-timestamp, git-commit, tool configs

Rule: If cần query → Label. Nếu chỉ metadata → Annotation.
</details>

**3. Service selector matching - Pods nào được selected?**

```yaml
# Service
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
    env: prod
  ports:
  - port: 80

# Pods
Pod 1: {app: webapp, env: prod, tier: frontend}
Pod 2: {app: webapp, env: dev}
Pod 3: {app: webapp, env: prod}
Pod 4: {app: api, env: prod}
```

<details>
<summary>Xem đáp án</summary>

**Selected:** Pod 1, Pod 3

**Why:**
- Pod 1: ✓ app=webapp AND env=prod (extra label 'tier' OK)
- Pod 2: ✗ env=dev (mismatch)
- Pod 3: ✓ app=webapp AND env=prod
- Pod 4: ✗ app=api (mismatch)

Rule: Pods cần có TẤT CẢ labels trong selector. Extra labels OK.
</details>

---

## 💪 Bài Tập Thực Hành

### Bài 1: Labels và Selectors

```bash
# 1. Create Pods với different labels
kubectl run pod1 --image=nginx --labels="app=webapp,tier=frontend,env=prod"
kubectl run pod2 --image=nginx --labels="app=webapp,tier=backend,env=prod"
kubectl run pod3 --image=nginx --labels="app=webapp,tier=frontend,env=dev"
kubectl run pod4 --image=nginx --labels="app=api,tier=frontend,env=prod"

# 2. Query exercises
# All Pods
kubectl get pods

# Frontend Pods
kubectl get pods -l tier=frontend

# Production Pods
kubectl get pods -l env=prod

# Production frontend Pods
kubectl get pods -l tier=frontend,env=prod

# Webapp OR api
kubectl get pods -l 'app in (webapp,api)'

# NOT backend
kubectl get pods -l 'tier!=backend'

# Has 'env' label
kubectl get pods -l env

# Show labels
kubectl get pods --show-labels

# 3. Add label to existing Pod
kubectl label pods pod1 version=v2.0

# 4. Update label
kubectl label pods pod1 version=v2.1 --overwrite

# 5. Remove label
kubectl label pods pod1 version-

# 6. Cleanup
kubectl delete pods pod1 pod2 pod3 pod4
```

### Bài 2: Service với Selectors

```yaml
# deploy-with-service.yaml
---
# Deployment với labels
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      tier: frontend
  template:
    metadata:
      labels:
        app: webapp
        tier: frontend
        version: v1.0
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80

---
# Service selects Pods
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp
    tier: frontend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

```bash
# Apply
kubectl apply -f deploy-with-service.yaml

# Check Pods và labels
kubectl get pods --show-labels

# Check Service endpoints
kubectl describe service webapp-service
# Look at "Endpoints:" - should show 3 Pod IPs

# Test Service selects correct Pods
# Add a Pod với wrong labels
kubectl run wrong-pod --image=nginx --labels="app=api,tier=backend"

# Check endpoints again
kubectl describe service webapp-service
# Should still show only 3 endpoints (wrong-pod not included!)

# Cleanup
kubectl delete -f deploy-with-service.yaml
kubectl delete pod wrong-pod
```

### Bài 3: Update Rolling với Labels

```bash
# Deploy v1.0
kubectl create deployment webapp --image=nginx:1.21 --replicas=3

# Add version label
kubectl label pods -l app=webapp version=v1.0

# Check
kubectl get pods -l version=v1.0 --show-labels

# Rolling update to v2.0
kubectl set image deployment/webapp nginx=nginx:1.22

# New Pods get created, old ones terminated
# Label new Pods
kubectl label pods -l app=webapp version=v2.0 --overwrite

# Now can query by version
kubectl get pods -l version=v2.0

# Cleanup
kubectl delete deployment webapp
```

---

## 🎯 Key Takeaways

1. **Labels = Identification**
   - Key-value pairs
   - Organize, query, select resources
   - Multiple labels per object

2. **Selectors = Query Language**
   - Equality: `key=value`, `key!=value`
   - Set-based: `key in (v1,v2)`, `key notin (v1,v2)`
   - Combine for powerful queries

3. **Labels trong Controllers**
   - Services: Select Pods
   - Deployments: Manage Pods
   - ReplicaSets: Track replicas

4. **Labels vs Annotations**
   - Labels: Selectable (query)
   - Annotations: Metadata (no query)

5. **Best Practices**
   - Multi-dimensional labels
   - Consistent naming
   - Standard prefixes
   - Document standards

---

## 🚀 Hoàn Thành Phần 3!

Xin chúc mừng! Bạn đã nắm vững Core Concepts!

**Next:** [Phần 4: Workloads →](../04-workloads/README.md)

Học về Deployments, ReplicaSets, StatefulSets, và workload patterns!

---

[⬅️ 3.3. Namespaces](./03-namespaces.md) | [🏠 Mục Lục](../README.md) | [📂 Phần 3: Core Concepts](./README.md) | [➡️ Phần 4: Workloads](../04-workloads/README.md)
