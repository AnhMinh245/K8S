# 📘 Phần 4: Workloads - Quản Lý Ứng Dụng

> Controllers để deploy và manage applications trong Kubernetes

---

## 🎯 Mục Tiêu Phần Này

Sau khi hoàn thành Phần 4, bạn sẽ:

✅ **Hiểu tất cả workload types** và use cases  
✅ **Deploy production applications** với Deployments  
✅ **Manage stateful apps** với StatefulSets  
✅ **Run system daemons** với DaemonSets  
✅ **Batch processing** với Jobs và CronJobs  
✅ **Choose right workload** cho mỗi scenario  

---

## 📚 Nội Dung

### [4.1. ReplicaSet - Duy Trì Replicas](./01-replicaset.md) ⭐⭐⭐

**Thời gian:** 45-60 phút

**Nội dung:**
- ReplicaSet là gì và TẠI SAO cần
- Maintain desired number of Pods
- Reconciliation loop
- Self-healing automatic
- Selector và Pod template
- Scaling applications

**Key Concepts:**
```
✓ ReplicaSet = Maintain replica count
✓ Automatic Pod creation/deletion
✓ Self-healing
✓ Selector matching
✓ Low-level controller (use Deployment instead!)
```

**Commands:**
```bash
kubectl get rs
kubectl describe rs <name>
kubectl scale rs <name> --replicas=5
kubectl delete pod <pod-name>  # Test self-healing
```

---

### [4.2. Deployment - Production Workload](./02-deployment.md) ⭐⭐⭐⭐⭐

**Thời gian:** 75-90 phút (QUAN TRỌNG NHẤT!)

**Nội dung:**
- Deployment = ReplicaSet + Updates + Rollbacks
- **Rolling Updates** (zero downtime!)
- **Rollback** khi deployment fails
- Update strategies (RollingUpdate vs Recreate)
- maxSurge và maxUnavailable
- Revision history
- Production deployment patterns

**Key Concepts:**
```
✓ Deployment = Production standard
✓ Rolling updates = Zero downtime
✓ Rollback = Safety net
✓ Declarative updates
✓ Always use Deployment (not ReplicaSet directly!)
```

**Commands:**
```bash
# Deploy
kubectl create deployment app --image=nginx --replicas=3

# Update (rolling)
kubectl set image deployment/app nginx=nginx:1.22
kubectl rollout status deployment/app

# Rollback
kubectl rollout undo deployment/app

# History
kubectl rollout history deployment/app

# Scale
kubectl scale deployment/app --replicas=5
```

---

### [4.3. StatefulSet - Stateful Apps](./03-statefulset.md) ⭐⭐⭐⭐

**Thời gian:** 75-90 phút

**Nội dung:**
- StatefulSet cho databases, caches
- **Stable Pod identity** (ordinal names)
- **Stable network identity** (DNS per Pod)
- **Persistent storage** per Pod
- **Ordered deployment** và scaling
- Headless Services
- volumeClaimTemplates

**Key Concepts:**
```
✓ StatefulSet = Stateful applications
✓ Predictable names: app-0, app-1, app-2
✓ Stable DNS: app-0.service.namespace
✓ Persistent storage per Pod
✓ Sequential operations (0→1→2)
```

**Use Cases:**
- Databases (PostgreSQL, MySQL, MongoDB)
- Distributed caches (Redis, Memcached)
- Message queues (Kafka, RabbitMQ)
- Coordination services (ZooKeeper, etcd)

**Commands:**
```bash
kubectl get statefulset
kubectl scale statefulset redis --replicas=5
kubectl delete pod redis-0  # Test persistence
```

---

### [4.4. DaemonSet - Pod Trên Mỗi Node](./04-daemonset.md) ⭐⭐⭐⭐

**Thời gian:** 60-75 phút

**Nội dung:**
- DaemonSet = One Pod per Node
- System daemons và agents
- **Automatic scaling** với cluster
- Node selectors và tolerations
- Run on Master Nodes
- Update strategies

**Key Concepts:**
```
✓ DaemonSet = Exactly 1 Pod per Node
✓ Auto-scale with cluster
✓ System-level services
✓ Every Node coverage
```

**Use Cases:**
- Log collection (Fluentd, Filebeat)
- Monitoring agents (Node Exporter, Datadog)
- Network plugins (kube-proxy, Calico)
- Storage drivers (CSI plugins)

**Commands:**
```bash
kubectl get daemonset
kubectl set image daemonset/log-collector fluentd=fluentd:v2
```

---

### [4.5. Jobs & CronJobs - Batch Workloads](./05-jobs-cronjobs.md) ⭐⭐⭐⭐

**Thời gian:** 60-75 phút

