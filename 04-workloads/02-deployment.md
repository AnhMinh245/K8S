# 4.2. Deployment - Production Workload Management

> Rolling updates, rollbacks, và declarative application management

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **Deployment là gì** và **TẠI SAO dùng thay vì ReplicaSet**
- ✅ Thực hiện **Rolling Updates** zero-downtime
- ✅ **Rollback** khi deployment fails
- ✅ Hiểu **Deployment strategies** (RollingUpdate vs Recreate)
- ✅ **Scale applications** horizontally
- ✅ **Troubleshoot** deployments trong production

---

## 📦 Deployment Là Gì?

### Định Nghĩa

**Deployment** = High-level controller quản lý ReplicaSets và Pods, provides declarative updates và rollback capabilities.

### Giải Thích Bằng Ví Dụ

**Deployment giống như quản lý rollout phiên bản app:**

```
🏢 TRIỂN KHAI PHẦN MỀM MỚI

Old way (Manual - like bare ReplicaSet):
1. Stop old version (downtime! ❌)
2. Deploy new version
3. Test
4. If fails: Panic! Rollback manual
→ Downtime, risk, stress

Deployment way (Automated):
1. Deploy new version gradually
   Old: 3 Pods → 2 Pods → 1 Pod → 0 Pods
   New: 0 Pods → 1 Pod → 2 Pods → 3 Pods
   → Always have running Pods! (Zero downtime ✓)

2. If new version fails:
   kubectl rollout undo deployment/app
   → Automatic rollback to previous version!

3. Track history:
   → Revision 1: v1.0
   → Revision 2: v1.1
   → Revision 3: v1.2 (current)
```

---

## 🤔 TẠI SAO Cần Deployment?

### ReplicaSet Limitations

```yaml
# With ReplicaSet only
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: webapp-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      version: v1  # Version in selector!
  template:
    metadata:
      labels:
        app: webapp
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

**Update to v2:**

```bash
# Problem: How to update to nginx:1.22?

# Option 1: Update ReplicaSet (DOESN'T WORK!)
kubectl edit rs webapp-v1
# Change image: nginx:1.22
# Save and exit
# → Pods NOT updated! Still running 1.21!
# ReplicaSet doesn't recreate Pods on template change

# Option 2: Delete Pods manually
kubectl delete pods -l app=webapp
# → Downtime while Pods restart!
# ❌ Manual process
# ❌ Downtime

# Option 3: Create new ReplicaSet, delete old
kubectl apply -f webapp-v2-rs.yaml  # New RS với v2
kubectl delete rs webapp-v1         # Delete old RS
# → Downtime!
# ❌ No gradual rollout
# ❌ No automatic rollback
```

### Deployment Solution

```yaml
# With Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

**Update to v2:**

```bash
# Simple! Just update image
kubectl set image deployment/webapp nginx=nginx:1.22

# Deployment automatically:
# 1. Creates new ReplicaSet (nginx:1.22)
# 2. Gradually scales up new RS
# 3. Gradually scales down old RS
# 4. Ensures always running Pods (zero downtime!)
# 5. Saves revision history

# If problem? Rollback:
kubectl rollout undo deployment/webapp
# → Automatic rollback to 1.21!

✓ Zero downtime
✓ Automated rollout
✓ Easy rollback
✓ Revision tracking
```

---

## 🏗️ Deployment Architecture

### Three-Layer Structure

```
┌─────────────────────────────────────────────────┐
│               DEPLOYMENT                        │
│  (Desired state: 3 replicas, image: nginx:1.22)│
└────────────────┬────────────────────────────────┘
                 │ manages
                 ↓
┌─────────────────────────────────────────────────┐
│            REPLICASETS                          │
│                                                 │
│  Old RS (nginx:1.21)         New RS (nginx:1.22)│
│  ┌──────────────────┐        ┌─────────────────┐│
│  │ Replicas: 0      │        │ Replicas: 3     ││
│  │ (scaled down)    │        │ (scaled up)     ││
│  └──────────────────┘        └────────┬────────┘│
└─────────────────────────────────────────────────┘
                                         │
                                         │ manages
                                         ↓
                          ┌──────────────────────────┐
                          │        PODS              │
                          │  ┌────┐ ┌────┐ ┌────┐  │
                          │  │Pod1│ │Pod2│ │Pod3│  │
                          │  │v1.22│ │v1.22│ │v1.22││
                          │  └────┘ └────┘ └────┘  │
                          └──────────────────────────┘
```

