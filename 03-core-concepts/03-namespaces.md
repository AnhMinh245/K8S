# 3.3. Namespaces - Phân Vùng Resources

> Tổ chức và cô lập resources trong Kubernetes

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **Namespaces là gì** và **tại sao cần**
- ✅ Biết **khi nào nên dùng** Namespaces
- ✅ Quản lý **resources across namespaces**
- ✅ Set **resource quotas** và **limits**
- ✅ **Best practices** trong production

---

## 📁 Namespace Là Gì?

### Định Nghĩa

**Namespace** = Virtual cluster trong physical cluster, dùng để chia resources thành các nhóm logic.

### Giải Thích Bằng Ví Dụ

**Namespace giống như các phòng ban trong công ty:**

```
🏢 CÔNG TY (Kubernetes Cluster)
├── 📂 Phòng R&D (namespace: development)
│   ├── Dev team resources
│   ├── Test environments
│   └── Experimental projects
│
├── 📂 Phòng QA (namespace: testing)
│   ├── QA team resources
│   ├── Integration tests
│   └── Performance tests
│
├── 📂 Phòng Operations (namespace: production)
│   ├── Production apps
│   ├── Customer-facing services
│   └── Critical infrastructure
│
└── 📂 Phòng IT (namespace: kube-system)
    ├── K8s system components
    ├── Monitoring tools
    └── Infrastructure services
```

---

## 🤔 TẠI SAO Cần Namespaces?

### Vấn Đề Không Có Namespace

```
Tất cả resources trong 1 namespace (default):
├── dev-frontend-pod
├── dev-backend-pod
├── test-frontend-pod
├── test-backend-pod
├── prod-frontend-pod
├── prod-backend-pod
├── staging-frontend-pod
└── staging-backend-pod

Problems:
❌ Naming conflicts (phải prefix mọi thứ)
❌ Không có resource isolation
❌ Không control quotas per team
❌ RBAC permissions phức tạp
❌ Khó organize và manage
❌ Dev có thể xóa nhầm prod resources!
```

### Giải Pháp: Namespaces

```
Multiple Namespaces:

namespace: development
├── frontend-pod
├── backend-pod
└── database-pod

namespace: testing  
├── frontend-pod (same name OK!)
├── backend-pod
└── database-pod

namespace: production
├── frontend-pod
├── backend-pod
└── database-pod

Benefits:
✓ Name isolation (same names OK in different NS)
✓ Resource quotas per namespace
✓ RBAC per namespace (dev team → dev NS only)
✓ Clear organization
✓ Logical separation
```

---

## 📦 Default Namespaces

### K8s Built-in Namespaces

```bash
# List all namespaces
kubectl get namespaces
# or
kubectl get ns

# Output:
# NAME              STATUS   AGE
# default           Active   10d
# kube-node-lease   Active   10d
# kube-public       Active   10d
# kube-system       Active   10d
```

**1. default**
```
Purpose: Default namespace cho resources
When: Không specify namespace → goes here
Use for: Quick testing, learning
⚠️  NOT for production!

Example:
kubectl run nginx --image=nginx
→ Creates Pod in 'default' namespace
```

**2. kube-system**
```
Purpose: K8s system components
Contains:
├── kube-apiserver
├── kube-scheduler
├── kube-controller-manager
├── kube-proxy
├── coredns
└── etcd

⚠️  DO NOT put your apps here!
⚠️  DO NOT delete this namespace!
```

**3. kube-public**
```
Purpose: Public resources (readable by all users)
Use case: ConfigMaps with public info
Rarely used in practice
```

**4. kube-node-lease**
```
Purpose: Node heartbeat objects (performance)
Lease objects: One per node
Used by: kubelet to send heartbeats
⚠️  System namespace, don't touch!
```

---

## 🔧 Working với Namespaces

### Create Namespace

**Method 1: Command**

```bash
# Create namespace
kubectl create namespace development

# or shorter
kubectl create ns development

# Verify
kubectl get ns

# Output:
# NAME          STATUS   AGE
# development   Active   5s
```

