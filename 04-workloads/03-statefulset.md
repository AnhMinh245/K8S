# 4.3. StatefulSet - Stateful Applications

> Quản lý ứng dụng với persistent identity và ordered deployment

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **StatefulSet là gì** và **khác Deployment như thế nào**
- ✅ Biết **khi nào dùng** StatefulSet
- ✅ Hiểu **Pod identity** và **stable network**
- ✅ Quản lý **persistent storage** cho stateful apps
- ✅ Deploy **databases, caches** trong Kubernetes
- ✅ **Scaling và updates** stateful applications

---

## 📦 StatefulSet Là Gì?

### Định Nghĩa

**StatefulSet** = Controller cho stateful applications, provides:
- **Stable, unique network identity**
- **Stable, persistent storage**
- **Ordered, graceful deployment và scaling**
- **Ordered, automated rolling updates**

### Giải Thích Bằng Ví Dụ

**Deployment vs StatefulSet:**

```
🏢 CÔNG TY (Cluster)

DEPLOYMENT = Nhân viên thay thế được:
├── Developer 1, 2, 3 (interchangeable)
├── Ai làm cũng được
├── Nghỉ việc? Tuyển người mới ngay
└── Không cần theo dõi history cá nhân

Use cases: Web servers, API servers (stateless)

STATEFULSET = Nhân viên có vai trò cố định:
├── CEO (database-0) - Luôn là người này
├── CTO (database-1) - Luôn là người này
├── CFO (database-2) - Luôn là người này
├── Không thể thay thế random
├── Có sổ sách riêng (persistent storage)
└── Phải theo thứ tự (CEO → CTO → CFO)

Use cases: Databases, caches (stateful)
```

---

## 🤔 TẠI SAO Cần StatefulSet?

### Deployment Limitations cho Stateful Apps

**Problem với Deployment cho databases:**

```bash
# Deployment với 3 PostgreSQL replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:14

# Problems:
❌ Random Pod names: postgres-abc12, postgres-def34, postgres-ghi56
   → Can't identify primary vs replica!

❌ No stable network identity
   → Connections break when Pod restarts (new IP)

❌ No persistent storage guarantee
   → Data lost when Pod dies!

❌ Random order deployment
   → Can't ensure primary starts before replicas

❌ No stable DNS names
   → Can't connect to specific database instance
```

### StatefulSet Solution

```yaml
# StatefulSet với 3 PostgreSQL replicas
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 3
  serviceName: postgres
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:14

# Benefits:
✓ Predictable names: postgres-0, postgres-1, postgres-2
  → postgres-0 = primary, postgres-1,2 = replicas

✓ Stable network identity:
  → postgres-0.postgres.default.svc.cluster.local
  → Always same DNS name, even after restart

✓ Persistent storage per Pod:
  → postgres-0 gets PVC: data-postgres-0
  → postgres-1 gets PVC: data-postgres-1
  → Data survives Pod restarts!

✓ Ordered deployment:
  → postgres-0 starts first
  → postgres-1 starts after postgres-0 ready
  → postgres-2 starts after postgres-1 ready

✓ Stable DNS for each Pod:
  → Can connect to postgres-0 specifically
```

---

## 🏗️ StatefulSet Features

### 1. Stable Pod Identity

**Deployment Pods:**
```
Random names (hash suffix):
├── webapp-7d4b7c9d8f-abc12
├── webapp-7d4b7c9d8f-def34
└── webapp-7d4b7c9d8f-ghi56

Pod restarts → New name:
webapp-7d4b7c9d8f-abc12 (old) → webapp-7d4b7c9d8f-xyz99 (new)
```

**StatefulSet Pods:**
```
Predictable names (ordinal suffix):
├── postgres-0
├── postgres-1
└── postgres-2

Pod restarts → Same name:
postgres-0 (old) → postgres-0 (new)
Always postgres-0!
```

---

### 2. Stable Network Identity

**Headless Service + StatefulSet:**

```yaml
# Headless Service (clusterIP: None)
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None  # Headless!
  selector:
    app: postgres
  ports:
  - port: 5432

# StatefulSet references Service
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres  # Links to Headless Service
  replicas: 3
```

**DNS Names:**
```
Each Pod gets stable DNS:

postgres-0.postgres.default.svc.cluster.local
postgres-1.postgres.default.svc.cluster.local
postgres-2.postgres.default.svc.cluster.local

Even after Pod restart:
postgres-0 dies → recreated → SAME DNS name!

Applications can connect to:
- Specific Pod: postgres-0.postgres
- All Pods: postgres (Service name)
```

