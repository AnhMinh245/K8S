# 7.3. StorageClass - Dynamic Provisioning

> Automatically provision storage on demand

---

## 🎯 Static vs Dynamic Provisioning

### Static (Manual)
```
1. Admin creates PV manually
2. Developer creates PVC
3. K8s binds PVC to existing PV
```

### Dynamic (Automatic)
```
1. Developer creates PVC
2. K8s automatically creates PV
3. Cloud provider creates disk
4. K8s binds PVC to new PV

No admin intervention! ✅
```

---

## 📝 StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "10"
  encrypted: "true"
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

---

## 🔧 Use StorageClass

**PVC references StorageClass:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd  # ← References StorageClass
  resources:
    requests:
      storage: 100Gi
```

**What happens:**
```
1. PVC created with storageClassName: fast-ssd
2. K8s calls AWS API
3. AWS creates 100GB EBS volume (gp3, encrypted)
4. K8s creates PV automatically
5. Binds PVC to new PV
6. Pod can use PVC
```

---

## ☁️ Cloud Provider Examples

### AWS EBS
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
```

### GCP Persistent Disk
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-pd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
```

### Azure Disk
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-disk
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
```

---

## 🎓 Key Takeaways

1. **StorageClass:** Template for dynamic provisioning
2. **Provisioner:** Creates actual storage (AWS EBS, GCE PD, etc.)
3. **Parameters:** Storage type, IOPS, encryption, etc.
4. **PVC → StorageClass:** Automatic PV creation
5. **No manual PV creation:** Admin-free!
6. **Default StorageClass:** Used when not specified

---

**Chúc mừng!** Hoàn thành **Phần 7: Storage** 🎉

👉 [**Phần 8: High Availability**](../08-high-availability/README.md)

---

[⬅️ 7.2. PV & PVC](./02-persistent-volumes.md) | [⬆️ Phần 7](./README.md) | [🏠 Mục Lục](../README.md)

