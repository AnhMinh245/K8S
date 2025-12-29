# 3.4. Labels & Selectors

> Organize và query Kubernetes resources

---

## 🎯 Labels Là Gì?

**Labels** = Key-value pairs attached to objects (Pods, Services, Nodes...)

```yaml
labels:
  app: web
  environment: production
  version: v1.2.3
  tier: frontend
  team: platform
```

---

## 🏢 Ví Dụ Thực Tế

**Labels = Nhãn dán trên hồ sơ**

```
Hồ sơ A:
  [Khẩn cấp] [Phòng Kế toán] [Năm 2024] [Chưa xử lý]

Hồ sơ B:
  [Bình thường] [Phòng IT] [Năm 2024] [Đã xử lý]

Tìm kiếm:
  • Tất cả hồ sơ "Khẩn cấp"
  • Hồ sơ "Phòng IT" + "Chưa xử lý"
```

---

## 📝 Labels Syntax

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web               # Application name
    environment: prod      # Environment
    version: "1.2.3"       # Version
    tier: frontend         # Architecture tier
    release: stable        # Release channel
```

**Rules:**
- Key: `[prefix/]name` (max 63 chars)
- Value: max 63 chars
- Alphanumeric + `-`, `_`, `.`

**Common label keys:**
```
app, component, tier, environment, version, release, 
managed-by, part-of, instance
```

---

## 🔍 Selectors

**Selectors** = Query objects by labels

### Equality-Based Selector

```yaml
# Select Pods where environment = production
selector:
  environment: production
```

```bash
# kubectl
kubectl get pods -l environment=production

# Multiple labels (AND)
kubectl get pods -l environment=production,tier=frontend
```

### Set-Based Selector

```yaml
# Select Pods where environment IN (prod, staging)
selector:
  matchExpressions:
  - key: environment
    operator: In
    values:
    - production
    - staging
  
  - key: tier
    operator: NotIn
    values:
    - backend
```

**Operators:**
- `In`: Value in set
- `NotIn`: Value not in set
- `Exists`: Key exists (any value)
- `DoesNotExist`: Key doesn't exist

---

## 🎯 Use Cases

### 1. Service → Pods

```yaml
# Service selects Pods with matching labels
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web          # Route traffic to Pods with label app=web
    tier: frontend
  ports:
  - port: 80
```

### 2. Deployment → ReplicaSet → Pods

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web        # Manage Pods with app=web
  template:
    metadata:
      labels:
        app: web      # Created Pods will have this label
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

### 3. Node Selector

```yaml
# Schedule Pod on Nodes with specific labels
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    gpu: "true"       # Only Nodes with label gpu=true
    zone: us-west-1
  containers:
  - name: tensorflow
    image: tensorflow/tensorflow:latest-gpu
```

### 4. Query Resources

```bash
# List Pods by environment
kubectl get pods -l environment=production

# List all prod frontend Pods
kubectl get pods -l environment=production,tier=frontend

# List Pods NOT in dev
kubectl get pods -l 'environment notin (dev)'

# List Pods with version label (any value)
kubectl get pods -l version

# List Pods without version label
kubectl get pods -l '!version'
```

---

## 🎨 Recommended Labels

**Standard labels (by K8s community):**

```yaml
metadata:
  labels:
    # Application identity
    app.kubernetes.io/name: mysql           # Component name
    app.kubernetes.io/instance: mysql-prod  # Unique instance
    app.kubernetes.io/version: "5.7.21"     # Version
    app.kubernetes.io/component: database   # Role in architecture
    app.kubernetes.io/part-of: wordpress    # Parent application
    
    # Operational
    app.kubernetes.io/managed-by: helm      # Tool managing this
```

**Example: WordPress + MySQL**

```yaml
# WordPress Deployment
labels:
  app.kubernetes.io/name: wordpress
  app.kubernetes.io/instance: blog
  app.kubernetes.io/version: "5.8"
  app.kubernetes.io/component: frontend
  app.kubernetes.io/part-of: blog-platform
  app.kubernetes.io/managed-by: helm

# MySQL Deployment
labels:
  app.kubernetes.io/name: mysql
  app.kubernetes.io/instance: blog-db
  app.kubernetes.io/version: "5.7"
  app.kubernetes.io/component: database
  app.kubernetes.io/part-of: blog-platform
  app.kubernetes.io/managed-by: helm
```

---

## 📊 Annotations vs Labels

| Feature | Labels | Annotations |
|---------|--------|-------------|
| **Purpose** | Identify & select objects | Attach metadata |
| **Selectable** | ✅ Yes | ❌ No |
| **Size limit** | 63 chars | 256 KB |
| **Use case** | Grouping, querying | Descriptions, configs |

**Annotations example:**

```yaml
metadata:
  annotations:
    description: "Production web server"
    contact: "team-platform@company.com"
    last-updated: "2024-01-15"
    deployment-strategy: "blue-green"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
```

---

## 💡 Best Practices

### ✅ DO

1. **Consistent naming:** Use same label keys across resources
2. **Meaningful values:** `environment: prod` not `env: p`
3. **Standard labels:** Use `app.kubernetes.io/*` prefix
4. **Version labels:** Track versions
5. **Owner labels:** `team: platform` for accountability

### ❌ DON'T

1. **Too many labels:** Keep it simple, 5-10 labels max
2. **Sensitive data:** Don't put secrets in labels (visible to all)
3. **Duplicate info:** Don't repeat what's in name
4. **Dynamic values:** Avoid frequently changing labels

---

## 🎓 Key Takeaways

1. **Labels:** Key-value pairs for organizing objects
2. **Selectors:** Query objects by labels
3. **Service → Pods:** Service uses selector to find Pods
4. **Deployment → Pods:** Deployment manages Pods via labels
5. **Node selector:** Schedule Pods on specific Nodes
6. **Annotations:** Non-selectable metadata
7. **Best practice:** Use standard `app.kubernetes.io/*` labels

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. Labels dùng để làm gì?
2. Sự khác biệt giữa Labels và Annotations?
3. Service tìm Pods bằng cách nào?
4. Node Selector hoạt động như thế nào?
5. Recommended label keys là gì?

---

## 🚀 Tiếp Theo

Bạn đã hoàn thành **Phần 3: Core Concepts**! 🎉

👉 [**Phần 4: Workloads - Quản Lý Ứng Dụng**](../04-workloads/README.md)

---

[⬅️ 3.3. Namespace](./03-namespaces.md) | [⬆️ Phần 3](./README.md) | [🏠 Mục Lục Chính](../README.md)


