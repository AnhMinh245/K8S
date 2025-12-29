# 4.3. StatefulSet - Cho Ứng Dụng Stateful

> StatefulSet cho ứng dụng cần persistent identity và ordered deployment

---

## 🎯 StatefulSet Là Gì?

**StatefulSet** = Workload controller cho stateful applications, đảm bảo:
- **Stable network identity** (tên Pod cố định)
- **Ordered deployment/scaling** (theo thứ tự 0, 1, 2...)
- **Persistent storage** per Pod

---

## 🏢 Ví Dụ Thực Tế

**Deployment = Nhân viên văn phòng**
```
Nhân viên:
  - Tên: Random (John-abc123, Jane-xyz789)
  - Bàn làm việc: Ngồi đâu cũng được
  - Thay thế: Người mới làm việc ngay
  → Stateless
```

**StatefulSet = Nhà khoa học trong lab**
```
Nhà khoa học:
  - Tên: Cố định (scientist-0, scientist-1)
  - Lab: Riêng, có thiết bị cá nhân
  - Thay thế: Cần bàn giao thiết bị, data
  - Thứ tự: Trưởng nhóm (0) setup trước
  → Stateful
```

---

## 🆚 StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---------|------------|-------------|
| **Pod names** | Random (web-abc123) | Predictable (web-0, web-1) |
| **Network identity** | Changes on restart | Stable (web-0.service) |
| **Pod creation** | Parallel (all at once) | Sequential (0 → 1 → 2) |
| **Pod deletion** | Random order | Reverse order (2 → 1 → 0) |
| **Storage** | Shared or ephemeral | Persistent per Pod |
| **Use case** | Stateless apps | Databases, distributed systems |

---

## 📝 StatefulSet YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
    name: mysql
  clusterIP: None  # Headless Service ← Important!
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql  # Headless Service name
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:  # PVC per Pod
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

---

## 🔄 StatefulSet Deployment Flow

### Initial Creation

```
kubectl apply -f statefulset.yaml

Step 1: Create mysql-0
  - Create PVC: data-mysql-0 (10Gi)
  - Wait for PVC bound
  - Create Pod: mysql-0
  - Wait for Pod Running and Ready

Step 2: Create mysql-1
  - Create PVC: data-mysql-1
  - Create Pod: mysql-1
  - Wait for Running and Ready

Step 3: Create mysql-2
  - Create PVC: data-mysql-2
  - Create Pod: mysql-2
  - Wait for Running and Ready

Result:
  mysql-0 (PVC: data-mysql-0)
  mysql-1 (PVC: data-mysql-1)
  mysql-2 (PVC: data-mysql-2)
```

**Key point:** Sequential, ordered, one at a time

---

### Scaling Up

```bash
kubectl scale statefulset mysql --replicas=5

Process:
  mysql-0, mysql-1, mysql-2 already running
  
  Create mysql-3:
    - Create data-mysql-3
    - Create Pod mysql-3
    - Wait for Ready
  
  Create mysql-4:
    - Create data-mysql-4
    - Create Pod mysql-4
    - Wait for Ready
```

---

### Scaling Down

```bash
kubectl scale statefulset mysql --replicas=2

Process (reverse order):
  Step 1: Delete mysql-2
    - Terminate Pod mysql-2
    - PVC data-mysql-2 REMAINS (not deleted!)
  
  Step 2: Delete mysql-1
    - Terminate Pod mysql-1
    - PVC data-mysql-1 remains
  
Result:
  - Pods: mysql-0 (only)
  - PVCs: data-mysql-0, data-mysql-1, data-mysql-2 (all remain)
```

**Important:** PVCs not deleted on scale down!

---

### Pod Replacement

```
Scenario: mysql-1 crashes

StatefulSet Controller:
  1. Detect mysql-1 terminated
  2. Create new Pod: mysql-1 (same name!)
  3. Mount same PVC: data-mysql-1
  4. Pod starts with previous data intact

Result: Same identity, same data ✅
```

---

## 🌐 Network Identity

### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None  # ← Headless (no ClusterIP)
  selector:
    app: mysql
  ports:
  - port: 3306
```

### DNS Names

```
Each Pod gets stable DNS:
  <pod-name>.<service-name>.<namespace>.svc.cluster.local

Examples:
  mysql-0.mysql.default.svc.cluster.local
  mysql-1.mysql.default.svc.cluster.local
  mysql-2.mysql.default.svc.cluster.local

These DNS names NEVER change, even if Pod restarts!
```

### Use Case: Database Cluster

```yaml
MySQL Cluster:
  mysql-0: Master (read/write)
  mysql-1: Slave (read-only)
  mysql-2: Slave (read-only)

Application connects:
  - Write: mysql-0.mysql:3306
  - Read: mysql-1.mysql:3306 or mysql-2.mysql:3306

Even if mysql-0 restarts → Same DNS → No config change needed
```

---

## 💾 Persistent Storage

### VolumeClaimTemplates

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    resources:
      requests:
        storage: 10Gi
```

**What happens:**
```
StatefulSet creates:
  Pod mysql-0 → PVC data-mysql-0 → PV (10Gi disk)
  Pod mysql-1 → PVC data-mysql-1 → PV (10Gi disk)
  Pod mysql-2 → PVC data-mysql-2 → PV (10Gi disk)

Each Pod has dedicated storage!
```

### PVC Lifecycle