---

### 3. Ordered Deployment

**Deployment:** All Pods created simultaneously (parallel)

**StatefulSet:** Sequential creation

```
StatefulSet: postgres, replicas: 3

Step 1: Create postgres-0
├── Create Pod postgres-0
├── Wait until Running AND Ready
└── Only then proceed to Step 2

Step 2: Create postgres-1
├── Create Pod postgres-1
├── Wait until Running AND Ready
└── Only then proceed to Step 3

Step 3: Create postgres-2
├── Create Pod postgres-2
├── Wait until Running AND Ready
└── Deployment complete!

Order guaranteed: 0 → 1 → 2
```

**Why needed?**
```
Database replication setup:
1. postgres-0 (Primary) must start first
2. postgres-1 (Replica) connects to postgres-0
3. postgres-2 (Replica) connects to postgres-0

If parallel: Replicas might start before Primary!
→ Connection failures
→ Setup broken
```

---

### 4. Ordered Scaling

**Scale Up:**
```
Current: postgres-0, postgres-1, postgres-2 (3 Pods)
Scale to: 5 replicas

Order:
1. Create postgres-3, wait Ready
2. Create postgres-4, wait Ready

New Pods added sequentially: 3 → 4
```

**Scale Down:**
```
Current: postgres-0, postgres-1, postgres-2, postgres-3, postgres-4 (5 Pods)
Scale to: 3 replicas

Order:
1. Delete postgres-4, wait Terminated
2. Delete postgres-3, wait Terminated

Pods deleted in REVERSE order: 4 → 3
(Always deletes highest ordinal first)
```

---

### 5. Persistent Storage

**volumeClaimTemplates:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 3
  serviceName: postgres
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:14
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  
  # Persistent storage template
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

**What happens:**
```
StatefulSet creates 3 Pods:

postgres-0:
├── Pod: postgres-0
└── PVC: data-postgres-0 (10Gi)
    └── Bound to PersistentVolume

postgres-1:
├── Pod: postgres-1
└── PVC: data-postgres-1 (10Gi)
    └── Bound to PersistentVolume

postgres-2:
├── Pod: postgres-2
└── PVC: data-postgres-2 (10Gi)
    └── Bound to PersistentVolume

Each Pod has dedicated persistent volume!

Pod restart:
postgres-0 dies → recreated → Reattaches to data-postgres-0
→ Data persists!
```

---

## 📝 StatefulSet YAML

### Complete Example

```yaml
---
# Headless Service (required!)
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: redis
spec:
  clusterIP: None  # Headless
  selector:
    app: redis
  ports:
  - port: 6379
    name: redis

---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  # Service name (must match Headless Service)
  serviceName: redis
  
  # Number of replicas
  replicas: 3
  
  # Selector
  selector:
    matchLabels:
      app: redis
  
  # Pod template
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
          name: redis
        volumeMounts:
        - name: data
          mountPath: /data
        command:
        - redis-server
        - "--appendonly"
        - "yes"
  
  # Persistent volume claim template
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard  # Or your StorageClass
      resources:
        requests:
          storage: 5Gi
```

---

## 🎮 Hands-On: Working với StatefulSets

### Create StatefulSet

```bash
# Apply (creates Headless Service + StatefulSet)
kubectl apply -f statefulset.yaml

# Watch Pods being created (ordered!)
kubectl get pods -w -l app=redis

# Output (sequential creation):
# NAME      READY   STATUS              AGE
# redis-0   0/1     ContainerCreating   2s
# redis-0   1/1     Running             10s
# redis-1   0/1     ContainerCreating   1s   ← Created after redis-0 Ready
# redis-1   1/1     Running             8s
# redis-2   0/1     ContainerCreating   1s   ← Created after redis-1 Ready
# redis-2   1/1     Running             7s

# Check StatefulSet
kubectl get statefulset
# or shorter
kubectl get sts

# Output:
# NAME    READY   AGE
# redis   3/3     1m

# Check Pods
kubectl get pods -l app=redis

# Output (predictable names!):
# NAME      READY   STATUS    RESTARTS   AGE
# redis-0   1/1     Running   0          2m
# redis-1   1/1     Running   0          2m
# redis-2   1/1     Running   0          2m

# Check PVCs (one per Pod!)
kubectl get pvc

# Output:
# NAME           STATUS   VOLUME       CAPACITY   ACCESS MODES
# data-redis-0   Bound    pvc-abc123   5Gi        RWO
# data-redis-1   Bound    pvc-def456   5Gi        RWO
# data-redis-2   Bound    pvc-ghi789   5Gi        RWO
```

