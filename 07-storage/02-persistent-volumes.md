# 7.2. PersistentVolume & PersistentVolumeClaim - Production Storage

> Abstraction layer cho persistent storage trong Kubernetes

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **PV vs PVC** và vai trò của từng loại
- ✅ Implement **static provisioning** workflow
- ✅ Understand **access modes** và **reclaim policies**
- ✅ **Bind PVC to PV** và use trong Pods
- ✅ **Troubleshoot** storage issues
- ✅ **Best practices** cho production storage

---

## 🔥 Vấn Đề: Volumes Không Đủ Cho Production

### Tại Sao Volumes (Phần 7.1) Không Đủ?

**Problem với direct volumes:**

```yaml
# ❌ Developer phải biết infrastructure details
apiVersion: v1
kind: Pod
spec:
  volumes:
  - name: data
    awsElasticBlockStore:
      volumeID: vol-0123456789abcdef0  # Hardcoded!
      fsType: ext4

Problems:
❌ Developer cần biết cloud provider details
❌ Hardcoded volume IDs (không portable)
❌ Tight coupling giữa app và infrastructure
❌ Không có abstraction layer
❌ Khó manage lifecycle
❌ Không có separation of concerns
```

**Real-World Scenario:**

```
👨‍💻 DEVELOPER:
"Tôi cần 100GB storage cho database"

👨‍🔧 ADMIN:
"OK, tôi tạo EBS volume vol-xyz cho bạn"

👨‍💻 DEVELOPER:
Updates YAML:
  awsElasticBlockStore:
    volumeID: vol-xyz  # Hardcoded!

Problems:
❌ Developer phải update YAML mỗi lần change storage
❌ Move to different cloud → Rewrite all YAMLs
❌ No abstraction
❌ Tight coupling
```

### Solution: PersistentVolume & PersistentVolumeClaim

**Abstraction layer:**

```
┌────────────────────────────────────────────┐
│  DEVELOPER (Application Layer)             │
│                                            │
│  "I need 100GB storage"                   │
│  → Creates PersistentVolumeClaim (PVC)    │
│  → Doesn't care about implementation      │
└────────────────────────────────────────────┘
                    ↕
        ┌─────────────────────┐
        │   KUBERNETES        │
        │   (Abstraction)     │
        │   Binds PVC to PV   │
        └─────────────────────┘
                    ↕
┌────────────────────────────────────────────┐
│  ADMIN (Infrastructure Layer)              │
│                                            │
│  "I have 500GB EBS volume available"     │
│  → Creates PersistentVolume (PV)          │
│  → Handles infrastructure details         │
└────────────────────────────────────────────┘

Benefits:
✓ Separation of concerns
✓ Developer không cần biết infrastructure
✓ Admin manages storage centrally
✓ Portable YAMLs (no hardcoded IDs)
✓ Lifecycle management
```

---

## 📦 PersistentVolume (PV) - Admin's View

### Định Nghĩa

**PersistentVolume (PV)** = Cluster resource đại diện cho storage, provisioned bởi admin hoặc dynamically bởi StorageClass.

**Ai tạo:** Cluster admin (hoặc automatic via StorageClass)

**Scope:** Cluster-wide (không thuộc namespace)

### Complete PV Example

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-database
  labels:
    type: fast-ssd
    environment: production
spec:
  # Capacity
  capacity:
    storage: 100Gi
  
  # Access modes
  accessModes:
  - ReadWriteOnce  # Single node R/W
  
  # Reclaim policy (what happens when PVC deleted)
  persistentVolumeReclaimPolicy: Retain
  
  # Storage class (for binding)
  storageClassName: fast-ssd
  
  # Mount options
  mountOptions:
  - hard
  - nfsvers=4.1
  
  # Node affinity (optional)
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1
          - node-2
  
  # Actual storage backend (choose one)
  # AWS EBS
  awsElasticBlockStore:
    volumeID: vol-0123456789abcdef0
    fsType: ext4
  
  # OR GCE Persistent Disk
  # gcePersistentDisk:
  #   pdName: my-disk
  #   fsType: ext4
  
  # OR Azure Disk
  # azureDisk:
  #   diskName: my-disk
  #   diskURI: /subscriptions/.../my-disk
  
  # OR NFS
  # nfs:
  #   server: nfs-server.example.com
  #   path: /exports/data
