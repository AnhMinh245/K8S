# 4.1. ReplicaSet - Duy Trì Số Lượng Pods

> Đảm bảo desired số Pods luôn running

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **ReplicaSet là gì** và **TẠI SAO cần**
- ✅ Biết **cách ReplicaSet hoạt động** (reconciliation loop)
- ✅ Tạo và quản lý **ReplicaSets**
- ✅ Hiểu **relationship với Pods và Deployments**
- ✅ **Scale applications** với ReplicaSets
- ✅ **Troubleshoot** ReplicaSet issues

---

## 📦 ReplicaSet Là Gì?

### Định Nghĩa

**ReplicaSet** = Controller đảm bảo một số lượng cố định Pod replicas luôn chạy trong cluster.

### Giải Thích Bằng Ví Dụ

**ReplicaSet giống như Supervisor quản lý ca làm việc:**

```
🏢 NHÀHÀNG - CA SÁNG (ReplicaSet)

Quy định: Luôn phải có 3 nhân viên phục vụ
Supervisor (ReplicaSet Controller) continuously checks:

Scenario 1: Đủ người
  Actual: 3 nhân viên ✓
  Desired: 3 nhân viên ✓
  → No action needed

Scenario 2: Thiếu người (1 người ốm)
  Actual: 2 nhân viên ✗
  Desired: 3 nhân viên ✓
  → Supervisor gọi thêm 1 người
  → Actual: 3 nhân viên ✓

Scenario 3: Dư người (4 người đến)
  Actual: 4 nhân viên ✗
  Desired: 3 nhân viên ✓
  → Supervisor cho 1 người về
  → Actual: 3 nhân viên ✓
```

### ReplicaSet trong K8s

```
REPLICASET = MAINTAIN REPLICA COUNT

Desired State: 3 Pod replicas
┌────────────────────────────────────┐
│  ReplicaSet Controller             │
│  (Watching continuously)           │
└────────────────────────────────────┘
         ↓
    ┌────────┐
    │ Check  │
    │ Actual │
    │ vs     │
    │ Desired│
    └────┬───┘
         │
    ┌────┴─────┐
    │          │
    ↓          ↓
Actual < 3    Actual > 3
Create Pod    Delete Pod
```

---

## 🤔 TẠI SAO Cần ReplicaSet?

### Vấn Đề Chỉ Dùng Pods

**Problem: Pods không self-healing**

```bash
# Create a single Pod
kubectl run webapp --image=nginx

# Pod is running
$ kubectl get pods
NAME     READY   STATUS    RESTARTS   AGE
webapp   1/1     Running   0          1m

# Simulate crash (delete Pod)
$ kubectl delete pod webapp
pod "webapp" deleted

# Pod is gone forever!
$ kubectl get pods
No resources found in default namespace.

❌ Application down!
❌ Manual intervention required
❌ No automatic recovery
❌ Single point of failure
```

### Giải Pháp: ReplicaSet

**Solution: Automatic recovery + scaling**

```bash
# Create ReplicaSet với 3 replicas
$ kubectl apply -f replicaset.yaml

# 3 Pods running
$ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
webapp-abc   1/1     Running   0          1m
webapp-def   1/1     Running   0          1m
webapp-ghi   1/1     Running   0          1m

# Simulate crash (delete 1 Pod)
$ kubectl delete pod webapp-abc
pod "webapp-abc" deleted

# ReplicaSet immediately creates replacement!
$ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
webapp-def   1/1     Running   0          2m
webapp-ghi   1/1     Running   0          2m
webapp-jkl   1/1     Running   0          5s  ← NEW!

✓ Automatic recovery!
✓ Always 3 Pods running
✓ High availability
✓ No manual intervention
```

---

## 🏗️ ReplicaSet Components

### ReplicaSet Structure

