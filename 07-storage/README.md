# 📘 Phần 7: Storage - Persistent Data

> Volumes, PersistentVolumes, và data persistence

---

## 🎯 Mục Tiêu

✅ **Persistent storage** cho stateful apps  
✅ **Volumes, PVs, PVCs** concepts  
✅ **StorageClasses** và dynamic provisioning  
✅ **Best practices** cho production storage  

---

## 📚 Nội Dung

### Volumes - Temporary Storage
**Lifecycle:** Tied to Pod (ephemeral)
**Use:** Shared data between containers, temporary cache

### PersistentVolumes (PV) - Long-term Storage  
**Lifecycle:** Independent of Pods
**Use:** Databases, file storage, persistent data

### PersistentVolumeClaims (PVC) - Storage Requests
**Purpose:** Request storage from PVs
**Pattern:** PVC binds to PV, Pod uses PVC

### StorageClasses - Dynamic Provisioning
**Purpose:** Automatically create PVs on demand
**Cloud:** AWS EBS, GCP PD, Azure Disk

---

## 🎯 Key Concepts

```yaml
# Volume (ephemeral)
volumes:
- name: cache
  emptyDir: {}

# PVC (persistent)
volumes:
- name: data
  persistentVolumeClaim:
    claimName: postgres-pvc
```

**Storage Hierarchy:**
```
StorageClass (how to create)
    ↓
PersistentVolume (actual storage)
    ↓
PersistentVolumeClaim (request)
    ↓
Pod (uses PVC)
```

---

## 💡 Quick Examples

**Create PVC:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

**Use in Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
  - name: postgres
    image: postgres:14
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: postgres-pvc
```

---

## 🎯 Access Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **ReadWriteOnce (RWO)** | Single Node R/W | Databases |
| **ReadOnlyMany (ROX)** | Multiple Nodes R | Static content |
| **ReadWriteMany (RWX)** | Multiple Nodes R/W | Shared storage |

---

## 🚀 Production Tips

```yaml
# 1. Always request specific size
resources:
  requests:
    storage: 100Gi  # Not too small!

# 2. Use StorageClass for dynamic provisioning
storageClassName: fast-ssd

# 3. Backup regularly
# Use Velero or cloud-native backup

# 4. Monitor storage usage
kubectl top pods
kubectl describe pvc

# 5. Set retention policy
persistentVolumeReclaimPolicy: Retain  # or Delete
```

---

[⬅️ Phần 6](../06-configuration/README.md) | [🏠 Mục Lục](../README.md) | [➡️ Phần 8](../08-high-availability/README.md)
