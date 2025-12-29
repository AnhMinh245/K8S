# 4.2. Deployment - Workload Phổ Biến Nhất

> Deployment là workload controller được dùng nhiều nhất trong K8s

---

## 🎯 Deployment Là Gì?

**Deployment** = Higher-level controller quản lý ReplicaSets và Pods, hỗ trợ rolling updates và rollback

```
Deployment
  ↓ manages
ReplicaSet (v1.21)
  ↓ manages
Pods (nginx:1.21)
```

---

## 🏢 Ví Dụ Thực Tế

**Deployment = Quản lý cập nhật phần mềm**

```
Version cũ (1.0):
  - 3 servers đang chạy v1.0

Update sang version 1.1:
  ❌ Cách cũ (downtime):
     1. Stop all 3 servers
     2. Update to v1.1
     3. Start all 3 servers
     → Downtime 5-10 phút

  ✅ Deployment (zero downtime):
     1. Start 1 server v1.1
     2. Test → OK
     3. Stop 1 server v1.0
     4. Repeat until all updated
     → No downtime!
```

---

## 📝 Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  labels:
    app: web
spec:
  replicas: 3                    # Desired Pods
  selector:
    matchLabels:
      app: web
  template:                      # Pod template
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

---

## 🔄 Deployment Workflow

### Create Deployment

```
1. User: kubectl apply -f deployment.yaml

2. Deployment Controller:
   - Creates ReplicaSet
   - Sets replicas = 3

3. ReplicaSet Controller:
   - Creates 3 Pods

4. Scheduler:
   - Assigns Pods to Nodes

5. kubelet:
   - Starts containers

Result:
  Deployment → ReplicaSet → 3 Pods (running)
```

---

## 🚀 Rolling Update

**Scenario:** Update image từ `nginx:1.20` → `nginx:1.21`

### Update Process

```yaml
# Old Deployment
spec:
  replicas: 3
  template:
    spec:
      containers:
      - image: nginx:1.20  # ← Change this

# Update
kubectl set image deployment/web-deployment nginx=nginx:1.21
# Or edit YAML and apply
```

### Rolling Update Flow

```
Initial state:
  ReplicaSet-v1 (nginx:1.20): 3 Pods

Step 1: Create new ReplicaSet
  ReplicaSet-v1: 3 Pods
  ReplicaSet-v2: 0 Pods  ← New

Step 2: Scale up new, scale down old
  ReplicaSet-v1: 3 Pods
  ReplicaSet-v2: 1 Pod   ← Created

Step 3: Wait for new Pod ready
  Health check passes → Continue

Step 4: Continue rolling
  ReplicaSet-v1: 2 Pods  ← Terminated 1
  ReplicaSet-v2: 2 Pods  ← Created 1 more

Step 5: Continue...
  ReplicaSet-v1: 1 Pod
  ReplicaSet-v2: 3 Pods

Step 6: Complete
  ReplicaSet-v1: 0 Pods  ← Scaled to 0
  ReplicaSet-v2: 3 Pods  ← All running

Result: Zero downtime update! ✅
```

---

## ⚙️ Update Strategies

### 1. RollingUpdate (Default)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # Max extra Pods during update
      maxUnavailable: 1   # Max unavailable Pods
```

**Example với 3 replicas:**

```
maxSurge: 1, maxUnavailable: 1

Step 1: 3 old + 1 new = 4 Pods (maxSurge allows)
Step 2: 2 old + 2 new = 4 Pods
Step 3: 1 old + 3 new = 4 Pods
Step 4: 0 old + 3 new = 3 Pods ✅
```

**Use case:** Most applications (default choice)

---

### 2. Recreate

```yaml
spec:
  strategy:
    type: Recreate
```

**Process:**
```
Step 1: Delete ALL old Pods (3 → 0)
Step 2: Wait for termination
Step 3: Create ALL new Pods (0 → 3)

Result: Downtime during update ❌
```

**Use case:**
- Application không support multiple versions cùng lúc
- Shared resource conflicts (database migration)
- Development/testing environments

---

## 🔙 Rollback

### Rollout History

```bash
# View rollout history
kubectl rollout history deployment/web-deployment

# Output:
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image nginx=1.21
3         kubectl set image nginx=1.22
```

### Rollback Commands

```bash
# Rollback to previous version
kubectl rollout undo deployment/web-deployment

# Rollback to specific revision
kubectl rollout undo deployment/web-deployment --to-revision=2

# Check rollout status
kubectl rollout status deployment/web-deployment
```

### Rollback Flow

```
Current: v1.22 (broken)

1. kubectl rollout undo

2. Deployment Controller:
   - Find previous ReplicaSet (v1.21)
   - Scale up ReplicaSet-v1.21: 0 → 3
   - Scale down ReplicaSet-v1.22: 3 → 0

3. Result: Back to v1.21 in ~30 seconds ✅
```

---

## 📊 Monitoring Deployment

### Check Status

```bash
# List Deployments
kubectl get deployments

