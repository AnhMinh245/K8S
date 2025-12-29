# 3.3. Namespace - Phân Vùng Resources

> Namespaces chia cluster thành các phòng ban ảo

---

## 🎯 Namespace Là Gì?

**Namespace** = Virtual cluster trong physical cluster, để isolate resources

```
Cluster
├── Namespace: dev
│   ├── Pods, Services, Deployments (team dev)
│   └── Resource quotas
├── Namespace: staging
│   └── ...
├── Namespace: production
│   └── ...
└── Namespace: team-a
    └── ...
```

---

## 🏢 Ví Dụ Thực Tế

**Namespace = Phòng ban trong công ty**

```
Công ty (Cluster):
├── Phòng R&D (dev namespace)
│   • Dự án A, B, C
│   • Budget: $5000
├── Phòng QA (test namespace)
│   • Test environments
│   • Budget: $3000
└── Phòng Production (prod namespace)
    • Live services
    • Budget: $20000
```

---

## 📋 Default Namespaces

```bash
kubectl get namespaces

NAME              STATUS   AGE
default           Active   30d  ← Default namespace
kube-system       Active   30d  ← K8s system components
kube-public       Active   30d  ← Public, readable by all
kube-node-lease   Active   30d  ← Node heartbeats
```

### default
- Namespace mặc định khi không chỉ định
- Dùng cho testing, development

### kube-system
- K8s system components (kube-proxy, CoreDNS, etc.)
- ❌ Không deploy app vào đây

### kube-public
- Resources public, readable by everyone
- Ít dùng

### kube-node-lease
- Node heartbeat objects
- Internal use

---

## 🛠️ Working with Namespaces

### Create Namespace

```bash
# Method 1: kubectl
kubectl create namespace dev

# Method 2: YAML
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: dev
EOF
```

### List Resources in Namespace

```bash
# Pods in namespace "dev"
kubectl get pods -n dev

# All pods in all namespaces
kubectl get pods --all-namespaces
# Or short form
kubectl get pods -A
```

### Set Default Namespace

```bash
# Set context to use "dev" namespace by default
kubectl config set-context --current --namespace=dev

# Verify
kubectl config view --minify | grep namespace
```

### Delete Namespace

```bash
# ⚠️ This deletes ALL resources in namespace!
kubectl delete namespace dev
```

---

## 🎯 Use Cases

### 1. Multi-Environment

```
dev namespace:
  - Development environment
  - Low resources
  - Relaxed security

staging namespace:
  - Pre-production
  - Medium resources
  - Production-like

production namespace:
  - Live system
  - High resources
  - Strict security
```

### 2. Multi-Tenancy

```
team-frontend namespace:
  - Frontend team resources
  - RBAC: Only frontend team access

team-backend namespace:
  - Backend team resources
  - RBAC: Only backend team access
```

### 3. Resource Isolation

```yaml
# Limit resources per namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "10"      # Max 10 CPU cores
    requests.memory: "20Gi" # Max 20 GB RAM
    pods: "50"              # Max 50 Pods
```

---

## 🔒 RBAC with Namespaces

```yaml
# User "alice" can only manage Pods in namespace "dev"
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-manager
  namespace: dev
subjects:
- kind: User
  name: alice
roleRef:
  kind: Role
  name: pod-manager
```

---

## 🌐 Cross-Namespace Communication

**DNS format:**
```
<service-name>.<namespace>.svc.cluster.local

Examples:
  - backend.dev.svc.cluster.local
  - database.production.svc.cluster.local
```

**Within same namespace:**
```bash
# Pod in "dev" namespace calling Service "api" in "dev"
curl http://api:8080
```

**Across namespaces:**
```bash
# Pod in "dev" calling Service "database" in "production"
curl http://database.production:3306
```

---

## 💡 Best Practices

### ✅ DO

1. **Separate environments:** dev, staging, prod
2. **Team isolation:** 1 namespace per team
3. **Resource quotas:** Prevent resource hogging
4. **Network policies:** Restrict cross-namespace traffic
5. **Naming convention:** `<team>-<env>` (e.g., `frontend-prod`)

### ❌ DON'T

1. **Too many namespaces:** Overhead, complexity
2. **Deploy to kube-system:** System only
3. **No quotas:** One team can starve others
4. **Ignore in CI/CD:** Always specify namespace explicitly

---

## 🎓 Key Takeaways

1. **Namespace:** Virtual cluster, isolate resources
2. **Multi-tenancy:** Teams/environments in same cluster
3. **Resource quotas:** Limit CPU/memory per namespace
4. **RBAC:** Control access per namespace
5. **DNS:** `service.namespace.svc.cluster.local`
6. **Default namespaces:** default, kube-system, kube-public
7. **Best practice:** Separate environments and teams

---

[⬅️ 3.2. Pods](./02-pods.md) | [⬆️ Phần 3](./README.md) | [➡️ 3.4. Labels & Selectors](./04-labels-selectors.md)