```
┌─────────────────────────────────────────────────┐
│            REPLICASET                           │
├─────────────────────────────────────────────────┤
│                                                 │
│  Metadata:                                      │
│  ├─ Name: webapp-rs                            │
│  ├─ Namespace: default                         │
│  └─ Labels: {app: webapp}                      │
│                                                 │
│  Spec:                                          │
│  ┌──────────────────────────────────────────┐  │
│  │  1. replicas: 3                          │  │
│  │     How many Pods to maintain            │  │
│  └──────────────────────────────────────────┘  │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │  2. selector:                            │  │
│  │     matchLabels:                         │  │
│  │       app: webapp                        │  │
│  │     How to find Pods to manage           │  │
│  └──────────────────────────────────────────┘  │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │  3. template:                            │  │
│  │     metadata:                            │  │
│  │       labels:                            │  │
│  │         app: webapp                      │  │
│  │     spec:                                │  │
│  │       containers: [...]                  │  │
│  │     Pod template for creating new Pods   │  │
│  └──────────────────────────────────────────┘  │
│                                                 │
└─────────────────────────────────────────────────┘
         ↓ manages ↓
┌─────────────────────────────────────────────────┐
│  PODS (created from template)                   │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │ Pod 1   │  │ Pod 2   │  │ Pod 3   │        │
│  │app:     │  │app:     │  │app:     │        │
│  │webapp   │  │webapp   │  │webapp   │        │
│  └─────────┘  └─────────┘  └─────────┘        │
└─────────────────────────────────────────────────┘
```

---

## 📝 ReplicaSet YAML

### Basic Example

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: webapp-rs
  labels:
    app: webapp
    tier: frontend
spec:
  # 1. Số lượng Pods mong muốn
  replicas: 3
  
  # 2. Selector: Tìm Pods để manage
  selector:
    matchLabels:
      app: webapp
      tier: frontend
  
  # 3. Template: Tạo Pods mới
  template:
    metadata:
      labels:
        app: webapp        # MUST match selector!
        tier: frontend     # MUST match selector!
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
```

### YAML Breakdown

**1. replicas**
```yaml
spec:
  replicas: 3

Meaning: "Tôi muốn 3 Pods"
ReplicaSet sẽ ensure luôn có exactly 3 Pods
```

**2. selector**
```yaml
spec:
  selector:
    matchLabels:
      app: webapp
      tier: frontend

Meaning: "Manage Pods có labels này"
ReplicaSet counts Pods với matching labels
```

**3. template**
```yaml
spec:
  template:
    metadata:
      labels:
        app: webapp
        tier: frontend
    spec:
      containers: [...]

Meaning: "Template để tạo Pods mới"
When need new Pod → Create from this template
```

**⚠️ CRITICAL:** Template labels MUST match selector!

```yaml
# GOOD ✓
selector:
  matchLabels:
    app: webapp
template:
  metadata:
    labels:
      app: webapp  # Matches!

# BAD ✗
selector:
  matchLabels:
    app: webapp
template:
  metadata:
    labels:
      app: api  # Mismatch! Will fail!
```

---

## 🔄 ReplicaSet Reconciliation Loop

### ReplicaSet Hoạt Động Như Thế Nào

**Vòng lặp liên tục:**

```
┌─────────────────────────────────────────────┐
│  VÒNG LẶP REPLICASET CONTROLLER             │
└─────────────────────────────────────────────┘

Lần lặp (mỗi 30s default):

1. LẤY desired replicas
   ↓
   replicas: 3

2. ĐẾM Pods hiện tại (matching selector)
   ↓
   kubectl get pods -l app=webapp
   ↓
   Tìm thấy: 2 Pods

3. SO SÁNH desired vs actual
   ↓
   Desired: 3
   Actual: 2
   Chênh lệch: -1 (cần thêm 1 Pod)

4. HÀNH ĐỘNG
   ↓
   Tạo 1 Pod từ template

5. CHỜ Pod Running
   ↓
   Pod created → Pending → Running

6. LẶP LẠI
   ↓
   Lần lặp tiếp:
   Desired: 3
   Actual: 3
   → Không cần action ✓