**Method 2: YAML (Recommended)**

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
    team: engineering
```

```bash
# Apply
kubectl apply -f namespace.yaml

# Namespace với labels giúp organize và query
kubectl get ns --show-labels
```

### Deploy Resources to Namespace

**Method 1: Inline namespace**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: development  # ← Specify namespace here
spec:
  containers:
  - name: nginx
    image: nginx
```

**Method 2: kubectl với -n flag**

```bash
# Create Pod trong specific namespace
kubectl run nginx --image=nginx -n development

# Get Pods từ namespace
kubectl get pods -n development

# All operations support -n
kubectl describe pod nginx -n development
kubectl logs nginx -n development
kubectl delete pod nginx -n development
```

**Method 3: Set default namespace**

```bash
# Set default namespace cho context
kubectl config set-context --current --namespace=development

# Now all commands use 'development' by default
kubectl get pods  # Gets pods from development

# Verify current namespace
kubectl config view --minify | grep namespace:

# Switch back to default
kubectl config set-context --current --namespace=default
```

---

## 🏷️ Organizing với Namespaces

### Common Organization Patterns

**Pattern 1: By Environment**

```
Namespaces:
├── dev
├── staging  
├── uat
└── production

Benefits:
✓ Clear separation
✓ Different resource quotas per env
✓ RBAC: Devs → dev, Ops → prod
```

**Pattern 2: By Team**

```
Namespaces:
├── team-frontend
├── team-backend
├── team-data
└── team-platform

Benefits:
✓ Team autonomy
✓ Resource quotas per team
✓ Clear ownership
```

**Pattern 3: By Application/Project**

```
Namespaces:
├── ecommerce-app
├── analytics-platform
├── payment-service
└── notification-system

Benefits:
✓ Application isolation
✓ Multi-tenant setup
✓ Independent lifecycle
```

**Pattern 4: Hybrid (Recommended)**

```
Namespaces:
├── ecommerce-dev
├── ecommerce-staging
├── ecommerce-prod
├── analytics-dev
├── analytics-prod
└── shared-services

Benefits:
✓ App + Env combination
✓ Best isolation
✓ Clear naming
```

---

## 💰 Resource Quotas

### Tại Sao Cần Quotas?

```
Without quotas:
Team A: Creates 100 Pods, uses 80% cluster CPU
Team B: Cannot deploy (no resources!)
❌ Resource starvation

With quotas:
Team A: Max 50 Pods, 40% cluster CPU
Team B: Max 50 Pods, 40% cluster CPU  
✓ Fair sharing
✓ No single team monopolizes cluster
```

### Create ResourceQuota

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    # Compute resources
    requests.cpu: "10"           # Max 10 CPU cores requested
    requests.memory: 20Gi        # Max 20 GB RAM requested
    limits.cpu: "20"             # Max 20 CPU cores limit
    limits.memory: 40Gi          # Max 40 GB RAM limit
    
    # Object counts
    pods: "50"                   # Max 50 Pods
    services: "10"               # Max 10 Services
    persistentvolumeclaims: "20" # Max 20 PVCs
    secrets: "100"               # Max 100 Secrets
    configmaps: "100"            # Max 100 ConfigMaps
```

```bash
# Apply quota
kubectl apply -f resource-quota.yaml

# View quotas
kubectl get resourcequota -n development

# Output:
# NAME        AGE   REQUEST                                                      LIMIT
# dev-quota   5s    pods: 0/50, requests.cpu: 0/10, requests.memory: 0/20Gi...  limits.cpu: 0/20...

# Describe for details
kubectl describe resourcequota dev-quota -n development
```

### Quota Enforcement

```yaml
# Pod without resource requests/limits (WILL FAIL if quota exists!)
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: development
spec:
  containers:
  - name: nginx
    image: nginx
    # ❌ Missing resources!
```

```bash
# Try to create
kubectl apply -f pod.yaml

# Error:
# Error from server (Forbidden): error when creating "pod.yaml": 
# pods "nginx" is forbidden: failed quota: dev-quota: 
# must specify limits.cpu,limits.memory,requests.cpu,requests.memory
```

**Fix: Add resource requests/limits**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: development
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 256Mi
```