```

### PV Lifecycle States

```
┌──────────────┐
│  Available   │  ← PV created, not bound to PVC
└──────┬───────┘
       │
       │ PVC created, K8s binds
       ↓
┌──────────────┐
│    Bound     │  ← PV bound to PVC, in use
└──────┬───────┘
       │
       │ PVC deleted
       ↓
┌──────────────┐
│   Released   │  ← PVC deleted, but data still exists
└──────┬───────┘
       │
       │ Admin cleanup (if Retain policy)
       ↓
┌──────────────┐
│   Available  │  ← Ready for reuse (or Deleted if Delete policy)
└──────────────┘
```

### Hands-On: Create PV

```bash
# Create AWS EBS volume first (outside K8s)
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 100 \
  --volume-type gp3 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=k8s-pv-database}]'

# Output: vol-0123456789abcdef0

# Create PV
cat > pv-database.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-database
  labels:
    type: ssd
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  awsElasticBlockStore:
    volumeID: vol-0123456789abcdef0
    fsType: ext4
EOF

kubectl apply -f pv-database.yaml

# Check PV
kubectl get pv

# Output:
# NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      STORAGECLASS
# pv-database   100Gi      RWO            Retain           Available   manual

# Describe PV
kubectl describe pv pv-database
```

---

## 📝 PersistentVolumeClaim (PVC) - Developer's View

### Định Nghĩa

**PersistentVolumeClaim (PVC)** = Request for storage bởi developer, K8s binds PVC to matching PV.

**Ai tạo:** Developer / Application team

**Scope:** Namespace-scoped

### Complete PVC Example

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
  namespace: production
  labels:
    app: postgres
spec:
  # Access modes (must match PV)
  accessModes:
  - ReadWriteOnce
  
  # Storage class (for matching PV)
  storageClassName: manual
  
  # Resource request
  resources:
    requests:
      storage: 50Gi  # Can be <= PV capacity
  
  # Selector (optional, for specific PV)
  selector:
    matchLabels:
      type: ssd
    matchExpressions:
    - key: environment
      operator: In
      values:
      - production
      - staging
  
  # Volume mode (Filesystem or Block)
  volumeMode: Filesystem
```

### PVC Binding Process

```
STEP 1: Developer creates PVC
  ↓
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: my-pvc
  spec:
    accessModes: [ReadWriteOnce]
    resources:
      requests:
        storage: 50Gi
    storageClassName: fast-ssd

STEP 2: Kubernetes searches for matching PV
  ↓
  Matching criteria:
  ✓ accessModes compatible
  ✓ capacity >= requested (50Gi)
  ✓ storageClassName matches
  ✓ selector matches (if specified)
  ✓ Status = Available

STEP 3: K8s binds PVC to PV
  ↓
  PV Status: Available → Bound
  PVC Status: Pending → Bound
  
  Binding is 1:1 (one PVC to one PV)
  Even if PV has 100Gi và PVC only needs 50Gi,
  entire PV is reserved for that PVC!

STEP 4: Pod can now use PVC
  ↓
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

### Hands-On: Create và Use PVC

```bash
# 1. Create PVC
cat > database-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: manual
EOF

kubectl apply -f database-pvc.yaml

# 2. Check PVC status
kubectl get pvc

# Output:
# NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS
# database-pvc   Bound    pv-database   100Gi      RWO            manual

# PVC bound to pv-database!

# 3. Describe PVC
kubectl describe pvc database-pvc

# Shows:
# - Bound to: pv-database
# - Capacity: 100Gi (entire PV, even though PVC only requested 50Gi)
# - Access Modes: RWO

# 4. Use PVC trong Pod
cat > postgres-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
  - name: postgres
    image: postgres:14
    env:
    - name: POSTGRES_PASSWORD
      value: mysecretpassword
    ports:
    - containerPort: 5432
    volumeMounts:
    - name: postgres-data
      mountPath: /var/lib/postgresql/data
  
  volumes:
  - name: postgres-data
    persistentVolumeClaim:
      claimName: database-pvc  # References PVC
EOF

kubectl apply -f postgres-pod.yaml

# 5. Verify Pod running
kubectl get pods

