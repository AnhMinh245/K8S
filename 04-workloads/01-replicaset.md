# 4.1. ReplicaSet

> Đảm bảo số lượng Pod mong muốn luôn chạy

---

## 🎯 ReplicaSet Là Gì?

**ReplicaSet** = Controller đảm bảo một số lượng cố định Pod replicas luôn chạy

```
Desired State: 3 Pods
Current State: 2 Pods (1 Pod crashed)

ReplicaSet detects difference
→ Creates 1 new Pod
→ Current State = Desired State ✅
```

---

## 🏢 Ví Dụ Thực Tế

**ReplicaSet = Giám sát ca làm việc**

```
Ca sáng cần 3 nhân viên:

Scenario 1: 1 người ốm
  Current: 2 người
  → Giám sát viên gọi thêm 1 người
  → Total: 3 người ✅

Scenario 2: Có 4 người đến
  Current: 4 người (dư 1)
  → Giám sát viên cho 1 người về
  → Total: 3 người ✅
```

---

## 📝 ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
  labels:
    app: web
spec:
  replicas: 3           # Desired number of Pods
  selector:
    matchLabels:
      app: web          # Select Pods with this label
  template:             # Pod template
    metadata:
      labels:
        app: web        # Pod labels (must match selector)
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

---

## 🔄 ReplicaSet Workflow

```
1. User creates ReplicaSet
   kubectl apply -f replicaset.yaml

2. ReplicaSet Controller watches API Server
   "New ReplicaSet created!"

3. Controller checks current vs desired
   Desired: 3 Pods
   Current: 0 Pods
   → Need to create 3 Pods

4. Controller creates 3 Pods
   Using template from ReplicaSet spec

5. Scheduler assigns Pods to Nodes

6. kubelet on Nodes start containers

7. ReplicaSet continuously monitors
   If Pod deleted → Create new Pod
   If extra Pod → Delete excess Pod
```

---

## 🎯 Use Cases

### 1. High Availability

```yaml
replicas: 3

Benefits:
  • 1 Pod fails → 2 still running (minimal disruption)
  • Automatic recovery
  • Load distributed across 3 Pods
```

### 2. Load Distribution

```yaml
replicas: 5

Traffic distributed across 5 Pods:
  → Better performance
  → No single point of failure
```

### 3. Scaling

```yaml
# Scale up
kubectl scale replicaset web-rs --replicas=10

# Scale down
kubectl scale replicaset web-rs --replicas=2
```

---

## 🔧 ReplicaSet Operations

### Create

```bash
kubectl apply -f replicaset.yaml
```

### List

```bash
kubectl get replicaset
# Or short form
kubectl get rs

# Output:
NAME     DESIRED   CURRENT   READY   AGE
web-rs   3         3         3       5m
```

### Describe

```bash
kubectl describe rs web-rs

# Shows:
# - Replicas status
# - Selector
# - Pod template
# - Events (Pod creation/deletion)
```

### Scale

```bash
# Imperative
kubectl scale rs web-rs --replicas=5

# Declarative (edit YAML and apply)
# Change replicas: 3 → replicas: 5
kubectl apply -f replicaset.yaml
```

### Delete

```bash
# Delete ReplicaSet and all Pods
kubectl delete rs web-rs

# Delete ReplicaSet but keep Pods
kubectl delete rs web-rs --cascade=orphan
```

---

## 🎨 Selector Types

### Equality-Based

```yaml
selector:
  matchLabels:
    app: web
    tier: frontend
    
# Selects Pods with:
# app=web AND tier=frontend
```

### Set-Based

```yaml
selector:
  matchExpressions:
  - key: app
    operator: In
    values:
    - web
    - api
  - key: environment
    operator: NotIn
    values:
    - dev
    
# Selects Pods where:
# app IN (web, api) AND environment NOT IN (dev)
```

---

## ⚠️ ReplicaSet Limitations

### 1. No Rolling Updates

```
Problem:
  Change image: nginx:1.20 → nginx:1.21
  
ReplicaSet behavior:
  • Doesn't automatically update existing Pods
  • Need to manually delete Pods for recreation
  
❌ Not ideal for updates
```

### 2. No Rollback

```
Problem:
  New version has bugs
  
ReplicaSet:
  • No built-in rollback mechanism
  • Need to manually revert
  
❌ Risky for production
```

### 3. No Update History

```
ReplicaSet:
  • No revision history
  • Can't track changes
  
❌ Hard to audit
```

---

## 🆚 ReplicaSet vs Deployment

| Feature | ReplicaSet | Deployment |
|---------|------------|------------|
| **Ensure replica count** | ✅ | ✅ |
| **Self-healing** | ✅ | ✅ |
| **Rolling updates** | ❌ | ✅ |
| **Rollback** | ❌ | ✅ |
| **Update history** | ❌ | ✅ |
| **Use directly** | ❌ Rare | ✅ Common |

**Recommendation:** Use **Deployment** instead of ReplicaSet directly!

---

## 📊 When to Use ReplicaSet

### ✅ Use When:

**Almost NEVER!** Use Deployment instead.

**Exception:** Custom controllers that manage ReplicaSets themselves

### ❌ Don't Use When:

**99% of cases** → Use Deployment

---

## 🔍 ReplicaSet Internals

### Pod Ownership

```bash
# Pods created by ReplicaSet have ownerReferences
kubectl get pod web-rs-abc123 -o yaml

metadata:
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: web-rs
    controller: true
    
# If ReplicaSet deleted → Pods deleted (cascade)
```

### Label Matching

```
ReplicaSet watches Pods with matching labels:
  selector: app=web
  
If Pod label changes:
  • Old label: app=web → app=api
  • ReplicaSet loses control of this Pod
  • Creates new Pod to maintain replica count
```

---

## 💡 Best Practices

### ✅ DO

1. **Use Deployment:** Not ReplicaSet directly
2. **Match labels:** Pod labels must match selector
3. **Unique names:** Avoid label conflicts
4. **Resource limits:** Set CPU/memory limits

### ❌ DON'T

1. **Create ReplicaSet directly:** Use Deployment
2. **Manual Pod creation:** Let ReplicaSet manage
3. **Change Pod labels:** Can break ReplicaSet control
4. **Multiple ReplicaSets with same selector:** Conflict!

---

## 🎓 Key Takeaways

1. **ReplicaSet:** Ensures N Pod replicas running
2. **Self-healing:** Automatic Pod replacement
3. **Selector:** Finds Pods by labels
4. **Template:** Defines Pod spec
5. **Use Deployment:** Instead of ReplicaSet directly
6. **No rolling updates:** Major limitation
7. **Foundation:** Deployment uses ReplicaSet internally

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. ReplicaSet làm gì khi 1 Pod chết?
2. Tại sao không nên dùng ReplicaSet trực tiếp?
3. Sự khác biệt giữa ReplicaSet và Deployment?
4. Selector matching hoạt động như thế nào?
5. Nếu thay đổi Pod template, ReplicaSet có update Pods cũ không?

---

## 🚀 Tiếp Theo

👉 [4.2. Deployment - Workload Phổ Biến Nhất](./02-deployment.md)

Deployment giải quyết tất cả limitations của ReplicaSet!

---

[⬅️ Phần 4: Workloads](./README.md) | [🏠 Mục Lục Chính](../README.md)