---

## 🎚️ LimitRange

### Default Limits cho Containers

**Problem:**
```
Every Pod must specify resources (when quota exists)
→ Tedious to add to every YAML
→ Developers forget
```

**Solution: LimitRange**
```
Set default requests/limits
Applied automatically if not specified
```

```yaml
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
  # Per container defaults
  - type: Container
    default:              # Default limits
      cpu: 500m
      memory: 512Mi
    defaultRequest:       # Default requests  
      cpu: 100m
      memory: 128Mi
    max:                  # Maximum allowed
      cpu: 2000m
      memory: 2Gi
    min:                  # Minimum required
      cpu: 50m
      memory: 64Mi
  
  # Per Pod limits
  - type: Pod
    max:
      cpu: 4000m
      memory: 4Gi
```

```bash
# Apply
kubectl apply -f limit-range.yaml

# View
kubectl get limitrange -n development
kubectl describe limitrange default-limits -n development
```

**Now Pods without resources get defaults:**

```yaml
# Pod without resources specified
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: development
spec:
  containers:
  - name: nginx
    image: nginx
    # No resources specified!
```

```bash
# Create Pod
kubectl apply -f pod.yaml

# Check applied resources
kubectl get pod nginx -n development -o yaml | grep -A 10 resources:

# Output: Defaults applied!
# resources:
#   limits:
#     cpu: 500m
#     memory: 512Mi
#   requests:
#     cpu: 100m
#     memory: 128Mi
```

---

## 🔍 Namespace Scope

### Namespaced vs Cluster-Scoped Resources

**Namespaced (isolated by NS):**
```
✓ Pods
✓ Services
✓ Deployments
✓ ConfigMaps
✓ Secrets
✓ ServiceAccounts
✓ PersistentVolumeClaims
✓ Jobs, CronJobs
✓ Ingress

→ Can have same name in different namespaces
```

**Cluster-Scoped (global):**
```
✓ Nodes
✓ PersistentVolumes
✓ Namespaces
✓ StorageClasses
✓ ClusterRoles
✓ ClusterRoleBindings

→ Must have unique names across cluster
```

**Check resource scope:**

```bash
# List namespaced resources
kubectl api-resources --namespaced=true

# List cluster-scoped resources
kubectl api-resources --namespaced=false
```

---

## 🌐 Cross-Namespace Communication

### Service DNS Names

**Within same namespace:**
```
Service: backend-service
Pod can access: 
  backend-service
  backend-service.default
  backend-service.default.svc.cluster.local
```

**Cross-namespace:**
```
Service: backend-service (in namespace: production)
Pod in 'development' namespace must use:
  backend-service.production
  backend-service.production.svc.cluster.local

Format: <service-name>.<namespace>.svc.cluster.local
```

**Example:**

```yaml
# frontend Pod trong 'development' namespace
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: development
spec:
  containers:
  - name: app
    image: frontend:v1
    env:
    - name: BACKEND_URL
      value: "http://backend-service.production:8080"
      # ↑ Cross-namespace: service.namespace:port
```

---

## 🎓 Kiểm Tra Hiểu Biết

**1. Khi nào nên tạo namespace mới?**
<details>
<summary>Xem đáp án</summary>

Nên tạo namespace khi:
- Different environments (dev, staging, prod)
- Different teams với resource isolation
- Different projects/applications
- Multi-tenancy requirements
- Need resource quotas per group

Không nên:
- Quá nhiều namespaces (hard to manage)
- Cho mỗi microservice riêng (overkill!)
- Thay thế labels (use labels for grouping)
</details>

**2. ResourceQuota vs LimitRange?**
<details>
<summary>Xem đáp án</summary>

**ResourceQuota:**
- Namespace-level limits
- Total resources allowed
- Example: Max 50 Pods, 10 CPU cores total

**LimitRange:**
- Per-container/Pod defaults và constraints
- Default requests/limits if not specified
- Min/Max values