```
1. Create StatefulSet → PVCs created
2. Scale up → New PVCs created
3. Scale down → PVCs REMAIN (not deleted)
4. Delete StatefulSet → Pods deleted, PVCs REMAIN

Why? Prevent data loss!

Manual cleanup:
  kubectl delete pvc data-mysql-0 data-mysql-1 data-mysql-2
```

---

## 🎯 Use Cases

### 1. Databases

**MySQL, PostgreSQL, MongoDB**

```yaml
StatefulSet: mysql-cluster
  mysql-0: Primary (write)
  mysql-1: Replica (read)
  mysql-2: Replica (read)

Benefits:
  - Stable identity for master/slave setup
  - Persistent data per instance
  - Ordered startup (master first)
```

---

### 2. Distributed Systems

**Kafka, ZooKeeper, Elasticsearch, Cassandra**

```yaml
StatefulSet: kafka-cluster
  kafka-0: Broker ID 0
  kafka-1: Broker ID 1
  kafka-2: Broker ID 2

Benefits:
  - Stable broker IDs
  - Persistent message logs
  - Ordered cluster formation
```

---

### 3. Key-Value Stores

**Redis Cluster, etcd**

```yaml
StatefulSet: redis-cluster
  redis-0: Master
  redis-1: Slave (replicates from redis-0)
  redis-2: Slave

Benefits:
  - Stable master/slave relationships
  - Persistent data
```

---

## ⚙️ StatefulSet Features

### 1. Pod Management Policy

```yaml
spec:
  podManagementPolicy: OrderedReady  # Default

# OrderedReady: Sequential (0 → 1 → 2)
# Parallel: All at once (like Deployment)
```

**When to use Parallel:**
- Pods don't depend on each other
- Fast startup needed
- Example: Stateless workers with persistent storage

---

### 2. Update Strategies

#### RollingUpdate (Default)

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Update Pods >= partition
```

**Example:**
```bash
# Update image
kubectl set image statefulset/mysql mysql=mysql:8.0

Process (reverse order):
  1. Update mysql-2 → Wait Ready
  2. Update mysql-1 → Wait Ready
  3. Update mysql-0 → Wait Ready
```

**Canary Deployment with partition:**
```yaml
partition: 2

Result:
  - Pods >= 2 updated (mysql-2)
  - Pods < 2 not updated (mysql-0, mysql-1)
  
Test mysql-2, if OK → Set partition: 0 to update all
```

---

#### OnDelete

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

**Behavior:**
```
Change StatefulSet template → Nothing happens
Delete Pod manually → New Pod created with new template

Use case: Manual control over updates
```

---

## 🔧 StatefulSet Operations

### Create

```bash
kubectl apply -f statefulset.yaml

# Watch creation
kubectl get pods -w
# NAME      READY   STATUS
# mysql-0   0/1     Pending
# mysql-0   0/1     ContainerCreating
# mysql-0   1/1     Running
# mysql-1   0/1     Pending
# ...
```

### Scale

```bash
# Scale up
kubectl scale statefulset mysql --replicas=5

# Scale down
kubectl scale statefulset mysql --replicas=1
```

### Update

```bash
# Update image
kubectl set image statefulset/mysql mysql=mysql:8.0

# Check rollout
kubectl rollout status statefulset/mysql
```

### Delete

```bash
# Delete StatefulSet (keep Pods)
kubectl delete statefulset mysql --cascade=orphan

# Delete StatefulSet and Pods (PVCs remain!)
kubectl delete statefulset mysql

# Delete everything including PVCs
kubectl delete statefulset mysql
kubectl delete pvc -l app=mysql
```

---

## 💡 Best Practices

### ✅ DO

1. **Headless Service required**
```yaml
clusterIP: None
```

2. **Readiness probes essential**
```yaml
readinessProbe:
  exec:
    command:
    - mysql
    - -h
    - 127.0.0.1
    - -e
    - "SELECT 1"
  initialDelaySeconds: 30
  periodSeconds: 10
```

3. **Backup PVCs regularly**
```bash
# PVCs persist even after deletion
# Regular backups essential
```

4. **Resource limits**
```yaml
resources:
  requests:
    cpu: 1
    memory: 2Gi
  limits:
    cpu: 2
    memory: 4Gi
```

5. **Ordered updates for databases**
```yaml
podManagementPolicy: OrderedReady
```

---

### ❌ DON'T

1. **No headless Service** → DNS won't work
2. **No readiness probe** → Sequential startup broken
3. **Delete PVCs carelessly** → Data loss!
4. **Use for stateless apps** → Use Deployment instead
5. **Ignore backup** → Data loss on corruption

---

## 🎓 Key Takeaways

1. **StatefulSet:** For stateful apps (databases, etc.)
2. **Stable identity:** Pod names predictable (mysql-0, mysql-1)
3. **Ordered:** Sequential creation/deletion
4. **Persistent storage:** Each Pod has own PVC
5. **Headless Service:** Stable DNS names
6. **PVC lifecycle:** PVCs survive Pod deletion
7. **Use cases:** Databases, distributed systems, key-value stores

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. StatefulSet khác gì với Deployment?
2. Headless Service là gì và tại sao cần?
3. PVC có bị xóa khi scale down không?
4. Thứ tự tạo/xóa Pod trong StatefulSet?
5. Khi nào dùng StatefulSet thay vì Deployment?
6. DNS name của Pod trong StatefulSet như thế nào?

---

[⬅️ 4.2. Deployment](./02-deployment.md) | [➡️ 4.4. DaemonSet](./04-daemonset.md) | [🏠 Mục Lục Chính](../README.md)

