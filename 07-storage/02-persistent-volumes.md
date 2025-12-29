# 7.2. PersistentVolume & PersistentVolumeClaim

> Abstraction layer cho persistent storage

---

## 🎯 PV & PVC Workflow

```
┌─────────────────────────────────────────┐
│  1. Admin creates PersistentVolume (PV) │
│     "I have 100GB storage available"    │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│ 2. Developer creates PVC                │
│    "I need 50GB storage"                │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│ 3. K8s binds PVC to PV                  │
│    PVC gets 50GB from that PV           │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│ 4. Pod mounts PVC                       │
│    Pod uses storage                     │
└─────────────────────────────────────────┘
```

---

## 📝 PersistentVolume (PV)

**Admin creates PV:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-1
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  awsElasticBlockStore:
    volumeID: vol-0123456789
    fsType: ext4
```

---

## 📝 PersistentVolumeClaim (PVC)

**Developer requests storage:**

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
      storage: 50Gi
  storageClassName: standard
```

**K8s finds matching PV and binds**

---

## 🔧 Pod Uses PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

---

## 🔑 Access Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **ReadWriteOnce (RWO)** | 1 Node mount read/write | Database, single Pod |
| **ReadOnlyMany (ROX)** | Many Nodes read-only | Shared config, static content |
| **ReadWriteMany (RWX)** | Many Nodes read/write | Shared storage (NFS, CephFS) |

---

## 🔄 Reclaim Policy

**What happens when PVC deleted?**

| Policy | Behavior |
|--------|----------|
| **Retain** | PV remains, data kept, manual cleanup |
| **Delete** | PV and underlying storage deleted |
| **Recycle** | Data deleted, PV reusable (deprecated) |

---

## 🎓 Key Takeaways

1. **PV:** Admin provisions storage
2. **PVC:** Developer requests storage
3. **Binding:** K8s matches PVC to PV
4. **Pod:** Uses PVC, not PV directly
5. **Access modes:** RWO, ROX, RWX
6. **Reclaim policy:** What happens when PVC deleted

---

[⬅️ 7.1. Volumes](./01-volumes.md) | [➡️ 7.3. StorageClass](./03-storage-classes.md) | [🏠 Mục Lục](../README.md)