Use together:
- ResourceQuota: Control total
- LimitRange: Control individual + defaults
</details>

**3. Làm sao access Service từ namespace khác?**
<details>
<summary>Xem đáp án</summary>

Use full DNS name:
```
<service-name>.<namespace>.svc.cluster.local

Examples:
- backend-service.production
- database.data-layer.svc.cluster.local
```

Note: Need NetworkPolicy to allow cross-NS traffic if policies exist!
</details>

---

## 💪 Bài Tập Thực Hành

### Bài 1: Create và Use Namespaces

```bash
# 1. Create 3 namespaces
kubectl create ns dev
kubectl create ns staging
kubectl create ns prod

# 2. Create Pods trong mỗi namespace
kubectl run nginx --image=nginx -n dev
kubectl run nginx --image=nginx -n staging
kubectl run nginx --image=nginx -n prod

# Note: Same name "nginx" OK vì different namespaces!

# 3. List Pods per namespace
kubectl get pods -n dev
kubectl get pods -n staging
kubectl get pods -n prod

# 4. List all Pods all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A

# 5. Set default namespace to dev
kubectl config set-context --current --namespace=dev

# 6. Now kubectl get pods shows dev namespace
kubectl get pods

# 7. Cleanup
kubectl delete ns dev staging prod
```

### Bài 2: Resource Quotas

```yaml
# 1. Create namespace with quota
# quota-namespace.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: limited
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: limited
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    pods: "5"
```

```bash
# Apply
kubectl apply -f quota-namespace.yaml

# Check quota
kubectl get resourcequota -n limited
kubectl describe resourcequota compute-quota -n limited
```

```yaml
# 2. Try create Pod without resources (FAIL)
apiVersion: v1
kind: Pod
metadata:
  name: no-resources
  namespace: limited
spec:
  containers:
  - name: nginx
    image: nginx
```

```bash
# This will fail!
kubectl apply -f pod.yaml
# Error: must specify limits.cpu,limits.memory,...
```

```yaml
# 3. Create Pod with resources (SUCCESS)
apiVersion: v1
kind: Pod
metadata:
  name: with-resources
  namespace: limited
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

```bash
# Success!
kubectl apply -f pod-with-resources.yaml

# Check quota usage
kubectl describe resourcequota compute-quota -n limited
# Shows: Used / Hard
```

### Bài 3: Cross-Namespace Service Access

```bash
# 1. Create 2 namespaces
kubectl create ns app
kubectl create ns data

# 2. Create backend Service trong 'data' namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: backend
  namespace: data
  labels:
    app: backend
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: data
spec:
  selector:
    app: backend
  ports:
  - port: 80
EOF

# 3. Create frontend Pod trong 'app' namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: app
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sleep', '3600']
EOF

# 4. Test cross-namespace access
kubectl exec -n app frontend -- wget -qO- backend-service.data
# Should return nginx welcome page!

# 5. Cleanup
kubectl delete ns app data
```

---

## 🎯 Key Takeaways

1. **Namespaces = Logical Isolation**
   - Virtual clusters
   - Resource organization
   - Not physical isolation

2. **Default Namespaces**
   - default: User resources
   - kube-system: K8s components
   - Don't put apps in kube-system!

3. **Resource Quotas**
   - Limit total resources per namespace
   - Prevent resource starvation
   - Must specify resources in Pods

4. **LimitRange**
   - Default requests/limits
   - Min/Max constraints
   - Simplify Pod specs

5. **Cross-Namespace Access**
   - Use full DNS: `service.namespace`
   - NetworkPolicy may restrict
   - ClusterIP Services accessible

---

## 🚀 Tiếp Theo

Namespaces giúp organize! Next: Labels để query và select!

**Next:** [3.4. Labels & Selectors →](./04-labels-selectors.md)

---

[⬅️ 3.2. Pods](./02-pods.md) | [🏠 Mục Lục](../README.md) | [📂 Phần 3: Core Concepts](./README.md) | [➡️ 3.4. Labels & Selectors](./04-labels-selectors.md)