### Deployment Flow

```
USER creates/updates Deployment
    ↓
Deployment Controller watches
    ↓
Creates/Updates ReplicaSet
    ↓
ReplicaSet Controller watches
    ↓
Creates/Updates Pods
    ↓
Pods running with desired state
```

---

## 📝 Deployment YAML

### Complete Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  # Số lượng replicas
  replicas: 3
  
  # Selector để match Pods
  selector:
    matchLabels:
      app: webapp
  
  # Template cho Pods
  template:
    metadata:
      labels:
        app: webapp
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
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
  
  # Update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max additional Pods during update
      maxUnavailable: 0  # Max unavailable Pods during update
  
  # Minimum time Pod is ready before considered available
  minReadySeconds: 10
  
  # Revision history limit
  revisionHistoryLimit: 10
```

---

## 🔄 Rolling Update Strategy

### Default Strategy: RollingUpdate

**Parameters:**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # Max PODs above desired (25% default)
    maxUnavailable: 0  # Max PODs unavailable during update (25% default)
```

**maxSurge:**
```
Desired replicas: 3
maxSurge: 1

During update:
├── Max total Pods: 3 + 1 = 4
├── Can have 4 Pods temporarily
└── Extra capacity for graceful transition
```

**maxUnavailable:**
```
Desired replicas: 3
maxUnavailable: 0

During update:
├── Min available Pods: 3 - 0 = 3
├── Must always have 3 Pods running
└── Zero downtime guaranteed!
```

### Rolling Update Process

**Step-by-step workflow:**

```
Initial state:
Desired: 3 replicas, image: nginx:1.21
Old RS: 3 Pods (nginx:1.21)
New RS: 0 Pods

User updates: image → nginx:1.22
    ↓
Step 1: Deployment creates new ReplicaSet
Old RS: 3 Pods (nginx:1.21)
New RS: 0 Pods (nginx:1.22) ← Created!
    ↓
Step 2: Scale up new RS by 1 (maxSurge: 1)
Old RS: 3 Pods (nginx:1.21)
New RS: 1 Pod (nginx:1.22) ← Creating
Total: 4 Pods (temporary)
    ↓
Step 3: Wait for new Pod Ready
New RS: 1 Pod (nginx:1.22) ← Running + Ready!
    ↓
Step 4: Scale down old RS by 1 (maxUnavailable: 0)
Old RS: 2 Pods (nginx:1.21) ← 1 Terminated
New RS: 1 Pod (nginx:1.22)
Total: 3 Pods (back to desired)
    ↓
Step 5: Scale up new RS by 1
Old RS: 2 Pods (nginx:1.21)
New RS: 2 Pods (nginx:1.22) ← Creating
Total: 4 Pods (temporary)
    ↓
Step 6: Wait for Ready, then scale down old
Old RS: 1 Pod (nginx:1.21) ← 1 Terminated
New RS: 2 Pods (nginx:1.22)
Total: 3 Pods
    ↓
Step 7: Repeat until complete
Old RS: 0 Pods (nginx:1.21) ← Fully scaled down
New RS: 3 Pods (nginx:1.22) ← Fully scaled up
Total: 3 Pods

✓ Update complete!
✓ Zero downtime (always 3 Pods available)
```

### Visual Timeline

```
Time    Old RS (1.21)    New RS (1.22)    Total
────────────────────────────────────────────────
t0      ███             -                3
t1      ███             █ (creating)     4
t2      ███             █ (ready)        4
t3      ██              █ (terminate 1)  3
t4      ██              ██ (creating)    4
t5      ██              ██ (ready)       4
t6      █               ██ (terminate 1) 3
t7      █               ███ (creating)   4
t8      █               ███ (ready)      4
t9      -               ███ (terminate 1)3

Final:  Old=0           New=3            3 ✓
```