---

### Test Stable Network Identity

```bash
# Get Service
kubectl get svc redis

# Output:
# NAME    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
# redis   ClusterIP   None         <none>        6379/TCP
#                     ↑ Headless!

# DNS resolution for each Pod
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup redis-0.redis

# Output:
# Server:    10.96.0.10
# Address:   10.96.0.10:53
# 
# Name:   redis-0.redis.default.svc.cluster.local
# Address: 10.244.1.5

# Test stable DNS
kubectl exec redis-0 -- hostname -f

# Output:
# redis-0.redis.default.svc.cluster.local

# Connect to specific Pod
kubectl run -it --rm redis-client --image=redis --restart=Never -- redis-cli -h redis-0.redis

# Inside redis-cli:
127.0.0.1:6379> SET key1 "Hello from redis-0"
127.0.0.1:6379> GET key1
"Hello from redis-0"
```

---

### Test Persistent Storage

```bash
# Write data to redis-0
kubectl exec -it redis-0 -- redis-cli SET mykey "persistent-data"

# Verify
kubectl exec redis-0 -- redis-cli GET mykey
# Output: "persistent-data"

# Delete Pod (simulate crash)
kubectl delete pod redis-0

# Pod recreated automatically
kubectl get pods -w
# redis-0   0/1     ContainerCreating   2s
# redis-0   1/1     Running             10s

# Data still there! (from PVC)
kubectl exec redis-0 -- redis-cli GET mykey
# Output: "persistent-data"

✓ Data persisted through Pod restart!
```

---

### Scale StatefulSet

**Scale Up:**

```bash
# Scale from 3 to 5
kubectl scale statefulset redis --replicas=5

# Watch (sequential creation!)
kubectl get pods -w -l app=redis

# Output:
# redis-0   1/1     Running   0    10m
# redis-1   1/1     Running   0    10m
# redis-2   1/1     Running   0    10m
# redis-3   0/1     ContainerCreating   2s  ← Created
# redis-3   1/1     Running             10s
# redis-4   0/1     ContainerCreating   1s  ← Created after redis-3 Ready
# redis-4   1/1     Running             9s

# New PVCs created
kubectl get pvc
# data-redis-0, data-redis-1, data-redis-2, data-redis-3, data-redis-4
```

**Scale Down:**

```bash
# Scale from 5 to 3
kubectl scale statefulset redis --replicas=3

# Watch (reverse order deletion!)
kubectl get pods -w -l app=redis

# Output:
# redis-4   1/1     Terminating   0    2m  ← Deleted first (highest)
# redis-4   0/1     Terminating   0    2m
# redis-3   1/1     Terminating   0    2m  ← Deleted second
# redis-3   0/1     Terminating   0    2m
# redis-0   1/1     Running       0    12m ← Kept
# redis-1   1/1     Running       0    12m ← Kept
# redis-2   1/1     Running       0    12m ← Kept

# PVCs NOT deleted! (data preserved)
kubectl get pvc
# data-redis-0, data-redis-1, data-redis-2, data-redis-3, data-redis-4
#                                          ↑ PVCs for redis-3, redis-4 still exist!

# Scale back up to 5
kubectl scale statefulset redis --replicas=5

# redis-3 và redis-4 recreated và reattach to existing PVCs!
# → Data from before scale-down is still there!
```

---

### Update StatefulSet

**Rolling Update (default):**

```yaml
spec:
  updateStrategy:
    type: RollingUpdate  # Default
```

```bash
# Update image
kubectl set image statefulset/redis redis=redis:7.2-alpine

# Pods updated in REVERSE order (2 → 1 → 0)
kubectl get pods -w -l app=redis

# Output:
# redis-2   1/1     Terminating         0    5m  ← Updated first
# redis-2   0/1     ContainerCreating   0    1s
# redis-2   1/1     Running             0    10s
# redis-1   1/1     Terminating         0    5m  ← Updated second
# redis-1   0/1     ContainerCreating   0    1s
# redis-1   1/1     Running             0    9s
# redis-0   1/1     Terminating         0    5m  ← Updated last
# redis-0   0/1     ContainerCreating   0    1s
# redis-0   1/1     Running             0    10s

# Order: Highest ordinal first (2 → 1 → 0)
# Why? Safer for databases (replicas updated before primary)
```

**OnDelete Update:**