# 6. Check data persistence
kubectl exec -it postgres -- psql -U postgres -c "CREATE TABLE test (id INT);"
kubectl exec -it postgres -- psql -U postgres -c "INSERT INTO test VALUES (1), (2), (3);"
kubectl exec -it postgres -- psql -U postgres -c "SELECT * FROM test;"

# 7. Delete Pod (data should persist)
kubectl delete pod postgres

# 8. Recreate Pod (same PVC)
kubectl apply -f postgres-pod.yaml

# 9. Verify data still exists!
kubectl exec -it postgres -- psql -U postgres -c "SELECT * FROM test;"
# Output: 1, 2, 3 (data persisted!)
```

---

## 🔑 Access Modes

### Three Access Modes

| Mode | Abbreviation | Description | Supported By |
|------|--------------|-------------|--------------|
| **ReadWriteOnce** | RWO | Mount read-write bởi single node | Most (EBS, GCE PD, Azure Disk) |
| **ReadOnlyMany** | ROX | Mount read-only bởi many nodes | NFS, CephFS, GlusterFS |
| **ReadWriteMany** | RWX | Mount read-write bởi many nodes | NFS, CephFS, GlusterFS |

### Access Mode Details

**ReadWriteOnce (RWO):**

```
Node 1: Pod A (R/W) ✓
Node 2: Pod B (R/W) ✗  ← Can't mount, already mounted on Node 1

Use cases:
- Databases (single writer)
- Block storage (EBS, GCE PD)
- Most common mode
```

**ReadOnlyMany (ROX):**

```
Node 1: Pod A (R) ✓
Node 2: Pod B (R) ✓
Node 3: Pod C (R) ✓

All pods read-only

Use cases:
- Shared configuration files
- Static website assets
- Shared libraries
```

**ReadWriteMany (RWX):**

```
Node 1: Pod A (R/W) ✓
Node 2: Pod B (R/W) ✓
Node 3: Pod C (R/W) ✓

All pods can read AND write

Use cases:
- Shared uploads folder (WordPress)
- Shared logs directory
- Collaborative editing
- Requires network filesystem (NFS, CephFS)
```

### Example: RWX với NFS

```yaml
# PV với NFS (supports RWX)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany  # RWX supported!
  nfs:
    server: nfs-server.example.com
    path: /exports/shared
---
# PVC requesting RWX
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
---
# Multiple Pods using same PVC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3  # 3 Pods, all mount same volume!
  template:
    spec:
      containers:
      - name: app
        image: wordpress
        volumeMounts:
        - name: uploads
          mountPath: /var/www/html/wp-content/uploads
      volumes:
      - name: uploads
        persistentVolumeClaim:
          claimName: shared-pvc  # All 3 Pods share this!
```

---

## 🔄 Reclaim Policies

### Điều Gì Xảy Ra Khi Xóa PVC?

**Three policies:**

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **Retain** | PV remains, data kept, manual cleanup | Production (safe) |
| **Delete** | PV và underlying storage deleted | Development (auto-cleanup) |
| **Recycle** | Data deleted, PV reusable | Deprecated, don't use |

### Retain Policy (Recommended for Production)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-prod
spec:
  persistentVolumeReclaimPolicy: Retain  # Safe!
  # ... rest of spec
```

**Workflow:**

```
1. PVC created → PV bound
   PV Status: Bound
   
2. Pod uses PVC → Data written
   
3. PVC deleted
   PV Status: Bound → Released
   Data: Still exists!
   
4. Admin must manually:
   - Backup data if needed
   - Clean up data
   - Delete PV
   - Recreate PV for reuse
   
Benefits:
✓ Data not accidentally deleted
✓ Admin has control
✓ Can recover data if needed
```

### Delete Policy (Use with Caution)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-dev
spec:
  persistentVolumeReclaimPolicy: Delete  # Auto-delete!
  # ... rest of spec
```

**Workflow:**

```
1. PVC created → PV bound
   
2. Pod uses PVC → Data written
   
3. PVC deleted
   ↓
   PV automatically deleted
   ↓
   Underlying storage deleted (EBS volume, GCE disk, etc.)
   ↓
   Data GONE forever! ⚠️

Use cases:
- Development/testing environments
- Temporary data
- Cost optimization (auto-cleanup)