**Nội dung:**
- Jobs = Run to completion
- CronJobs = Scheduled tasks
- **Parallel jobs** và work queues
- Retry logic (backoffLimit)
- Timeouts (activeDeadlineSeconds)
- TTL cleanup
- Cron syntax

**Key Concepts:**
```
✓ Job = One-time task (completes)
✓ CronJob = Scheduled jobs
✓ Parallel execution
✓ Automatic retries
✓ vs Deployment (always running)
```

**Use Cases:**
- Database migrations
- Backups và restores
- Batch processing
- Report generation
- Scheduled cleanup
- Data pipelines

**Commands:**
```bash
# Job
kubectl create job backup --image=backup-tool
kubectl get jobs

# CronJob
kubectl create cronjob cleanup --schedule="0 2 * * *" --image=cleanup
kubectl get cronjobs
```

---

## 🗺️ Workload Decision Tree

### Chọn Workload Nào?

```
START
  ↓
Always running service?
  ├─ YES → Stateless?
  │         ├─ YES → DEPLOYMENT ✓
  │         └─ NO → StatefulSet (database, cache)
  │
  └─ NO → Run on every Node?
            ├─ YES → DAEMONSET ✓
            └─ NO → One-time task?
                      ├─ YES → JOB ✓
                      └─ NO → Scheduled?
                                ├─ YES → CRONJOB ✓
                                └─ NO → (review requirements)
```

### Quick Reference Table

| Workload | Use When | Examples |
|----------|----------|----------|
| **Deployment** | Stateless, always running | Web apps, APIs, microservices |
| **StatefulSet** | Stateful, need stable identity | Databases, caches, queues |
| **DaemonSet** | One per Node, system-level | Logging, monitoring, network |
| **Job** | One-time task, run to completion | Migrations, backups, batch |
| **CronJob** | Scheduled tasks | Cleanup, reports, periodic jobs |

---

## 🎓 Self-Assessment

### Checkpoint: Sẵn Sàng Phần 5?

**1. Deployment Mastery**
```
□ Create Deployment
□ Rolling update (kubectl set image)
□ Rollback (kubectl rollout undo)
□ Scale up/down
□ Understand maxSurge/maxUnavailable
```

**2. StatefulSet Understanding**
```
□ Know when to use (databases)
□ Understand stable identity
□ Create with Headless Service
□ Persistent storage per Pod
□ Ordered deployment
```

**3. DaemonSet Concepts**
```
□ One Pod per Node
□ Auto-scaling với cluster
□ Use nodeSelector
□ Tolerate Master taints
```

**4. Jobs & CronJobs**
```
□ Create Job
□ Understand completions vs parallelism
□ CronJob với cron syntax
□ Set backoffLimit và timeout
```

**If all checked → Ready for Phần 5! 🎉**

---

## 💪 Consolidated Exercises

### Exercise 1: Complete Application Stack

Deploy multi-tier application với appropriate workloads:

```yaml
# 1. Frontend (Deployment - stateless)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80

---
# 2. Backend API (Deployment - stateless)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: api
        image: api-server:v1
        ports:
        - containerPort: 8080

---
# 3. Database (StatefulSet - stateful)
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
        ports:
        - containerPort: 5432
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi

---
# 4. Log Collector (DaemonSet - per Node)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
      - name: fluentd
        image: fluentd:v1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log

---
# 5. Backup (CronJob - scheduled)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: backup
            image: backup-tool:v1
```

```bash
# Deploy all
kubectl apply -f application-stack.yaml

# Verify each workload type
kubectl get deployments
kubectl get statefulsets
kubectl get daemonsets
kubectl get cronjobs

# Test scenarios
kubectl scale deployment frontend --replicas=5
kubectl set image deployment/backend api=api-server:v2
kubectl delete pod postgres-0  # Test StatefulSet persistence
```

---

### Exercise 2: Workload Comparison

Deploy same app với different workloads, compare behavior:

```bash
# 1. As Deployment
kubectl create deployment test-deploy --image=nginx --replicas=3
kubectl get pods -o wide  # Random Node placement

# 2. As DaemonSet
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: test-daemon
spec:
  selector:
    matchLabels:
      app: test-daemon
  template:
    metadata:
      labels:
        app: test-daemon
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
kubectl get pods -o wide  # One per Node

# 3. As Job
kubectl create job test-job --image=nginx -- sleep 10
kubectl get jobs  # Completes và stops

# Observe differences!
```

---

## 🎯 Key Takeaways - Phần 4

### 10 Điều Quan Trọng Nhất

**1. Deployment = Production Standard**
```
Always use for stateless apps
Rolling updates, rollbacks built-in
Best for: Web apps, APIs, microservices
```

**2. StatefulSet = Stateful Apps**
```
Stable identity, persistent storage
Ordered operations
Best for: Databases, caches, queues
```

