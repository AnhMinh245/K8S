# 7.1. Volumes

> Persistent data trong Kubernetes

---

## 🎯 Vấn Đề

**Container filesystem is ephemeral:**
```
Container writes file → Container restarts → File LOST ❌
```

**Solution: Volumes**
```
Container writes to Volume → Container restarts → File PERSISTS ✅
```

---

## 📦 Volume Types

### 1. emptyDir
**Temporary storage, shared between containers in Pod**

```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: cache
      mountPath: /cache
  - name: sidecar
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir: {}

# Pod deleted → emptyDir deleted
```

**Use case:** Cache, scratch space, shared data between containers

---

### 2. hostPath
**Mount directory from Node**

```yaml
volumes:
- name: logs
  hostPath:
    path: /var/log
    type: Directory
```

**⚠️ Warning:** Pod on different Node → Different data!

**Use case:** DaemonSet accessing Node filesystem

---

### 3. Cloud Volumes

**AWS EBS:**
```yaml
volumes:
- name: data
  awsElasticBlockStore:
    volumeID: vol-0123456789
    fsType: ext4
```

**GCE PD:**
```yaml
volumes:
- name: data
  gcePersistentDisk:
    pdName: my-disk
    fsType: ext4
```

---

### 4. NFS
```yaml
volumes:
- name: shared
  nfs:
    server: nfs-server.example.com
    path: /exports
```

---

## 💡 Key Points

1. **emptyDir:** Temporary, Pod-scoped
2. **hostPath:** Node-specific, risky
3. **Cloud volumes:** Persistent, but hard-coded IDs
4. **Better solution:** PersistentVolume & PVC (next section)

---

[➡️ 7.2. PersistentVolume & PVC](./02-persistent-volumes.md) | [🏠 Mục Lục](../README.md)