Vòng lặp tiếp tục mãi mãi...
```

### Example Scenarios

**Scenario 1: Pod Crashes**

```
Initial state:
Desired: 3
Actual: 3 Pods running
✓ In sync

--- Pod 2 crashes ---

ReplicaSet detects (next loop):
Desired: 3
Actual: 2 Pods (Pod 1, Pod 3)
Difference: -1

Action:
→ Create 1 new Pod (Pod 4)

Final state:
Actual: 3 Pods (Pod 1, Pod 3, Pod 4)
✓ Recovered automatically!
```

**Scenario 2: Manual Scale Up**

```
Initial state:
Desired: 3
Actual: 3 Pods
✓ In sync

--- User scales to 5 ---
kubectl scale rs webapp-rs --replicas=5

ReplicaSet detects:
Desired: 5 (updated)
Actual: 3 Pods
Difference: -2

Action:
→ Create 2 new Pods

Final state:
Actual: 5 Pods
✓ Scaled up!
```

**Scenario 3: Manual Scale Down**

```
Initial state:
Desired: 5
Actual: 5 Pods
✓ In sync

--- User scales to 2 ---
kubectl scale rs webapp-rs --replicas=2

ReplicaSet detects:
Desired: 2 (updated)
Actual: 5 Pods
Difference: +3 (too many)

Action:
→ Delete 3 Pods (oldest first, by default)

Final state:
Actual: 2 Pods
✓ Scaled down!
```

---

## 🎮 Hands-On: Working với ReplicaSets

### Create ReplicaSet

**Method 1: YAML file (Recommended)**

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
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
# Create
kubectl apply -f replicaset.yaml

# Output:
# replicaset.apps/nginx-rs created

# Verify
kubectl get replicaset
# or shorter
kubectl get rs

# Output:
# NAME       DESIRED   CURRENT   READY   AGE
# nginx-rs   3         3         3       30s
```

**Method 2: kubectl create (less common)**

```bash
# Create ReplicaSet imperatively
kubectl create -f replicaset.yaml

# Difference: create vs apply
# create: Fails if exists
# apply: Updates if exists (declarative, recommended)
```

---

### Get ReplicaSets

```bash
# List all ReplicaSets
kubectl get rs

# Output:
# NAME       DESIRED   CURRENT   READY   AGE
# nginx-rs   3         3         3       5m

# With more details
kubectl get rs -o wide

# Output:
# NAME       DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES       SELECTOR
# nginx-rs   3         3         3       5m    nginx        nginx:1.21   app=nginx

# Show labels
kubectl get rs --show-labels
```

---

### Describe ReplicaSet

```bash
# Detailed information
kubectl describe rs nginx-rs

# Output (sample):
Name:         nginx-rs
Namespace:    default
Selector:     app=nginx
Labels:       app=nginx
Annotations:  <none>
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.21
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  2m    replicaset-controller  Created pod: nginx-rs-abc12
  Normal  SuccessfulCreate  2m    replicaset-controller  Created pod: nginx-rs-def34
  Normal  SuccessfulCreate  2m    replicaset-controller  Created pod: nginx-rs-ghi56
```

---

### List Pods Managed by ReplicaSet

```bash
# Get Pods với label selector
kubectl get pods -l app=nginx

# Output:
# NAME             READY   STATUS    RESTARTS   AGE
# nginx-rs-abc12   1/1     Running   0          5m
# nginx-rs-def34   1/1     Running   0          5m
# nginx-rs-ghi56   1/1     Running   0          5m

# Show owner reference (which ReplicaSet owns Pod)
kubectl get pods nginx-rs-abc12 -o yaml | grep -A 5 ownerReferences

# Output:
# ownerReferences:
# - apiVersion: apps/v1
#   kind: ReplicaSet
#   name: nginx-rs
#   uid: 12345678-1234-1234-1234-123456789012
```

---

### Scale ReplicaSet

**Method 1: kubectl scale**