```yaml
spec:
  updateStrategy:
    type: OnDelete  # Manual control
```

```bash
# Update StatefulSet
kubectl set image statefulset/redis redis=redis:7.2-alpine

# Pods NOT updated automatically!
kubectl get pods -l app=redis -o jsonpath='{.items[*].spec.containers[*].image}'
# Output: redis:7-alpine redis:7-alpine redis:7-alpine (still old image)

# Manually delete Pods to trigger update
kubectl delete pod redis-2
# redis-2 recreated with new image

kubectl delete pod redis-1
# redis-1 recreated with new image

kubectl delete pod redis-0
# redis-0 recreated with new image

# Now all updated to redis:7.2-alpine
```

---

### Delete StatefulSet

**Option 1: Delete StatefulSet + Pods**

```bash
# Default: Deletes StatefulSet và Pods
kubectl delete statefulset redis

# Pods deleted in reverse order (2 → 1 → 0)
kubectl get pods -w -l app=redis

# PVCs NOT deleted (must delete manually)
kubectl get pvc
# data-redis-0, data-redis-1, data-redis-2 still exist!

# Delete PVCs if needed
kubectl delete pvc data-redis-0 data-redis-1 data-redis-2
```

**Option 2: Delete StatefulSet, keep Pods**

```bash
# Cascade=orphan: Delete StatefulSet, keep Pods
kubectl delete statefulset redis --cascade=orphan

# StatefulSet deleted
kubectl get statefulset
# No resources found

# Pods still running (orphaned)
kubectl get pods -l app=redis
# redis-0, redis-1, redis-2 still Running

# Pods now unmanaged (must delete manually)
```

---

## 🔗 StatefulSet vs Deployment

### Comparison Table

| Feature | Deployment | StatefulSet |
|---------|------------|-------------|
| **Pod names** | Random (hash suffix) | Predictable (ordinal) |
| **Network identity** | Random IPs, changes on restart | Stable DNS per Pod |
| **Storage** | Shared or ephemeral | Persistent per Pod |
| **Deployment order** | Parallel (simultaneous) | Sequential (ordered) |
| **Scaling order** | Random | Ordered (0→1→2 up, 2→1→0 down) |
| **Update order** | Random | Reverse order (2→1→0) |
| **Use cases** | Stateless apps | Stateful apps |
| **Examples** | Web servers, APIs | Databases, caches |

---

## 🎯 When to Use StatefulSet?

### Use StatefulSet For:

```
✓ Databases (PostgreSQL, MySQL, MongoDB)
✓ Distributed caches (Redis, Memcached)
✓ Message queues (Kafka, RabbitMQ)
✓ Distributed coordination (ZooKeeper, etcd)
✓ Applications needing:
  - Stable network identity
  - Persistent storage per instance
  - Ordered deployment/updates
  - Peer discovery (cluster formation)
```

### Use Deployment For:

```
✓ Stateless web applications
✓ REST APIs
✓ Microservices (stateless)
✓ Batch processing workers
✓ Applications where:
  - Pods are interchangeable
  - No persistent state needed
  - Fast scaling required
  - Random deployment OK
```

---

## 🐛 Troubleshooting StatefulSets

### Issue 1: Pod Stuck in Pending

```bash
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
redis-0   0/1     Pending   0          5m

# Pod 0 stuck, Pods 1,2 not created yet

# Describe Pod
$ kubectl describe pod redis-0

# Events might show:
# Warning  FailedScheduling  0/3 nodes are available: persistentvolumeclaim "data-redis-0" not found

# Cause: No PVC or PV available

# Fix:
# 1. Check PVC exists
kubectl get pvc data-redis-0

# 2. Check PV available
kubectl get pv

# 3. Create StorageClass if missing
# 4. Create PV manually if using static provisioning
```

---

### Issue 2: Pods Created Out of Order

```bash
$ kubectl get pods -l app=redis
NAME      READY   STATUS    RESTARTS   AGE
redis-0   1/1     Running   0          2m
redis-1   1/1     Running   0          2m
redis-2   0/1     Pending   0          2m

# redis-1 và redis-2 shouldn't be created until redis-0 Ready!

# Cause: podManagementPolicy: Parallel

# Check StatefulSet
$ kubectl get statefulset redis -o yaml | grep podManagementPolicy
podManagementPolicy: Parallel  ← Found it!

# Fix: Change to OrderedReady (default)
kubectl patch statefulset redis -p '{"spec":{"podManagementPolicy":"OrderedReady"}}'
```