⚠️ NEVER use Delete policy for production data!
```

### Hands-On: Reclaim Policy

```bash
# 1. Create PV với Retain policy
cat > pv-retain.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-retain
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain  # ← Retain
  storageClassName: manual
  hostPath:
    path: /tmp/pv-retain
    type: DirectoryOrCreate
EOF

kubectl apply -f pv-retain.yaml

# 2. Create PVC
cat > pvc-test.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-test
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: manual
EOF

kubectl apply -f pvc-test.yaml

# 3. Check binding
kubectl get pv,pvc

# PV: Bound
# PVC: Bound

# 4. Create Pod, write data
kubectl run test-pod --image=busybox --command -- sh -c "echo 'important data' > /data/file.txt && sleep 3600" --overrides='
{
  "spec": {
    "volumes": [{
      "name": "data",
      "persistentVolumeClaim": {"claimName": "pvc-test"}
    }],
    "containers": [{
      "name": "test-pod",
      "image": "busybox",
      "command": ["sh", "-c", "echo important-data > /data/file.txt && sleep 3600"],
      "volumeMounts": [{
        "name": "data",
        "mountPath": "/data"
      }]
    }]
  }
}'

# 5. Verify data written
kubectl exec test-pod -- cat /data/file.txt
# Output: important-data

# 6. Delete PVC
kubectl delete pvc pvc-test

# 7. Check PV status
kubectl get pv pv-retain

# Output:
# NAME        CAPACITY   STATUS     CLAIM
# pv-retain   1Gi        Released   default/pvc-test

# PV is Released, not Available!
# Data still exists on disk!

# 8. Check data on Node (if using hostPath)
# SSH to Node
ls -la /tmp/pv-retain/
# file.txt still exists!

# 9. To reuse PV, admin must:
# - Delete PV
kubectl delete pv pv-retain

# - Recreate PV (or clean data và patch PV)
kubectl apply -f pv-retain.yaml

# Now PV is Available again
```

---

## 🎮 Complete Lab: StatefulSet với PVC

### Lab: Deploy PostgreSQL StatefulSet

```yaml
# postgres-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None  # Headless service
  selector:
    app: postgres
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          value: mysecretpassword
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  
  # VolumeClaimTemplate: Auto-create PVC for each Pod
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
      storageClassName: standard  # Use default StorageClass
```

**Run Lab:**

```bash
# 1. Deploy StatefulSet
kubectl apply -f postgres-statefulset.yaml

# 2. Check PVC auto-created
kubectl get pvc

# Output:
# NAME                      STATUS   VOLUME                 CAPACITY
# postgres-data-postgres-0  Bound    pvc-xyz123...          10Gi

# StatefulSet auto-created PVC!

# 3. Check Pod
kubectl get pods

# 4. Connect to database
kubectl exec -it postgres-0 -- psql -U postgres

# 5. Create table và insert data
CREATE TABLE users (id INT, name VARCHAR(50));
INSERT INTO users VALUES (1, 'Alice'), (2, 'Bob');
SELECT * FROM users;

# 6. Exit
\q

# 7. Delete Pod (simulate crash)
kubectl delete pod postgres-0

# StatefulSet recreates Pod

# 8. Wait for Pod running
kubectl wait --for=condition=Ready pod/postgres-0

# 9. Reconnect và verify data persisted
kubectl exec -it postgres-0 -- psql -U postgres -c "SELECT * FROM users;"

# Output:
#  id | name
# ----+-------
#   1 | Alice
#   2 | Bob

# ✓ Data persisted! PVC survived Pod deletion

# 10. Cleanup
kubectl delete statefulset postgres
kubectl delete service postgres

# PVC still exists (not auto-deleted)
kubectl get pvc

# Manual cleanup
kubectl delete pvc postgres-data-postgres-0
```

---

## 🐛 Troubleshooting

### Issue 1: PVC Stuck in Pending

```bash
$ kubectl get pvc
NAME      STATUS    VOLUME   CAPACITY
my-pvc    Pending   

# PVC not binding to PV

# Debug:
$ kubectl describe pvc my-pvc

# Events might show:
# "no persistent volumes available for this claim"

# Reasons:
# 1. No matching PV exists
# 2. All PVs already bound
# 3. Access modes don't match
# 4. StorageClass doesn't match
# 5. Capacity insufficient