```bash
# Scale up to 5 replicas
kubectl scale rs nginx-rs --replicas=5

# Output:
# replicaset.apps/nginx-rs scaled

# Verify
kubectl get rs nginx-rs

# Output:
# NAME       DESIRED   CURRENT   READY   AGE
# nginx-rs   5         5         5       10m

# Get Pods (now 5)
kubectl get pods -l app=nginx
```

**Method 2: Edit YAML**

```bash
# Edit ReplicaSet
kubectl edit rs nginx-rs

# In editor, change:
# spec:
#   replicas: 3  →  replicas: 7

# Save and exit
# ReplicaSet automatically scales to 7!

# Verify
kubectl get rs nginx-rs
# DESIRED: 7
```

**Method 3: Update YAML file và apply**

```yaml
# replicaset.yaml
spec:
  replicas: 10  # Changed from 3
```

```bash
# Apply changes
kubectl apply -f replicaset.yaml

# ReplicaSet scales to 10
kubectl get rs nginx-rs
```

---

### Test Self-Healing

```bash
# Get current Pods
kubectl get pods -l app=nginx

# Output:
# NAME             READY   STATUS    RESTARTS   AGE
# nginx-rs-abc12   1/1     Running   0          5m
# nginx-rs-def34   1/1     Running   0          5m
# nginx-rs-ghi56   1/1     Running   0          5m

# Delete one Pod
kubectl delete pod nginx-rs-abc12

# Immediately check
kubectl get pods -l app=nginx

# Output: ReplicaSet created new Pod!
# NAME             READY   STATUS    RESTARTS   AGE
# nginx-rs-def34   1/1     Running   0          5m
# nginx-rs-ghi56   1/1     Running   0          5m
# nginx-rs-jkl78   1/1     Running   0          2s  ← NEW!

# Always 3 Pods maintained!
```

---

### Delete ReplicaSet

**Option 1: Delete ReplicaSet AND Pods**

```bash
# Delete ReplicaSet (default: deletes Pods too)
kubectl delete rs nginx-rs

# Output:
# replicaset.apps "nginx-rs" deleted

# Pods are also deleted
kubectl get pods -l app=nginx
# No resources found
```

**Option 2: Delete ReplicaSet but KEEP Pods**

```bash
# Delete ReplicaSet without deleting Pods
kubectl delete rs nginx-rs --cascade=orphan

# ReplicaSet deleted, but Pods remain!
kubectl get rs
# No resources found

kubectl get pods -l app=nginx
# NAME             READY   STATUS    RESTARTS   AGE
# nginx-rs-abc12   1/1     Running   0          10m
# nginx-rs-def34   1/1     Running   0          10m
# nginx-rs-ghi56   1/1     Running   0          10m

# Pods are now orphaned (no longer managed)
# Must manually delete if needed
```

---

## ⚠️ ReplicaSet Label Matching

### Selector Hoạt Động Như Thế Nào

**ReplicaSet counts ALL Pods với matching labels trong namespace**

```bash
# Create ReplicaSet với 3 replicas
kubectl apply -f replicaset.yaml

# ReplicaSet creates 3 Pods
kubectl get pods -l app=nginx
# 3 Pods

# Manually create Pod với SAME labels
kubectl run manual-pod --image=nginx --labels="app=nginx"

# ReplicaSet sees 4 Pods với app=nginx!
kubectl get pods -l app=nginx
# 4 Pods

# ReplicaSet detects:
# Desired: 3
# Actual: 4
# Difference: +1 (too many!)

# ReplicaSet DELETES 1 Pod (random selection)
# Could be manual-pod or one of ReplicaSet Pods!

kubectl get pods -l app=nginx
# Back to 3 Pods
```

**⚠️ WARNING:** Don't manually create Pods với labels matching ReplicaSet selector!

---

## 🔗 ReplicaSet vs Deployment

### Relationship

```
DEPLOYMENT
    ↓ creates và manages
REPLICASET
    ↓ creates và manages
PODS

Trong thực tế:
❌ DON'T use ReplicaSets directly
✓ USE Deployments instead!
```

### Tại Sao Dùng Deployments?

