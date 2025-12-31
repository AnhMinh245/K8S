# 11.6. Backup & Disaster Recovery

> Đảm bảo business continuity

---

## 🎯 RTO & RPO

**Recovery Time Objective (RTO):** Thời gian tối đa để restore  
**Recovery Point Objective (RPO):** Data loss tối đa chấp nhận được  

```
Example:
RTO = 1 hour → Phải restore trong 1h
RPO = 15 minutes → Chấp nhận mất tối đa 15 phút data
```

---

## 💾 Velero Backup

### Install Velero

```bash
# Install CLI
brew install velero

# Install server (GCP example)
velero install \
  --provider gcp \
  --plugins velero/velero-plugin-for-gcp:v1.8.0 \
  --bucket velero-backups \
  --secret-file ./credentials-velero
```

### Backup Entire Cluster

```bash
# Full cluster backup
velero backup create full-backup

# Backup specific namespace
velero backup create prod-backup --include-namespaces production

# Scheduled backup (daily)
velero schedule create daily-backup --schedule="0 2 * * *"
```

### Restore

```bash
# List backups
velero backup get

# Restore
velero restore create --from-backup full-backup

# Restore to different namespace
velero restore create --from-backup prod-backup \
  --namespace-mappings production:production-restored
```

---

## 🗄️ etcd Backup

### Manual Backup

```bash
# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db
```

### Automated Backup (CronJob)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: gcr.io/etcd-development/etcd:v3.5.0
            command:
            - /bin/sh
            - -c
            - |
              etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db
              # Upload to GCS
              gsutil cp /backup/etcd-*.db gs://etcd-backups/
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
            - name: backup
              mountPath: /backup
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup
            emptyDir: {}
          restartPolicy: OnFailure
```

---

## 🌍 Multi-Region DR

### Active-Passive Setup

```
Primary Region (us-central1)
  ├─ Production cluster
  ├─ Continuous backups
  └─ Replication to DR region

DR Region (us-east1)
  ├─ Standby cluster (minimal)
  ├─ Restore from backups
  └─ Scale up when needed
```

### Failover Process

```bash
# 1. Verify DR cluster
kubectl --context=dr-cluster get nodes

# 2. Restore latest backup
velero restore create dr-restore \
  --from-backup latest \
  --context=dr-cluster

# 3. Update DNS to point to DR
# (Manual or automated)

# 4. Scale up DR cluster
kubectl --context=dr-cluster scale deployment --all --replicas=3
```

---

## 🎓 Key Takeaways

**1. Velero:** Cluster-level backups  
**2. etcd:** Control plane backup  
**3. RTO/RPO:** Define targets  
**4. Test:** Regular DR drills  

---

[⬅️ 11.5. Monitoring](./05-monitoring-logging.md) | [➡️ 11.7. Cost](./07-cost-optimization.md) | [🏠 Mục Lục](../README.md)