---

## 🎮 Hands-On: Deployment Lifecycle

### Create Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

```bash
# Create Deployment
kubectl apply -f deployment.yaml

# Output:
# deployment.apps/nginx-deployment created

# Check Deployment
kubectl get deployments
# or shorter
kubectl get deploy

# Output:
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deployment   3/3     3            3           30s

# Check ReplicaSets (created by Deployment)
kubectl get rs

# Output:
# NAME                          DESIRED   CURRENT   READY   AGE
# nginx-deployment-7d4b7c9d8f   3         3         3       30s

# Check Pods (created by ReplicaSet)
kubectl get pods

# Output:
# NAME                                READY   STATUS    RESTARTS   AGE
# nginx-deployment-7d4b7c9d8f-abc12   1/1     Running   0          30s
# nginx-deployment-7d4b7c9d8f-def34   1/1     Running   0          30s
# nginx-deployment-7d4b7c9d8f-ghi56   1/1     Running   0          30s
```

---

### Update Deployment (Rolling Update)

**Method 1: kubectl set image**

```bash
# Update image từ 1.21 → 1.22
kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# Output:
# deployment.apps/nginx-deployment image updated

# Watch rollout status
kubectl rollout status deployment/nginx-deployment

# Output:
# Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
# Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
# Waiting for deployment "nginx-deployment" rollout to finish: 2 old replicas are pending termination...
# Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
# deployment "nginx-deployment" successfully rolled out

# Verify
kubectl get rs

# Output: 2 ReplicaSets!
# NAME                          DESIRED   CURRENT   READY   AGE
# nginx-deployment-7d4b7c9d8f   0         0         0       5m    ← Old (1.21)
# nginx-deployment-5d9f7b8c6e   3         3         3       1m    ← New (1.22)

kubectl describe deployment nginx-deployment | grep Image
# Image: nginx:1.22
```

**Method 2: kubectl edit**

```bash
# Edit Deployment YAML interactively
kubectl edit deployment nginx-deployment

# In editor, change:
# spec.template.spec.containers[0].image: nginx:1.22 → nginx:1.23

# Save and exit
# Deployment automatically triggers rollout!

# Watch
kubectl rollout status deployment/nginx-deployment
```

**Method 3: Update YAML file và apply**

```yaml
# deployment.yaml (updated)
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.23  # Changed from 1.22
```

```bash
# Apply changes
kubectl apply -f deployment.yaml

# Deployment detects difference và rolls out!
kubectl rollout status deployment/nginx-deployment
```

---

### Watch Rolling Update in Real-Time

```bash
# Terminal 1: Watch Pods
kubectl get pods -w

# Terminal 2: Trigger update
kubectl set image deployment/nginx-deployment nginx=nginx:1.23

# Terminal 1 shows:
# NAME                                READY   STATUS              AGE
# nginx-deployment-5d9f7b8c6e-abc12   1/1     Running             5m
# nginx-deployment-5d9f7b8c6e-def34   1/1     Running             5m
# nginx-deployment-5d9f7b8c6e-ghi56   1/1     Running             5m
# nginx-deployment-6e8g8d9f7e-jkl78   0/1     ContainerCreating   2s  ← New!
# nginx-deployment-6e8g8d9f7e-jkl78   1/1     Running             5s  ← Ready!
# nginx-deployment-5d9f7b8c6e-abc12   1/1     Terminating         5m  ← Old terminating
# nginx-deployment-6e8g8d9f7e-mno90   0/1     ContainerCreating   3s  ← New!
# ... continues until all updated ...
```

---

### Pause và Resume Rollout

```bash
# Pause rollout (useful for canary testing)
kubectl rollout pause deployment/nginx-deployment

# Update image
kubectl set image deployment/nginx-deployment nginx=nginx:1.24

# Deployment created new RS but paused!
kubectl get rs
# New RS exists nhưng replicas = 0

# Check a subset of Pods với new version
# ... test/monitor ...

# Resume rollout
kubectl rollout resume deployment/nginx-deployment

# Continues rolling update!
kubectl rollout status deployment/nginx-deployment
```