---

### Issue 3: Data Lost After Scale Down/Up

```bash
# Scale down
kubectl scale statefulset redis --replicas=1

# redis-2, redis-1 deleted
# data-redis-1, data-redis-2 PVCs still exist ✓

# BUT: PVCs deleted manually (mistake!)
kubectl delete pvc data-redis-1 data-redis-2

# Scale back up
kubectl scale statefulset redis --replicas=3

# redis-1, redis-2 recreated with NEW PVCs
# → Data from before is LOST!

# Prevention:
# 1. Don't delete PVCs manually unless intentional
# 2. Backup data before scaling down
# 3. Use proper backup solutions
```

---

## 🎓 Kiểm Tra Hiểu Biết

**1. StatefulSet vs Deployment - khi nào dùng gì?**
<details>
<summary>Xem đáp án</summary>

**StatefulSet:**
- Stateful applications
- Need stable identity
- Persistent storage per Pod
- Ordered deployment
- Examples: Databases, caches

**Deployment:**
- Stateless applications
- Pods interchangeable
- No persistent state
- Fast scaling
- Examples: Web servers, APIs

**Rule:** If in doubt → Try Deployment first. Only use StatefulSet if you NEED stateful features.
</details>

**2. Tại sao cần Headless Service?**
<details>
<summary>Xem đáp án</summary>

Headless Service (clusterIP: None) provides:

1. **Stable DNS per Pod:**
   - redis-0.redis.default.svc.cluster.local
   - redis-1.redis.default.svc.cluster.local

2. **Direct Pod access:**
   - No load balancing
   - Connect to specific Pod

3. **Peer discovery:**
   - Pods can find each other
   - Cluster formation (e.g., Redis Cluster)

Without Headless Service: No stable DNS per Pod!
</details>

**3. Scale down StatefulSet 5→3, PVCs bị xóa không?**
<details>
<summary>Xem đáp án</summary>

**NO! PVCs NOT deleted automatically.**

```
Before scale down:
- Pods: redis-0, redis-1, redis-2, redis-3, redis-4
- PVCs: data-redis-0, ..., data-redis-4

After scale down to 3:
- Pods: redis-0, redis-1, redis-2
- PVCs: data-redis-0, data-redis-1, data-redis-2, data-redis-3, data-redis-4
         ↑ All PVCs still exist!

Scale back up to 5:
- redis-3, redis-4 reattach to existing PVCs
- Data preserved!

Must manually delete PVCs if want to remove data.
```
</details>

---

## 💪 Bài Tập Thực Hành

### Bài 1: Deploy Redis StatefulSet

(Use YAML from earlier examples)

```bash
# 1. Apply
kubectl apply -f redis-statefulset.yaml

# 2. Watch ordered creation
kubectl get pods -w -l app=redis

# 3. Test stable DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup redis-0.redis

# 4. Write data
kubectl exec redis-0 -- redis-cli SET test-key "hello"

# 5. Delete Pod
kubectl delete pod redis-0

# 6. Verify data persists
kubectl exec redis-0 -- redis-cli GET test-key
# Should return: "hello"

# 7. Scale up
kubectl scale sts redis --replicas=5

# 8. Scale down
kubectl scale sts redis --replicas=2

# 9. Check PVCs (should have 5, even though only 2 Pods)
kubectl get pvc

# 10. Cleanup
kubectl delete sts redis
kubectl delete svc redis
kubectl delete pvc --all
```

---

## 🎯 Key Takeaways

1. **StatefulSet = Stateful Apps**
   - Stable identity
   - Persistent storage
   - Ordered operations

2. **Key Features**
   - Predictable Pod names (ordinal)
   - Stable DNS per Pod
   - Persistent storage per Pod
   - Sequential deployment/scaling

3. **Headless Service Required**
   - clusterIP: None
   - Enables stable DNS

4. **PVCs Not Auto-Deleted**
   - Scale down → PVCs kept
   - Data preserved
   - Must delete manually

5. **When to Use**
   - Databases: YES
   - Caches: YES
   - Web APIs: NO (use Deployment)

---

## 🚀 Tiếp Theo

StatefulSets nắm rồi! Next: DaemonSets - run on every Node!

**Next:** [4.4. DaemonSet →](./04-daemonset.md)

---

[⬅️ 4.2. Deployment](./02-deployment.md) | [🏠 Mục Lục](../README.md) | [📂 Phần 4: Workloads](./README.md) | [➡️ 4.4. DaemonSet](./04-daemonset.md)