```
ReplicaSet limitations:
❌ No rolling updates
❌ No rollback capability
❌ No update strategies
❌ Manual version management

Deployment features:
✓ Rolling updates
✓ Rollback to previous versions
✓ Update strategies (RollingUpdate, Recreate)
✓ Revision history
✓ Declarative updates

Deployment = ReplicaSet + Update Management
```

**Example:**

```yaml
# Create Deployment (recommended)
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

```bash
# Deployment creates ReplicaSet automatically
kubectl apply -f deployment.yaml

# Check Deployment
kubectl get deployments
# NAME     READY   UP-TO-DATE   AVAILABLE   AGE
# webapp   3/3     3            3           1m

# Check ReplicaSet (created by Deployment)
kubectl get rs
# NAME                DESIRED   CURRENT   READY   AGE
# webapp-7d4b7c9d8f   3         3         3       1m

# Check Pods (created by ReplicaSet)
kubectl get pods
# NAME                      READY   STATUS    RESTARTS   AGE
# webapp-7d4b7c9d8f-abc12   1/1     Running   0          1m
# webapp-7d4b7c9d8f-def34   1/1     Running   0          1m
# webapp-7d4b7c9d8f-ghi56   1/1     Running   0          1m
```

---

## 🐛 Troubleshooting ReplicaSets

### Issue 1: Pods Not Created

```bash
$ kubectl get rs
NAME       DESIRED   CURRENT   READY   AGE
webapp-rs  3         0         0       5m

# No Pods created!

# Describe ReplicaSet
$ kubectl describe rs webapp-rs

# Events might show:
# Error creating: Pod "webapp-rs-abc12" is invalid: 
# spec.containers[0].image: Required value

# Fix: Check Pod template in ReplicaSet
# - Image name correct?
# - Resource limits valid?
# - Volumes exist?
```

---

### Issue 2: Desired != Current

```bash
$ kubectl get rs
NAME       DESIRED   CURRENT   READY   AGE
webapp-rs  5         3         3       5m

# Stuck at 3, can't reach 5

# Possible causes:
# 1. Resource limits (cluster full)
kubectl describe nodes | grep -A 5 "Allocated resources"

# 2. Image pull errors
kubectl get pods -l app=webapp
# Look for ImagePullBackOff

# 3. Node selector mismatch
kubectl describe rs webapp-rs | grep "Node Selector"
```

---

### Issue 3: Pods Constantly Restarting

```bash
$ kubectl get pods -l app=webapp
NAME             READY   STATUS             RESTARTS   AGE
webapp-rs-abc    0/1     CrashLoopBackOff   5          5m

# ReplicaSet keeps creating Pods, they keep crashing

# Debug:
kubectl logs webapp-rs-abc
kubectl logs webapp-rs-abc --previous

kubectl describe pod webapp-rs-abc
# Check: Last State, Exit Code, Reason

# Common causes:
# - Application error
# - Missing environment variables
# - OOMKilled (increase memory limits)
```

---

### Issue 4: Too Many Pods

```bash
$ kubectl get rs
NAME       DESIRED   CURRENT   READY   AGE
webapp-rs  3         3         3       5m

$ kubectl get pods -l app=webapp
# Shows 6 Pods!?

# Reason: Multiple ReplicaSets với same selector!
kubectl get rs --show-labels
# Check if multiple ReplicaSets have same selector

# Or: Orphaned Pods với matching labels
kubectl get pods -l app=webapp -o yaml | grep ownerReferences
# Pods without owner = orphaned

# Fix: Delete orphaned Pods or redundant ReplicaSets
```

---

## 🎓 Kiểm Tra Hiểu Biết

**1. ReplicaSet làm gì khi Pod crashes?**
<details>
<summary>Xem đáp án</summary>

1. ReplicaSet controller detects Pod count < desired
2. Creates new Pod from template
3. Waits for Pod to be Running
4. Count matches desired → Done

Automatic, no human intervention!
</details>

**2. Selector labels và template labels phải match?**
<details>
<summary>Xem đáp án</summary>

YES! MUST match!

```yaml
# Correct
selector:
  matchLabels:
    app: webapp