**3. DaemonSet = System Services**
```
One Pod per Node, auto-scaling
System-level coverage
Best for: Logging, monitoring, network
```

**4. Job = Run to Completion**
```
One-time tasks, batch processing
Automatic retries, timeouts
Best for: Migrations, backups, batch
```

**5. CronJob = Scheduled Tasks**
```
Cron syntax, automatic Job creation
Recurring tasks
Best for: Cleanup, reports, periodic
```

**6. Rolling Updates = Zero Downtime**
```
Gradual rollout (old → new)
maxSurge và maxUnavailable
Production-safe updates
```

**7. Self-Healing Automatic**
```
Pod crashes → Auto-recreated
Node fails → Pods rescheduled
No manual intervention
```

**8. Declarative Management**
```
Describe desired state
K8s makes it happen
Continuous reconciliation
```

**9. Choose Right Workload**
```
Stateless always running → Deployment
Stateful always running → StatefulSet
Per-Node system → DaemonSet
One-time task → Job
Scheduled task → CronJob
```

**10. Best Practices**
```
✓ Always set resource requests/limits
✓ Configure readiness/liveness probes
✓ Use labels for organization
✓ Set appropriate replica counts
✓ Test rollback procedures
```

---

## 📚 Commands Cheat Sheet

### Deployment

```bash
# Create
kubectl create deployment app --image=nginx --replicas=3

# Update
kubectl set image deployment/app nginx=nginx:1.22
kubectl rollout status deployment/app

# Rollback
kubectl rollout undo deployment/app
kubectl rollout history deployment/app

# Scale
kubectl scale deployment/app --replicas=5

# Delete
kubectl delete deployment app
```

### StatefulSet

```bash
# Create (need Headless Service first!)
kubectl apply -f statefulset.yaml

# Scale
kubectl scale statefulset redis --replicas=5

# Update
kubectl set image statefulset/redis redis=redis:7

# Delete (keeps PVCs!)
kubectl delete statefulset redis
kubectl delete pvc data-redis-0  # Manual cleanup
```

### DaemonSet

```bash
# Create
kubectl apply -f daemonset.yaml

# Update
kubectl set image daemonset/log-collector fluentd=fluentd:v2
kubectl rollout status daemonset/log-collector

# Delete
kubectl delete daemonset log-collector
```

### Jobs & CronJobs

```bash
# Job
kubectl create job backup --image=backup-tool
kubectl get jobs
kubectl logs job/backup

# CronJob
kubectl create cronjob cleanup --schedule="0 2 * * *" --image=cleanup
kubectl get cronjobs
kubectl get jobs  # See created Jobs

# Suspend/Resume CronJob
kubectl patch cronjob cleanup -p '{"spec":{"suspend":true}}'
kubectl patch cronjob cleanup -p '{"spec":{"suspend":false}}'
```

---

## ❓ FAQs

**Q: Khi nào dùng StatefulSet vs Deployment cho database?**
```
A: StatefulSet nếu cần:
✓ Stable Pod names (postgres-0, postgres-1)
✓ Persistent storage per instance
✓ Ordered deployment (primary before replicas)
✓ Peer discovery (cluster formation)

Deployment nếu:
✓ Stateless database proxy
✓ Read-only replicas (no strict order)
✓ Shared storage (not recommended!)

Recommendation: StatefulSet for databases!
```

**Q: DaemonSet có auto-scale không?**
```
A: YES! Automatic.

3 Nodes → 3 DaemonSet Pods
Add Node 4 → 4 Pods (auto-created!)
Remove Node 2 → 3 Pods (auto-deleted!)

No manual scaling needed!
```

**Q: Job vs CronJob - khác gì?**
```
A:
Job = One-time task
- Create manually
- Runs once (or multiple completions)
- Example: Database migration

CronJob = Scheduled tasks
- Runs on schedule (cron format)
- Automatically creates Jobs
- Example: Daily backups

CronJob internally creates Jobs!
```

**Q: Có thể rollback StatefulSet không?**
```
A: Yes, but more complex than Deployment:

kubectl rollout undo statefulset/redis

But:
- Updates in reverse order (2→1→0)
- Slower than Deployment
- May need manual intervention for data

Recommendation: Test thoroughly before production update!
```

---

## 🚀 Tiếp Theo

**Completed:** Workloads - Quản lý ứng dụng ✅

**Next:** [Phần 5: Networking →](../05-networking/README.md)

Learn about:
- Services (ClusterIP, NodePort, LoadBalancer)
- Pod-to-Pod communication
- Ingress controllers
- Network Policies
- DNS trong Kubernetes

Let's connect our applications! 🌐

---

[⬅️ Phần 3: Core Concepts](../03-core-concepts/README.md) | [🏠 Mục Lục Chính](../README.md) | [➡️ Phần 5: Networking](../05-networking/README.md)