---

### Rollback Deployment

**View Rollout History**

```bash
# List revision history
kubectl rollout history deployment/nginx-deployment

# Output:
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
# 3         <none>

# View specific revision details
kubectl rollout history deployment/nginx-deployment --revision=2

# Output:
# deployment.apps/nginx-deployment with revision #2
# Pod Template:
#   Labels:       app=nginx
#   Containers:
#    nginx:
#     Image:      nginx:1.22
#     Port:       80/TCP
#     ...
```

**Record Change Cause**

```bash
# Update với --record flag (adds to CHANGE-CAUSE)
kubectl set image deployment/nginx-deployment nginx=nginx:1.23 --record

# History now shows:
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
# 3         kubectl set image deployment/nginx-deployment nginx=nginx:1.23 --record
```

**Rollback to Previous Version**

```bash
# Rollback to previous revision
kubectl rollout undo deployment/nginx-deployment

# Output:
# deployment.apps/nginx-deployment rolled back

# Check status
kubectl rollout status deployment/nginx-deployment

# Verify image
kubectl describe deployment nginx-deployment | grep Image
# Image: nginx:1.22 (back to previous!)

# History updates:
kubectl rollout history deployment/nginx-deployment
# REVISION  CHANGE-CAUSE
# 1         <none>
# 3         kubectl set image...
# 4         <none>  ← Rollback created new revision!
```

**Rollback to Specific Revision**

```bash
# Rollback to revision 1
kubectl rollout undo deployment/nginx-deployment --to-revision=1

# Back to original image!
kubectl describe deployment nginx-deployment | grep Image
# Image: nginx:1.21
```

---

## ⚙️ Deployment Strategies

### Strategy 1: RollingUpdate (Default)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

**Use cases:**
- ✓ Stateless applications
- ✓ Zero downtime requirement
- ✓ Gradual rollout preferred
- ✓ Quick rollback capability

**Example configurations:**

```yaml
# Conservative (safest, slowest)
maxSurge: 1           # Add 1 at a time
maxUnavailable: 0     # Never below desired

# Balanced
maxSurge: 25%         # Add 25% extra
maxUnavailable: 25%   # Can lose 25%

# Aggressive (fastest, riskier)
maxSurge: 100%        # Double Pods temporarily
maxUnavailable: 50%   # Half can be down
```

---

### Strategy 2: Recreate

```yaml
spec:
  strategy:
    type: Recreate
```

**Process:**
```
1. Delete ALL old Pods
2. Wait for all terminated
3. Create all new Pods
4. Wait for all running

→ Downtime between step 2 and 4!
```