# Output:
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
web-deployment   3/3     3            3           10m

# READY: Current/Desired Pods
# UP-TO-DATE: Pods with latest template
# AVAILABLE: Pods ready to serve traffic
```

### Detailed Info

```bash
kubectl describe deployment web-deployment

# Shows:
# - Replicas status
# - Update strategy
# - Conditions
# - Events (scaling, updates)
# - ReplicaSets managed
```

### Watch Rollout

```bash
# Watch update progress
kubectl rollout status deployment/web-deployment

# Output:
Waiting for deployment "web-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "web-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "web-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "web-deployment" successfully rolled out
```

---

## 🎨 Advanced Features

### 1. Pause & Resume

```bash
# Pause rollout (make multiple changes)
kubectl rollout pause deployment/web-deployment

# Make changes
kubectl set image deployment/web-deployment nginx=nginx:1.22
kubectl set resources deployment/web-deployment -c nginx --limits=cpu=1

# Resume (apply all changes at once)
kubectl rollout resume deployment/web-deployment
```

**Use case:** Batch multiple updates

---

### 2. Revision History Limit

```yaml
spec:
  revisionHistoryLimit: 10  # Keep last 10 ReplicaSets
```

**Default:** 10 revisions

**Why limit:**
- Old ReplicaSets (scaled to 0) still stored
- Cleanup old revisions to save etcd space

---

### 3. Progress Deadline

```yaml
spec:
  progressDeadlineSeconds: 600  # 10 minutes
```

**Meaning:**
- Deployment must progress in 10 minutes
- If stuck → Marked as failed
- Prevents infinite wait

---

### 4. Min Ready Seconds

```yaml
spec:
  minReadySeconds: 30  # Wait 30s before considering Pod ready
```

**Use case:**
- App needs warm-up time
- Prevent premature traffic routing

---

## 🔧 Scaling Deployments

### Manual Scaling

```bash
# Scale up
kubectl scale deployment web-deployment --replicas=10

# Scale down
kubectl scale deployment web-deployment --replicas=2

# Declarative (edit YAML)
# Change replicas: 3 → 5
kubectl apply -f deployment.yaml
```

### Auto-Scaling (HPA)

```bash
# Create HorizontalPodAutoscaler
kubectl autoscale deployment web-deployment \
  --min=3 \
  --max=10 \
  --cpu-percent=70

# HPA automatically scales based on CPU
# CPU > 70% → Scale up
# CPU < 70% → Scale down
```

---

## 💡 Best Practices

### ✅ DO

1. **Always use Deployment** (not ReplicaSet directly)

2. **Set resource limits**
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

3. **Health checks**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
```

4. **Meaningful labels**
```yaml
metadata:
  labels:
    app: web
    version: v1.2.3
    environment: production
```

5. **Update strategy**
```yaml
strategy:
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
```

6. **Record changes**
```bash
kubectl apply -f deployment.yaml --record
# Saves command in rollout history
```

---

### ❌ DON'T

1. **No resource limits** → OOMKilled, node starvation
2. **No health checks** → Route traffic to broken Pods
3. **Recreate strategy** in prod → Downtime
4. **Manual Pod edits** → Changes lost on next rollout
5. **Too aggressive updates** → maxUnavailable=100%

---

## 🎯 Common Use Cases

### 1. Web Applications

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 5
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: react-app
        image: myapp/frontend:v1.2.3
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

### 2. API Services

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Never reduce capacity during update
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
        version: v2.0.1
    spec:
      containers:
      - name: api
        image: myapp/api:v2.0.1
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
```

---

### 3. Background Workers

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
      - name: worker
        image: myapp/worker:latest
        env:
        - name: QUEUE_URL
          value: "redis://redis:6379"
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
```

---

## 🎓 Key Takeaways

1. **Deployment:** Most common workload in K8s
2. **Rolling updates:** Zero downtime deployments
3. **Rollback:** Easy revert to previous version
4. **Manages ReplicaSets:** Deployment → ReplicaSet → Pods
5. **Update strategies:** RollingUpdate (default), Recreate
6. **Health checks:** Essential for safe rollouts
7. **Scalable:** Manual or auto-scaling (HPA)

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. Deployment khác gì với ReplicaSet?
2. Rolling update hoạt động như thế nào?
3. Làm sao rollback về version trước?
4. maxSurge và maxUnavailable là gì?
5. Khi nào dùng Recreate strategy?
6. Health checks ảnh hưởng rolling update như thế nào?

---

## 🚀 Tiếp Theo

👉 [4.3. StatefulSet - Cho Ứng Dụng Stateful](./03-statefulset.md)

StatefulSet cho database và ứng dụng cần persistent identity

---

[⬅️ 4.1. ReplicaSet](./01-replicaset.md) | [⬆️ Phần 4](./README.md) | [🏠 Mục Lục Chính](../README.md)

