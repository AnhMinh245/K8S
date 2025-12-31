# 7.3. StorageClass - Dynamic Provisioning

> Tự động cấp phát storage theo yêu cầu, không cần admin can thiệp

---

## 📖 Mục Lục

1. [Vấn đề với Static Provisioning](#-vấn-đề-với-static-provisioning)
2. [StorageClass là gì?](#-storageclass-là-gì)
3. [Workflow Dynamic Provisioning](#-workflow-dynamic-provisioning)
4. [StorageClass Anatomy](#-storageclass-anatomy)
5. [Cloud Provider Examples](#-cloud-provider-examples)
6. [VolumeBindingMode](#-volumebindingmode)
7. [ReclaimPolicy](#-reclaimpolicy)
8. [Default StorageClass](#-default-storageclass)
9. [Hands-on Labs](#-hands-on-labs)
10. [Troubleshooting](#-troubleshooting)
11. [Best Practices](#-best-practices)

---

## 🔴 Vấn đề với Static Provisioning

### Kịch bản thực tế

**❌ Cách cũ (Static - Thủ công):**

```
Nhà phát triển: "Anh cho em 100GB storage với!"
Admin (lúc 2h sáng): "Ok đợi anh tạo PV..."

1. Admin SSH vào hệ thống
2. Admin tạo disk trên cloud provider (AWS/GCP/Azure)
3. Admin tạo PV trong K8s
4. Admin gửi PV name cho dev
5. Dev tạo PVC bind vào PV đó

⏰ Mất 30-60 phút (nếu admin thức!)
🤦 Scale 100 apps → Admin phải tạo 100 PV!
```

**Vấn đề:**
- ⏰ **Slow:** Phải chờ admin
- 🔥 **Bottleneck:** Admin become single point of failure
- ❌ **Error-prone:** Nhầm lẫn size, region, type
- 💸 **Lãng phí:** Tạo trước → không dùng → vẫn tốn tiền

---

## 🎯 StorageClass là gì?

### Định nghĩa

**StorageClass** là một **template** định nghĩa **cách tạo storage tự động**.

**Ví dụ thực tế:**
```
StorageClass = Form đặt pizza online 📝

- Size: 100GB → "Size L"
- Type: SSD → "Extra cheese"
- Encrypted: true → "Gluten-free"

Submit form (PVC) → Pizza (PV) tự động được làm! 🍕
```

### ✅ Cách mới (Dynamic - Tự động)

```
Nhà phát triển: "Anh cho em 100GB storage với!"
K8s: "OK, 5 phút nữa xong!"

1. Dev tạo PVC với storageClassName
2. K8s gọi cloud provider API
3. Cloud provider tạo disk
4. K8s tạo PV tự động
5. K8s bind PVC vào PV
6. Done! ✅

⏰ Hoàn toàn tự động trong 1-5 phút!
🚀 Scale 100 apps → 100 PV tự động tạo!
```

---

## 🔄 Workflow Dynamic Provisioning

### Flow chi tiết

```
┌─────────────────────────────────────────────────────────────┐
│                   DYNAMIC PROVISIONING                       │
└─────────────────────────────────────────────────────────────┘

1️⃣  Developer tạo PVC
    ├── storageClassName: fast-ssd
    ├── storage: 100Gi
    └── accessMode: ReadWriteOnce

        ↓

2️⃣  PV Controller phát hiện PVC mới
    ├── Check: Có PV nào phù hợp không? → KHÔNG!
    ├── Check: PVC có storageClassName? → CÓ: fast-ssd
    └── Action: Trigger dynamic provisioning

        ↓

3️⃣  External Provisioner (CSI Driver)
    ├── Đọc StorageClass "fast-ssd"
    ├── Lấy parameters: type=gp3, encrypted=true
    └── Gọi Cloud Provider API

        ↓

4️⃣  Cloud Provider (AWS/GCP/Azure)
    ├── Tạo disk: 100GB, gp3, encrypted
    ├── Attach disk vào cluster
    └── Return disk ID

        ↓

5️⃣  K8s tạo PV tự động
    ├── Create PV với disk ID
    ├── Set capacity: 100Gi
    └── Set claimRef: my-pvc

        ↓

6️⃣  Bind PVC → PV
    └── Status: Bound ✅

        ↓

7️⃣  Pod có thể mount PVC ngay!
```

### Timeline

```
0s    - PVC created
0s    - PV Controller detects
1s    - External Provisioner triggered
2-30s - Cloud API creates disk (AWS: ~10s, GCP: ~15s)
31s   - PV created
32s   - PVC Bound
33s   - Pod can mount
```

---

## 📝 StorageClass Anatomy

### Full Example với giải thích

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd                      # ← Tên để PVC reference
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"

# PROVISIONER - Component tạo storage
provisioner: ebs.csi.aws.com          # ← CSI driver của AWS EBS
# Các provisioner khác:
# - pd.csi.storage.gke.io (GCP)
# - disk.csi.azure.com (Azure)
# - csi.vsphere.vmware.com (vSphere)

# PARAMETERS - Truyền vào cloud provider
parameters:
  type: gp3                           # ← Loại disk EBS
  # gp3: General Purpose SSD (mới, rẻ hơn gp2)
  # io2: High IOPS SSD (database, critical apps)
  # st1: Throughput HDD (big data, logs)
  
  iopsPerGB: "10"                     # ← IOPS (Input/Output per second)
  throughput: "125"                   # ← MB/s throughput
  encrypted: "true"                   # ← Mã hóa disk
  kmsKeyId: "arn:aws:kms:..."         # ← Key để encrypt
  
  fsType: ext4                        # ← Filesystem type
  # ext4: Linux default
  # xfs: High performance, large files
  
# RECLAIM POLICY - Khi xóa PVC thì PV sao?
reclaimPolicy: Delete                 # ← Xóa PV + disk khi xóa PVC
# Retain: Giữ PV + disk (manual cleanup)
# Delete: Xóa PV + disk tự động (tiết kiệm tiền!)

# VOLUME BINDING MODE
volumeBindingMode: WaitForFirstConsumer
# WaitForFirstConsumer: Chờ Pod schedule mới tạo PV (tránh cross-AZ)
# Immediate: Tạo PV ngay khi PVC created

# ALLOW VOLUME EXPANSION
allowVolumeExpansion: true            # ← Cho phép tăng size sau (100GB → 200GB)

# MOUNT OPTIONS
mountOptions:
  - debug                             # ← Debug mount issues
  - noatime                           # ← Không update access time (faster)
```

### Parameters cho từng Cloud Provider

**AWS EBS:**
```yaml
parameters:
  type: gp3                    # gp3, gp2, io1, io2, st1, sc1
  iopsPerGB: "10"              # IOPS per GB (io1/io2)
  throughput: "125"            # MB/s (gp3)
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:..."
```

**GCP Persistent Disk:**
```yaml
parameters:
  type: pd-ssd                 # pd-standard, pd-ssd, pd-balanced
  replication-type: none       # none, regional-pd (HA)
  disk-encryption-kms-key: "projects/..."
```

**Azure Disk:**
```yaml
parameters:
  skuName: Premium_LRS         # Premium_LRS, Standard_LRS, StandardSSD_LRS
  location: eastus
  resourceGroup: my-rg
```

---

## ☁️ Cloud Provider Examples

### 1. AWS EBS (Elastic Block Store)

#### Standard SSD (General Purpose)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iopsPerGB: "10"
  throughput: "125"
  encrypted: "true"
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

**Use case:**
- ✅ Web apps, databases, development
- 💰 Cost: $0.08/GB/month
- ⚡ Performance: Up to 16,000 IOPS

#### High Performance SSD (Database)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs-io2
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iopsPerGB: "50"
  encrypted: "true"
allowVolumeExpansion: true
reclaimPolicy: Retain  # ← Giữ lại data khi xóa PVC!
volumeBindingMode: WaitForFirstConsumer
```

**Use case:**
- ✅ Production databases (PostgreSQL, MySQL)
- 💰 Cost: $0.125/GB/month + $0.065/IOPS
- ⚡ Performance: Up to 64,000 IOPS, 99.999% durability

### 2. GCP Persistent Disk

#### Standard SSD
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-pd-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: none
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

#### Regional PD (High Availability)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-pd-regional
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd  # ← Replicate across 2 zones!
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate  # ← Regional → immediate OK
```

**Use case:**
- ✅ HA databases (replicated across zones)
- 💰 Cost: 2x standard price
- ⚡ Survive zone failure!

### 3. Azure Disk

#### Premium SSD
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-disk-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  cachingMode: ReadOnly
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### 4. Local Storage (Bare Metal / On-premise)

#### NFS Provisioner
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.example.com
  share: /export/volumes
mountOptions:
  - hard
  - nfsvers=4.1
reclaimPolicy: Retain
```

#### Longhorn (Cloud-native distributed storage)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  fromBackup: ""
allowVolumeExpansion: true
reclaimPolicy: Delete
```

---

## 🔀 VolumeBindingMode

### Immediate vs WaitForFirstConsumer

**Immediate:**
```
PVC created
    ↓ (ngay lập tức)
PV created in Zone A
    ↓
Pod scheduled to Zone B
    ↓
❌ ERROR! PV ở Zone A, Pod ở Zone B → không attach được!
```

**WaitForFirstConsumer:**
```
PVC created
    ↓
Pod scheduled to Zone B
    ↓ (chờ xem Pod ở đâu)
PV created in Zone B (same zone as Pod!)
    ↓
✅ SUCCESS! Cùng Zone → attach OK!
```

### Example

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: wait-for-pod
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer  # ← Recommended!
```

**Timeline:**
```
0s    - PVC created (Status: Pending)
10s   - Pod created, scheduler assigns to node-2 (Zone B)
11s   - PV created in Zone B
12s   - PVC Bound
13s   - Pod Running
```

---

## 🗑️ ReclaimPolicy

### Delete vs Retain

**Delete (Default):**
```
PVC deleted
    ↓
PV deleted
    ↓
Cloud disk deleted
    ↓
💸 Không tốn tiền nữa!
⚠️  Data mất hẳn!
```

**Retain:**
```
PVC deleted
    ↓
PV still exists (Status: Released)
    ↓
Cloud disk still exists
    ↓
💰 Vẫn tốn tiền!
✅ Data vẫn còn (có thể recover)
```

### Use cases

**Delete:**
- ✅ Development/Testing
- ✅ Cache/Temporary data
- ✅ Stateless apps với persistent config

**Retain:**
- ✅ Production databases
- ✅ Critical data (compliance, audit)
- ✅ Data cần backup trước khi xóa

### Example

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: production-db
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  encrypted: "true"
reclaimPolicy: Retain  # ← Giữ lại data!
```

---

## 🎯 Default StorageClass

### Tại sao cần Default?

**Không có default:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  # storageClassName: ???  ← MUST specify!
```

**Có default:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  # storageClassName not specified → use default! ✅
```

### Set default StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # ← Make default!
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
```

### Check default

```bash
kubectl get storageclass
```

Output:
```
NAME                PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
fast-ssd            ebs.csi.aws.com   Delete          WaitForFirstConsumer   true
standard (default)  ebs.csi.aws.com   Delete          Immediate              true
```

### Override default

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: fast-ssd  # ← Override default!
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

---

## 🧪 Hands-on Labs

### Lab 1: Setup StorageClass (AWS EKS)

**Step 1: Install AWS EBS CSI Driver**
```bash
# Add IAM policy cho EBS CSI Driver
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

# Install EBS CSI Driver
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```

**Step 2: Create StorageClass**
```yaml
# storageclass-gp3.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```bash
kubectl apply -f storageclass-gp3.yaml
kubectl get storageclass
```

**Step 3: Create PVC**
```yaml
# pvc-dynamic.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: ebs-gp3  # ← Use StorageClass!
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f pvc-dynamic.yaml

# Watch PVC creation
kubectl get pvc dynamic-pvc -w
```

**Expected:**
```
NAME          STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
dynamic-pvc   Pending                                      ebs-gp3
```
↓ (sau khi Pod schedule)
```
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
dynamic-pvc   Bound    pvc-12345678-1234-1234-1234-123456789012   10Gi       RWO            ebs-gp3
```

**Step 4: Verify on AWS Console**
```bash
# Get volume ID
kubectl get pv

# Check AWS EBS volumes
aws ec2 describe-volumes --filters "Name=tag:kubernetes.io/cluster/my-cluster,Values=owned"
```

### Lab 2: Test VolumeBindingMode

**Immediate Mode:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: immediate-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: Immediate  # ← Create PV ngay!
```

```bash
kubectl apply -f immediate-sc.yaml

# Create PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: immediate-pvc
spec:
  storageClassName: immediate-sc
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOF

# PV created immediately!
kubectl get pv
```

**WaitForFirstConsumer Mode:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: wait-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer  # ← Wait for Pod!
```

```bash
kubectl apply -f wait-sc.yaml

# Create PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wait-pvc
spec:
  storageClassName: wait-sc
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOF

# PVC Pending (no PV yet!)
kubectl get pvc wait-pvc
# NAME       STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
# wait-pvc   Pending                                      wait-sc

# Create Pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: wait-pvc
EOF

# Now PV created in same zone as Pod!
kubectl get pvc wait-pvc
# NAME       STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS
# wait-pvc   Bound    pvc-xxxxx   5Gi        RWO            wait-sc
```

### Lab 3: Volume Expansion

```yaml
# Create StorageClass với allowVolumeExpansion
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
allowVolumeExpansion: true  # ← Cho phép tăng size!
volumeBindingMode: WaitForFirstConsumer
```

```bash
# Create PVC 10Gi
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: expand-pvc
spec:
  storageClassName: expandable
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF

# Create Pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: expand-pvc
EOF

# Wait for Bound
kubectl wait --for=condition=ready pod/app-pod

# Tăng size 10Gi → 20Gi
kubectl patch pvc expand-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Verify
kubectl get pvc expand-pvc -w
```

**Expected:**
```
NAME         STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS
expand-pvc   Bound    pvc-xxxxx   10Gi       RWO            expandable
expand-pvc   Bound    pvc-xxxxx   20Gi       RWO            expandable  ← Updated!
```

---

## 🔧 Troubleshooting

### Issue 1: PVC stuck in Pending

**Symptoms:**
```bash
kubectl get pvc
# NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
# my-pvc    Pending                                      fast-ssd
```

**Debug:**
```bash
kubectl describe pvc my-pvc
```

**Common causes:**

**A. StorageClass không tồn tại**
```
Events:
  Warning  ProvisioningFailed  storageclass.storage.k8s.io "fast-ssd" not found
```

Fix:
```bash
kubectl get storageclass
# Check tên có đúng không?
```

**B. CSI Driver chưa cài**
```
Events:
  Warning  ProvisioningFailed  waiting for a volume to be created, either by external provisioner "ebs.csi.aws.com" or manually created by system administrator
```

Fix:
```bash
# Check CSI Driver pods
kubectl get pods -n kube-system | grep csi

# Install CSI Driver
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```

**C. VolumeBindingMode = WaitForFirstConsumer (chưa có Pod)**
```
Events:
  Normal  WaitForFirstConsumer  waiting for first consumer to be created before binding
```

Fix: Tạo Pod sử dụng PVC!

**D. Insufficient IAM permissions (AWS)**
```
Events:
  Warning  ProvisioningFailed  failed to provision volume with StorageClass "ebs-gp3": rpc error: code = Internal desc = Could not create volume: UnauthorizedOperation: You are not authorized to perform this operation.
```

Fix:
```bash
# Add IAM policy cho EBS CSI Driver
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

### Issue 2: Volume Expansion Failed

**Symptoms:**
```bash
kubectl get pvc
# NAME      STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS
# my-pvc    Bound    pvc-xxxxx   10Gi       RWO            standard
# (expected 20Gi but still 10Gi!)
```

**Debug:**
```bash
kubectl describe pvc my-pvc
```

**Causes:**

**A. allowVolumeExpansion = false**
```
Events:
  Warning  VolumeResizeFailed  volume expansion not allowed for StorageClass "standard"
```

Fix:
```bash
kubectl patch storageclass standard -p '{"allowVolumeExpansion":true}'
```

**B. Filesystem resize requires Pod restart**
```
Events:
  Normal  FileSystemResizeRequired  waiting for user to (re-)start a pod to finish file system resize of volume on node
```

Fix:
```bash
# Restart Pod
kubectl delete pod my-pod
# Pod recreates, mounts PVC, filesystem resizes automatically!
```

### Issue 3: ReclaimPolicy Retain - Can't reuse PV

**Symptoms:**
```bash
# Deleted PVC, PV still exists
kubectl get pv
# NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM
# pvc-xxxxx   10Gi       RWO            Retain           Released   default/my-pvc
```

**Fix để reuse PV:**
```bash
# 1. Edit PV, remove claimRef
kubectl patch pv pvc-xxxxx -p '{"spec":{"claimRef":null}}'

# 2. PV Available lại
kubectl get pv pvc-xxxxx
# STATUS: Available

# 3. Tạo PVC mới với same size/accessMode
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: reuse-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  volumeName: pvc-xxxxx  # ← Bind to specific PV!
EOF
```

---

## 💡 Best Practices

### 1. Sử dụng WaitForFirstConsumer

```yaml
volumeBindingMode: WaitForFirstConsumer  # ← Always!
```

**Why?**
- ✅ Tránh cross-AZ issues
- ✅ PV và Pod cùng Zone → lower latency, cheaper data transfer
- ✅ Especially important cho multi-AZ clusters

### 2. Enable Volume Expansion

```yaml
allowVolumeExpansion: true  # ← Always enable!
```

**Why?**
- ✅ Có thể tăng size sau mà không cần recreate
- ✅ Downtime-free scaling
- ✅ No data migration needed

### 3. Choose Right ReclaimPolicy

**Development/Testing:**
```yaml
reclaimPolicy: Delete  # ← Tự động cleanup, tiết kiệm tiền
```

**Production:**
```yaml
reclaimPolicy: Retain  # ← Giữ lại data, manual cleanup
```

### 4. Create Multiple StorageClasses

```yaml
# Fast SSD cho databases
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iopsPerGB: "50"
reclaimPolicy: Retain  # ← Production data!

# Standard cho apps
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete

# Cheap HDD cho logs/backups
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cheap-hdd
provisioner: ebs.csi.aws.com
parameters:
  type: st1  # ← Throughput HDD
reclaimPolicy: Delete
```

**Use cases:**
```yaml
# Database
storageClassName: fast-ssd

# Web app
storageClassName: standard  # ← hoặc không specify (dùng default)

# Log aggregator
storageClassName: cheap-hdd
```

### 5. Enable Encryption

```yaml
parameters:
  encrypted: "true"  # ← Always encrypt!
  kmsKeyId: "arn:aws:kms:..."  # ← Use custom KMS key
```

**Why?**
- ✅ Security compliance (GDPR, HIPAA)
- ✅ No performance penalty (encryption ở hardware level)
- ✅ Same cost

### 6. Monitor Storage Costs

```bash
# List all PVs với size
kubectl get pv -o custom-columns=NAME:.metadata.name,SIZE:.spec.capacity.storage,STORAGECLASS:.spec.storageClassName,STATUS:.status.phase

# Find unused PVs (Released/Available)
kubectl get pv --field-selector=status.phase=Released

# Delete unused PVs
kubectl delete pv $(kubectl get pv --field-selector=status.phase=Released -o jsonpath='{.items[*].metadata.name}')
```

### 7. Use Labels for Organization

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: prod-database
  labels:
    environment: production
    tier: database
    cost-center: engineering
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  encrypted: "true"
```

### 8. Set Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: dev-team
spec:
  hard:
    requests.storage: 100Gi  # ← Max 100GB total storage
    persistentvolumeclaims: "10"  # ← Max 10 PVCs
```

---

## 🎓 Key Takeaways

### Concepts

1. **StorageClass:** Template cho dynamic provisioning
2. **Provisioner:** CSI Driver tạo storage (ebs.csi.aws.com, pd.csi.storage.gke.io)
3. **Parameters:** Truyền vào cloud provider (type, IOPS, encryption)
4. **ReclaimPolicy:** 
   - `Delete`: Xóa PV + disk khi xóa PVC (development)
   - `Retain`: Giữ lại data (production)
5. **VolumeBindingMode:**
   - `Immediate`: Tạo PV ngay (có thể cross-AZ issues)
   - `WaitForFirstConsumer`: Chờ Pod schedule (recommended!)
6. **allowVolumeExpansion:** Cho phép tăng size sau

### Workflow

```
PVC created với storageClassName
    ↓
PV Controller trigger provisioning
    ↓
External Provisioner gọi cloud API
    ↓
Cloud provider tạo disk
    ↓
K8s tạo PV tự động
    ↓
Bind PVC → PV
    ↓
Pod mount PVC
```

### Commands

```bash
# List StorageClasses
kubectl get storageclass
kubectl get sc

# Describe
kubectl describe sc fast-ssd

# Create
kubectl apply -f storageclass.yaml

# Set default
kubectl patch storageclass fast-ssd -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Enable expansion
kubectl patch storageclass standard -p '{"allowVolumeExpansion":true}'

# Delete
kubectl delete sc old-storage
```

### Best Practices Summary

- ✅ Use `WaitForFirstConsumer` (avoid cross-AZ)
- ✅ Enable `allowVolumeExpansion`
- ✅ Use `Delete` for dev, `Retain` for prod
- ✅ Enable encryption
- ✅ Create multiple StorageClasses (fast/standard/cheap)
- ✅ Monitor storage costs
- ✅ Set resource quotas

---

## 🧐 Self-Check

1. **Static provisioning có vấn đề gì?**
   <details>
   <summary>Xem đáp án</summary>
   
   - Slow (phải chờ admin tạo PV)
   - Bottleneck (admin thành single point of failure)
   - Error-prone (dễ nhầm lẫn size, type, region)
   - Lãng phí (tạo trước không dùng vẫn tốn tiền)
   </details>

2. **Sự khác biệt giữa `Immediate` và `WaitForFirstConsumer`?**
   <details>
   <summary>Xem đáp án</summary>
   
   - **Immediate:** Tạo PV ngay khi PVC created, có thể gây cross-AZ issues
   - **WaitForFirstConsumer:** Chờ Pod schedule rồi mới tạo PV ở cùng Zone với Pod, recommended!
   </details>

3. **Khi nào dùng `reclaimPolicy: Delete` vs `Retain`?**
   <details>
   <summary>Xem đáp án</summary>
   
   - **Delete:** Development/Testing, cache, temporary data (tự động cleanup, tiết kiệm tiền)
   - **Retain:** Production databases, critical data, compliance data (giữ lại data, manual cleanup)
   </details>

4. **PVC stuck in Pending, làm sao debug?**
   <details>
   <summary>Xem đáp án</summary>
   
   ```bash
   kubectl describe pvc <name>
   ```
   
   Check:
   - StorageClass có tồn tại không?
   - CSI Driver đã cài chưa?
   - VolumeBindingMode = WaitForFirstConsumer → cần tạo Pod!
   - IAM permissions (cloud provider)
   </details>

5. **Có thể tăng size PV từ 10Gi → 20Gi không? Cần gì?**
   <details>
   <summary>Xem đáp án</summary>
   
   Có! Cần:
   - StorageClass có `allowVolumeExpansion: true`
   - Cloud provider hỗ trợ volume expansion
   - Restart Pod để filesystem resize
   
   ```bash
   kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
   kubectl delete pod my-pod  # Recreate để resize filesystem
   ```
   </details>

---

**Chúc mừng!** Hoàn thành **Phần 7: Storage** 🎉

Bạn đã học:
- ✅ Persistent Volumes & Claims
- ✅ StorageClass & Dynamic Provisioning
- ✅ Cloud provider integration (AWS/GCP/Azure)
- ✅ Volume management best practices

👉 [**Phần 8: High Availability & Scaling**](../08-high-availability/README.md)

---

[⬅️ 7.2. PV & PVC](./02-persistent-volumes.md) | [⬆️ Phần 7](./README.md) | [➡️ 8. HA](../08-high-availability/README.md) | [🏠 Mục Lục](../README.md)