**Use cases:**
- ✓ Stateful applications that can't run multiple versions
- ✓ Database migrations required
- ✓ Shared resources (can't have old và new together)
- ✓ Downtime acceptable

**Example:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-app
spec:
  replicas: 1
  strategy:
    type: Recreate  # Terminate old before new
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:14
```

```bash
# Update triggers Recreate strategy
kubectl set image deployment/database-app postgres=postgres:15

# Behavior:
# 1. Old Pod: Running → Terminating → Terminated
# 2. Downtime period (no Pods running!)
# 3. New Pod: Creating → Running
```

---

## 📊 Scaling Deployments

### Manual Scaling

```bash
# Scale up to 5 replicas
kubectl scale deployment nginx-deployment --replicas=5

# Verify
kubectl get deployment nginx-deployment
# READY: 5/5

kubectl get pods -l app=nginx
# 5 Pods running

# Scale down to 2
kubectl scale deployment nginx-deployment --replicas=2

# 3 Pods terminated
kubectl get pods -l app=nginx
# 2 Pods running
```

### Autoscaling (HPA)

```bash
# Create Horizontal Pod Autoscaler
kubectl autoscale deployment nginx-deployment --min=3 --max=10 --cpu-percent=80

# HPA scales based on CPU usage
# CPU > 80% → Scale up (max 10)
# CPU < 80% → Scale down (min 3)

# Check HPA
kubectl get hpa

# Output:
# NAME               REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS
# nginx-deployment   Deployment/nginx-deployment   50%/80%   3         10        3
```

---

## 🐛 Troubleshooting Deployments

### Issue 1: Deployment Stuck (Progressing)

```bash
$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/3     2            2           10m

# Stuck at 2/3 for long time

# Check Deployment events
$ kubectl describe deployment nginx-deployment

# Events might show:
# ReplicaSet "nginx-deployment-xyz" has timed out progressing

# Check Pod status
$ kubectl get pods -l app=nginx

# NAME                                READY   STATUS             RESTARTS   AGE
# nginx-deployment-xyz-abc12          1/1     Running            0          10m
# nginx-deployment-xyz-def34          1/1     Running            0          10m
# nginx-deployment-xyz-ghi56          0/1     ImagePullBackOff   0          10m

# Fix: Check Pod errors (image issues, resource limits, etc.)
```

---

### Issue 2: Rollout Failed (Bad Image)

```bash
# Update to non-existent image
kubectl set image deployment/nginx-deployment nginx=nginx:wrongtag

# Rollout starts but Pods fail
kubectl get pods

# NAME                                READY   STATUS             RESTARTS   AGE
# nginx-deployment-bad123-abc        0/1     ImagePullBackOff    0          2m
# nginx-deployment-old456-def        1/1     Running             0          10m
# nginx-deployment-old456-ghi        1/1     Running             0          10m
# nginx-deployment-old456-jkl        1/1     Running             0          10m

# Old Pods still running! (maxUnavailable: 0)
# → Application still available ✓

# Fix: Rollback
kubectl rollout undo deployment/nginx-deployment

# Or fix image
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
```

---

### Issue 3: Deployment Doesn't Update Pods

```bash
# Update Deployment
kubectl edit deployment nginx-deployment
# Changed some config

# But Pods don't restart!

# Reason: Only template changes trigger rollout
# Changes to replicas, labels, etc. don't restart Pods

# Template changes that trigger rollout:
✓ Image change
✓ Env vars change
✓ Resource limits change
✓ Volume mounts change
✓ Command/args change

# Non-template changes (no rollout):
✗ Replicas change (scales, doesn't restart)
✗ Deployment labels (metadata, not template)
✗ Strategy change (doesn't affect running Pods)

# Force rollout:
kubectl rollout restart deployment/nginx-deployment
# Triggers rolling restart of all Pods
```

---

### Issue 4: Too Many ReplicaSets

```bash
$ kubectl get rs

# Output: 15 old ReplicaSets!
# NAME                          DESIRED   CURRENT   READY   AGE
# nginx-deployment-abc123       0         0         0       30d
# nginx-deployment-def456       0         0         0       29d
# nginx-deployment-ghi789       0         0         0       28d
# ... 12 more old ReplicaSets ...
# nginx-deployment-xyz999       3         3         3       1d  ← Current

# Reason: revisionHistoryLimit not set (default 10)

# Fix: Set limit
kubectl edit deployment nginx-deployment

# Add/update:
spec:
  revisionHistoryLimit: 3  # Keep only 3 old ReplicaSets

# Or in YAML:
apiVersion: apps/v1
kind: Deployment
spec:
  revisionHistoryLimit: 3
```

---

## 🎓 Kiểm Tra Hiểu Biết

**1. Deployment vs ReplicaSet - khác nhau gì?**
<details>
<summary>Xem đáp án</summary>

**ReplicaSet:**
- Low-level controller
- Only maintains replica count
- No update management
- No rollback
- Directly manages Pods

**Deployment:**
- High-level controller
- Manages ReplicaSets
- Rolling updates
- Automatic rollback
- Revision history
- Declarative updates

**Use:** Always use Deployment in production!
</details>

**2. maxSurge và maxUnavailable làm gì?**
<details>
<summary>Xem đáp án</summary>

**maxSurge:**
- Max Pods above desired during update
- Example: replicas=3, maxSurge=1 → Max 4 Pods during update
- Higher = Faster rollout (more resources)

**maxUnavailable:**
- Max Pods below desired during update
- Example: replicas=3, maxUnavailable=0 → Always 3 Pods available
- 0 = Zero downtime guarantee

**Balance:**
- maxSurge=1, maxUnavailable=0: Safe, zero downtime
- maxSurge=50%, maxUnavailable=50%: Fast, more risk
</details>

**3. Khi nào dùng Recreate strategy?**
<details>
<summary>Xem đáp án</summary>

Use Recreate when:
- Can't run old và new versions simultaneously
- Database schema changes required
- Shared resources conflict
- Downtime is acceptable

Examples:
- Stateful databases
- Applications với breaking changes
- Resource-constrained environments

Don't use for:
- Stateless web apps (use RollingUpdate)
- Production với zero downtime requirement
</details>

---

## 💪 Bài Tập Thực Hành

### Bài 1: Complete Deployment Lifecycle

```yaml
# app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

```bash
# 1. Deploy v1.21
kubectl apply -f app-deployment.yaml

# 2. Verify
kubectl get deploy,rs,pods

# 3. Update to v1.22
kubectl set image deployment/webapp nginx=nginx:1.22 --record

# 4. Watch rollout
kubectl rollout status deployment/webapp

# 5. Verify new version
kubectl describe deployment webapp | grep Image

# 6. Update to v1.23
kubectl set image deployment/webapp nginx=nginx:1.23 --record

# 7. Check history
kubectl rollout history deployment/webapp

# 8. Rollback to v1.22
kubectl rollout undo deployment/webapp

# 9. Verify rolled back
kubectl describe deployment webapp | grep Image

# 10. Cleanup
kubectl delete deployment webapp
```

---

### Bài 2: Test Different Strategies

```bash
# Create Deployment với RollingUpdate
kubectl create deployment test-rolling --image=nginx:1.21 --replicas=5

# Watch Pods trong terminal 1
kubectl get pods -w -l app=test-rolling

# Update in terminal 2
kubectl set image deployment/test-rolling nginx=nginx:1.22

# Observe: Gradual rollout

# Now test Recreate
kubectl create deployment test-recreate --image=nginx:1.21 --replicas=5

# Change strategy
kubectl patch deployment test-recreate -p '{"spec":{"strategy":{"type":"Recreate"}}}'

# Watch Pods
kubectl get pods -w -l app=test-recreate

# Update
kubectl set image deployment/test-recreate nginx=nginx:1.22

# Observe: All Pods terminate, then new Pods create (downtime!)

# Cleanup
kubectl delete deployment test-rolling test-recreate
```

---

## 🎯 Key Takeaways

1. **Deployment = Production Standard**
   - Always use for stateless apps
   - ReplicaSet + Updates + Rollbacks
   - Declarative management

2. **Rolling Update = Zero Downtime**
   - Gradual rollout (scale up new, scale down old)
   - maxSurge và maxUnavailable control speed/risk
   - Always have available Pods

3. **Rollback = Safety Net**
   - Easy rollback: kubectl rollout undo
   - Revision history tracked
   - Can rollback to specific revision

4. **Two Strategies**
   - RollingUpdate: Gradual, zero downtime (default)
   - Recreate: Terminate all, then create (downtime)

5. **Best Practices**
   - Use --record for change tracking
   - Set appropriate maxSurge/maxUnavailable
   - Configure revisionHistoryLimit
   - Add readiness probes
   - Test rollbacks in staging

---

## 🚀 Tiếp Theo

Deployment mastered! Next: StatefulSets cho stateful applications!

**Next:** [4.3. StatefulSet →](./03-statefulset.md)

---

[⬅️ 4.1. ReplicaSet](./01-replicaset.md) | [🏠 Mục Lục](../README.md) | [📂 Phần 4: Workloads](./README.md) | [➡️ 4.3. StatefulSet](./03-statefulset.md)