# Fix:
# Create matching PV or use StorageClass for dynamic provisioning
```

### Issue 2: Pod Stuck in ContainerCreating

```bash
$ kubectl get pods
NAME    READY   STATUS              RESTARTS   AGE
mypod   0/1     ContainerCreating   0          5m

$ kubectl describe pod mypod

# Events:
# "Unable to attach or mount volumes: timed out waiting for the condition"

# Reason: PVC not bound, or volume attach failed

# Fix:
# 1. Check PVC status: kubectl get pvc
# 2. Check PV status: kubectl get pv
# 3. Check Node events: kubectl get events
```

### Issue 3: Volume Already Attached

```bash
# Error: Volume already attached to different node

# Reason: RWO volume can only attach to one node
# Pod rescheduled to different node before volume detached

# Fix:
# Wait for volume to detach (automatic, may take 1-2 minutes)
# Or force delete old Pod: kubectl delete pod <old-pod> --force --grace-period=0
```

---

## 💡 Best Practices

### ✅ DO

**1. Use StorageClass for dynamic provisioning (Phần 7.3)**
```yaml
# Instead of manual PV creation
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  storageClassName: fast-ssd  # Auto-provision!
  # ... rest
```

**2. Use Retain policy for production**
```yaml
persistentVolumeReclaimPolicy: Retain  # Safe!
```

**3. Set resource limits**
```yaml
resources:
  requests:
    storage: 100Gi
  limits:
    storage: 100Gi  # Prevent expansion
```

**4. Use labels for organization**
```yaml
metadata:
  labels:
    app: postgres
    environment: production
    tier: database
```

### ❌ DON'T

**1. Don't use Delete policy for production**
```yaml
# ❌ DANGEROUS for production:
persistentVolumeReclaimPolicy: Delete
```

**2. Don't hardcode volume IDs**
```yaml
# ❌ BAD:
awsElasticBlockStore:
  volumeID: vol-xyz  # Hardcoded!

# ✅ GOOD: Use StorageClass
```

**3. Don't request more than needed**
```yaml
# ❌ Wasteful:
resources:
  requests:
    storage: 1Ti  # But only use 10Gi

# ✅ Right-size:
resources:
  requests:
    storage: 100Gi
```

---

## 🎯 Key Takeaways

**1. PV vs PVC:**
- **PV:** Admin creates, cluster-scoped, represents actual storage
- **PVC:** Developer creates, namespace-scoped, requests storage
- **Binding:** K8s matches PVC to PV (1:1 relationship)

**2. Access Modes:**
- **RWO:** Single node R/W (most common, EBS, GCE PD)
- **ROX:** Multiple nodes read-only (config, static files)
- **RWX:** Multiple nodes R/W (requires NFS, CephFS)

**3. Reclaim Policies:**
- **Retain:** Data kept, manual cleanup (production)
- **Delete:** Auto-delete (development only)
- **Recycle:** Deprecated

**4. Workflow:**
```
Admin creates PV → Developer creates PVC → K8s binds → Pod uses PVC
```

**5. For Production:**
- Use StorageClass (dynamic provisioning)
- Retain policy
- Right-size capacity
- Monitor usage

---

## 📚 Commands Cheat Sheet

```bash
# PersistentVolumes
kubectl get pv
kubectl describe pv <pv-name>
kubectl delete pv <pv-name>

# PersistentVolumeClaims
kubectl get pvc
kubectl describe pvc <pvc-name>
kubectl delete pvc <pvc-name>

# Check binding
kubectl get pv,pvc

# Events
kubectl get events --field-selector involvedObject.name=<pvc-name>

# Storage usage (requires metrics-server)
kubectl top pods

# Debug Pod with PVC
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- df -h /mountpath
```

---

## 🚀 Tiếp Theo

PV & PVC học xong! Next: StorageClass cho dynamic provisioning!

**Next:** [7.3. StorageClass - Dynamic Provisioning →](./03-storage-classes.md)

Ở phần tiếp theo, bạn sẽ học:
- Automatic PV provisioning
- Cloud provider integration
- No manual PV creation needed!
- Production-ready storage automation

---

[⬅️ 7.1. Volumes](./01-volumes.md) | [🏠 Mục Lục](../README.md) | [📂 Phần 7: Storage](./README.md) | [➡️ 7.3. StorageClass](./03-storage-classes.md)