template:
  metadata:
    labels:
      app: webapp  # Matches selector

# Wrong
selector:
  matchLabels:
    app: webapp
template:
  metadata:
    labels:
      app: api  # Doesn't match! Error!
```

If don't match → ReplicaSet creation fails with validation error.
</details>

**3. Nên dùng ReplicaSet hay Deployment?**
<details>
<summary>Xem đáp án</summary>

**USE DEPLOYMENT!**

Deployment = ReplicaSet + Updates + Rollbacks

ReplicaSet alone: Only maintains replica count
Deployment: Replica count + Rolling updates + Rollback + History

Exception: Chỉ dùng ReplicaSet directly if you need low-level control và manage updates manually.
</details>

---

## 💪 Bài Tập Thực Hành

### Bài 1: Create và Scale ReplicaSet

```yaml
# exercise-rs.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: webapp-rs
spec:
  replicas: 2
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
        image: nginx
        ports:
        - containerPort: 80
```

```bash
# 1. Create ReplicaSet
kubectl apply -f exercise-rs.yaml

# 2. Verify
kubectl get rs
kubectl get pods -l app=webapp

# 3. Scale to 5
kubectl scale rs webapp-rs --replicas=5

# 4. Verify
kubectl get pods -l app=webapp
# Should see 5 Pods

# 5. Scale down to 1
kubectl scale rs webapp-rs --replicas=1

# 6. Verify (4 Pods terminated)
kubectl get pods -l app=webapp

# 7. Cleanup
kubectl delete rs webapp-rs
```

---

### Bài 2: Test Self-Healing

```bash
# Use ReplicaSet from Bài 1 (3 replicas)
kubectl scale rs webapp-rs --replicas=3

# Get Pod names
kubectl get pods -l app=webapp

# Pick one Pod and delete it
kubectl delete pod webapp-rs-<random-hash>

# Immediately watch
kubectl get pods -l app=webapp -w

# You'll see:
# - Deleted Pod: Terminating
# - New Pod: ContainerCreating → Running

# Always 3 Pods maintained!
```

---

### Bài 3: Label Matching Experiment

```bash
# 1. Create ReplicaSet với 2 replicas
kubectl apply -f exercise-rs.yaml

# 2. Verify 2 Pods created
kubectl get pods -l app=webapp

# 3. Manually create Pod với SAME labels
kubectl run manual-pod --image=nginx --labels="app=webapp"

# 4. Check Pods (3 Pods!)
kubectl get pods -l app=webapp

# 5. Wait a few seconds...
# ReplicaSet detects 3 > 2, deletes 1 Pod!

kubectl get pods -l app=webapp
# Back to 2 Pods (manual-pod might be deleted!)

# 6. Cleanup
kubectl delete rs webapp-rs
kubectl delete pod manual-pod --ignore-not-found
```

---

## 🎯 Key Takeaways

1. **ReplicaSet = Maintain Replica Count**
   - Ensures desired number of Pods
   - Automatic self-healing
   - Continuous reconciliation loop

2. **Three Key Components**
   - replicas: Desired count
   - selector: How to find Pods
   - template: How to create Pods

3. **Selector = Template Labels**
   - MUST match!
   - Validation error if mismatch

4. **Self-Healing Automatic**
   - Pod crashes → New Pod created
   - No manual intervention
   - High availability

5. **Use Deployments Instead**
   - ReplicaSet = Low-level
   - Deployment = ReplicaSet + Updates
   - Production: Always use Deployments

---

## 🚀 Tiếp Theo

ReplicaSet nắm rồi! Next: Deployment - Production-ready workload management!

**Next:** [4.2. Deployment →](./02-deployment.md)

---

[⬅️ Phần 3: Core Concepts](../03-core-concepts/README.md) | [🏠 Mục Lục](../README.md) | [📂 Phần 4: Workloads](./README.md) | [➡️ 4.2. Deployment](./02-deployment.md)
