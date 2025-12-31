# 8.1. Self-Healing - Tự Phục Hồi

> Kubernetes tự động phát hiện và sửa lỗi mà không cần can thiệp thủ công

---

## 📖 Mục Lục

1. [Self-Healing là gì?](#-self-healing-là-gì)
2. [Control Loop - Vòng Lặp Điều Khiển](#-control-loop---vòng-lặp-điều-khiển)
3. [Scenario 1: Container Crash](#-scenario-1-container-crash)
4. [Scenario 2: Health Check Failed](#-scenario-2-health-check-failed)
5. [Scenario 3: Pod Deleted](#-scenario-3-pod-deleted)
6. [Scenario 4: Node Failure](#-scenario-4-node-failure)
7. [Scenario 5: OOMKilled](#-scenario-5-oomkilled)
8. [Scenario 6: Disk Pressure](#-scenario-6-disk-pressure)
9. [RestartPolicy](#-restartpolicy)
10. [Hands-on Labs](#-hands-on-labs)
11. [Troubleshooting](#-troubleshooting)
12. [Best Practices](#-best-practices)

---

## 🤔 Self-Healing là gì?

### Định nghĩa

**Self-Healing** là khả năng của Kubernetes tự động:
- 🔍 **Detect:** Phát hiện lỗi (container crash, node down, health check fail)
- 🔧 **Fix:** Tự động sửa lỗi (restart container, recreate Pod, reschedule)
- ✅ **Verify:** Đảm bảo trạng thái mong muốn đạt được

### Ví dụ thực tế

**❌ Không có Self-Healing (Traditional servers):**
```
3:00 AM - Server crash
3:05 AM - Monitoring alert
3:10 AM - On-call engineer wakes up ☕
3:20 AM - SSH vào server
3:30 AM - Debug logs
3:45 AM - Restart service
4:00 AM - Service back online

Total downtime: 60 minutes 🔥
Engineer sleep lost: 1 night 😴
```

**✅ Có Self-Healing (Kubernetes):**
```
3:00 AM - Container crash
3:00 AM - kubelet detects (instant!)
3:00 AM - Restart container (automatic!)
3:01 AM - Service back online ✅

Total downtime: 1 minute 🎉
Engineer sleep lost: 0 nights 😴✅
```

### So sánh

| Aspect | Traditional | Kubernetes Self-Healing |
|--------|-------------|-------------------------|
| **Detection** | Manual monitoring | Automatic (kubelet, controllers) |
| **Response time** | Minutes to hours | Seconds |
| **Human intervention** | Required | Not required |
| **Consistency** | Depends on engineer | Always same process |
| **Cost** | On-call engineers | Free (built-in) |

---

## 🔄 Control Loop - Vòng Lặp Điều Khiển

### Nguyên lý cơ bản

**Mọi controller trong K8s đều chạy vòng lặp này:**

```
┌─────────────────────────────────────────┐
│         CONTROL LOOP (Vòng lặp)         │
└─────────────────────────────────────────┘

loop forever {
  1️⃣  desired_state = read_from_api_server()
      // "Tôi muốn 3 replicas"
  
  2️⃣  current_state = observe_reality()
      // "Hiện tại chỉ có 2 Pods running"
  
  3️⃣  if (current_state != desired_state) {
        take_action_to_fix()
        // "Tạo thêm 1 Pod nữa!"
      }
  
  4️⃣  sleep(sync_interval)  // Default: 10s
}
```

### Ví dụ: ReplicaSet Controller

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web
spec:
  replicas: 3  # ← DESIRED STATE
```

**Timeline:**

```
00:00 - ReplicaSet created
        ├── Desired: 3 Pods
        └── Current: 0 Pods
        
00:00 - Controller loop iteration #1
        ├── Diff detected: 0 != 3
        └── Action: Create 3 Pods
        
00:05 - Pods created
        ├── Desired: 3 Pods
        └── Current: 3 Pods ✅
        
00:10 - Controller loop iteration #2
        └── No diff, do nothing
        
01:00 - Someone deletes 1 Pod!
        ├── Desired: 3 Pods
        └── Current: 2 Pods ❌
        
01:00 - Controller loop iteration #n
        ├── Diff detected: 2 != 3
        └── Action: Create 1 Pod
        
01:05 - New Pod created
        ├── Desired: 3 Pods
        └── Current: 3 Pods ✅
```

**Key insight:**
```
Self-Healing = Control Loop chạy liên tục + API Server làm source of truth
```

---

## 🔥 Scenario 1: Container Crash

### Kịch bản

**Application có bug, crash khi nhận request đặc biệt:**

```go
// Buggy code
func handleRequest(r *Request) {
  data := r.Body
  result := process(data)  // ← Panic if data == nil!
  return result
}
```

### Timeline chi tiết

```
00:00:00 - Pod running (container ID: abc123)
           └── Status: Running, Ready: True
           
00:00:15 - User sends malformed request
           └── data == nil
           
00:00:15 - Application panics!
           └── exit code: 1
           
00:00:15 - Container exits
           ├── kubelet detects immediately (waiting on container process)
           └── Status: Running → Terminated
           
00:00:15 - kubelet checks restartPolicy
           └── restartPolicy: Always → RESTART!
           
00:00:15 - kubelet starts new container (container ID: def456)
           ├── Pull image (if needed)
           ├── Create container
           └── Start container
           
00:00:20 - Container running again
           ├── Status: Running, Ready: True
           └── Restart count: 1
           
Total downtime: 5 seconds ✅
```

### Workflow diagram

```
┌────────────────────────────────────────────────┐
│           CONTAINER CRASH RECOVERY             │
└────────────────────────────────────────────────┘

Pod Running
    ↓
Container crashes (exit code != 0)
    ↓
kubelet detects container exit
    ↓
Check restartPolicy
    ├─ Always      → Restart
    ├─ OnFailure   → Restart (exit != 0)
    └─ Never       → Don't restart
    ↓
Pull image (if needed)
    ↓
Create new container
    ↓
Start container
    ↓
Run postStart hook (if configured)
    ↓
Health checks pass?
    ├─ YES → Status: Running, Ready: True
    └─ NO  → Keep restarting (exponential backoff)
```

### Exponential Backoff

**Nếu container liên tục crash:**

```
Crash #1 → Wait 0s   → Restart (immediate)
Crash #2 → Wait 10s  → Restart
Crash #3 → Wait 20s  → Restart
Crash #4 → Wait 40s  → Restart
Crash #5 → Wait 80s  → Restart
Crash #6 → Wait 160s → Restart
...
Max wait: 5 minutes (300s)
```

**Why?** Tránh "thundering herd" problem:
```
100 Pods cùng crash → cùng restart → cùng hit database
→ Database overwhelmed → crash again → infinite loop! 🔥
```

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: crashy-app
spec:
  containers:
  - name: app
    image: my-buggy-app:1.0
    restartPolicy: Always  # ← Tự động restart khi crash
```

```bash
# Deploy
kubectl apply -f crashy-app.yaml

# Simulate crash
kubectl exec crashy-app -- killall -9 app

# Watch restart
kubectl get pod crashy-app -w
```

**Output:**
```
NAME         READY   STATUS    RESTARTS   AGE
crashy-app   1/1     Running   0          1m
crashy-app   0/1     Error     0          1m5s   ← Container crashed
crashy-app   1/1     Running   1          1m10s  ← Restarted! (RESTARTS=1)
```

---

## 💊 Scenario 2: Health Check Failed

### Kịch bản

**Application stuck (deadlock, infinite loop) nhưng process vẫn running:**

```python
# App stuck in infinite loop
while True:
    try:
        result = database.query("SELECT * FROM huge_table")
        process(result)  # ← Takes 10 minutes!
    except:
        pass  # ← Swallow errors, keep "running"

# Container process: Running ✅
# Application: Completely stuck ❌
# Health endpoint: Timeout ❌
```

### Timeline

```
00:00 - Pod starts, app healthy
        └── Liveness probe: GET /health → 200 OK ✅
        
00:10 - App enters deadlock
        ├── Process: Still running
        └── Liveness probe: GET /health → Timeout ❌
        
00:20 - Liveness probe fails (attempt 1/3)
        └── kubelet: "Hmm, might be temporary..."
        
00:30 - Liveness probe fails (attempt 2/3)
        └── kubelet: "Still failing..."
        
00:40 - Liveness probe fails (attempt 3/3)
        └── kubelet: "OK, killing container!"
        
00:40 - kubelet kills container
        └── SIGTERM → wait 30s → SIGKILL
        
00:45 - Container terminated, restart
        └── New container starts
        
00:50 - App healthy again
        └── Liveness probe: GET /health → 200 OK ✅
        
Total stuck time: 50 seconds ✅
```

### Liveness Probe config

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: app
    image: my-web-app:1.0
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10  # ← Wait 10s after start
      periodSeconds: 10        # ← Check every 10s
      timeoutSeconds: 5        # ← 5s timeout
      failureThreshold: 3      # ← Fail 3 times → restart
      successThreshold: 1      # ← Success 1 time → healthy
```

**What happens:**
```
initialDelaySeconds (10s): Wait for app to start
    ↓
First probe at 10s → Success
    ↓
Second probe at 20s → Success
    ↓
Third probe at 30s → FAIL (timeout after 5s)
    ↓
Fourth probe at 40s → FAIL (attempt 2/3)
    ↓
Fifth probe at 50s → FAIL (attempt 3/3)
    ↓
Restart container! 🔄
```

---

## 🗑️ Scenario 3: Pod Deleted

### Kịch bản

**Admin vô tình xóa Pod:**

```bash
kubectl delete pod web-app-12345
```

**Hoặc Node drain:**

```bash
kubectl drain node-1 --ignore-daemonsets
```

### Timeline

```
00:00 - Deployment: web-app, replicas: 3
        ├── Pod web-app-aaa (Node 1)
        ├── Pod web-app-bbb (Node 2)
        └── Pod web-app-ccc (Node 3)
        
00:00 - Admin deletes Pod web-app-aaa
        └── kubectl delete pod web-app-aaa
        
00:00 - API Server marks Pod for deletion
        └── deletionTimestamp set
        
00:00 - kubelet on Node 1 detects
        ├── Send SIGTERM to container
        ├── Wait terminationGracePeriodSeconds (default 30s)
        └── Send SIGKILL if still running
        
00:01 - Pod deleted from API Server
        └── Current replicas: 2 (bbb, ccc)
        
00:01 - ReplicaSet Controller wakes up
        ├── Desired: 3 replicas
        ├── Current: 2 replicas
        └── Diff: Need 1 more Pod!
        
00:01 - Controller creates new Pod
        └── Pod web-app-ddd created
        
00:02 - Scheduler assigns to Node
        └── Pod web-app-ddd → Node 1
        
00:03 - kubelet on Node 1 starts Pod
        ├── Pull image
        ├── Create container
        └── Start container
        
00:08 - Pod web-app-ddd ready
        └── Current replicas: 3 (bbb, ccc, ddd) ✅
        
Total recovery time: 8 seconds ✅
```

### Graceful Shutdown

**Để Pod shutdown sạch sẽ (save data, close connections):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: app
    image: my-app:1.0
    lifecycle:
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          - |
            echo "Shutting down gracefully..."
            # Close connections
            curl -X POST localhost:8080/shutdown
            # Wait for requests to finish
            sleep 10
  terminationGracePeriodSeconds: 30  # ← Max wait time
```

**Workflow:**
```
kubectl delete pod web-app
    ↓
1. Pod.status.phase = Terminating
2. Removed from Service endpoints (no new traffic)
3. preStop hook executed
4. Wait up to terminationGracePeriodSeconds (30s)
5. SIGTERM sent to container
6. Wait 30s
7. SIGKILL if still running (force kill)
8. Pod deleted
```

---

## 🖥️ Scenario 4: Node Failure

### Kịch bản

**Node bị mất điện/network:**

```
Data center power outage
    ↓
Node 2 shutdown
    ↓
All Pods on Node 2 unreachable
```

### Timeline chi tiết

```
00:00 - Cluster healthy
        ├── Node 1: 10 Pods
        ├── Node 2: 10 Pods
        └── Node 3: 10 Pods
        
00:00 - Node 2 goes down (power outage)
        └── kubelet stops sending heartbeats
        
00:10 - Node Controller detects missing heartbeats
        └── Last heartbeat: 10s ago
        
00:40 - Node marked NotReady (after 40s)
        └── Node 2: Ready → NotReady
        
00:40 - Node Controller waits (default: 5 minutes)
        └── "Maybe temporary network issue..."
        
05:40 - Still NotReady, start eviction
        └── Node Controller marks all Pods for deletion
        
05:40 - ReplicaSet Controllers detect Pod deletion
        ├── Deployment A: 3 replicas, now 2
        ├── Deployment B: 5 replicas, now 3
        └── ...
        
05:40 - Controllers create replacement Pods
        └── Schedule to healthy Nodes (1 & 3)
        
05:45 - Scheduler assigns new Pods
        ├── Node 1: +5 Pods
        └── Node 3: +5 Pods
        
05:50 - kubelet starts Pods on Node 1 & 3
        └── Pull images, create containers
        
06:00 - All Pods running on healthy Nodes ✅
        ├── Node 1: 15 Pods
        └── Node 3: 15 Pods
        
Total downtime: 6 minutes
```

### Pod Eviction Settings

**Tùy chỉnh thời gian eviction:**

```bash
# kube-controller-manager flags
--pod-eviction-timeout=5m0s  # ← Default: 5 minutes
--node-monitor-grace-period=40s  # ← Mark NotReady after 40s
--node-monitor-period=5s  # ← Check every 5s
```

**Faster eviction cho dev clusters:**
```bash
--pod-eviction-timeout=1m0s  # ← 1 minute (faster recovery)
```

**Slower eviction cho production:**
```bash
--pod-eviction-timeout=10m0s  # ← 10 minutes (avoid false positives)
```

### PodDisruptionBudget

**Đảm bảo availability trong quá trình eviction:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2  # ← Always keep 2 Pods running
  selector:
    matchLabels:
      app: web-app
```

**What happens:**
```
Deployment: 3 replicas
Node failure → 1 Pod down
    ↓
Current: 2 replicas (meets minAvailable=2)
    ↓
Controller creates 1 new Pod
    ↓
New Pod starts → Current: 3 replicas ✅
```

---

## 💀 Scenario 5: OOMKilled

### Kịch bản

**Container sử dụng quá nhiều memory:**

```python
# Memory leak
data = []
while True:
    data.append("x" * 1024 * 1024)  # ← 1MB per iteration
    # Forget to clear data → memory grows infinitely!
```

### Timeline

```
00:00 - Pod starts
        └── Memory limit: 512Mi
        
00:10 - App uses 256Mi (50%)
        └── Normal
        
00:30 - Memory leak!
        └── App uses 400Mi (78%)
        
00:45 - App uses 500Mi (98%)
        └── Linux OOM killer watching...
        
00:50 - App tries to allocate more (520Mi > 512Mi limit)
        └── OOM killer: "STOP RIGHT THERE!" 💀
        
00:50 - Container killed (SIGKILL)
        ├── Reason: OOMKilled
        └── Exit code: 137
        
00:50 - kubelet restarts container
        └── Fresh start with 0Mi memory
        
00:55 - Container running again
        └── Memory: 100Mi (normal)
        
If memory leak persists → crash again!
```

### Memory limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
  - name: app
    image: my-app:1.0
    resources:
      requests:
        memory: "256Mi"  # ← Minimum guaranteed
      limits:
        memory: "512Mi"  # ← Maximum allowed (OOM if exceeded)
```

### Debug OOMKilled

```bash
# Check Pod status
kubectl get pod memory-demo
# NAME          READY   STATUS      RESTARTS   AGE
# memory-demo   0/1     OOMKilled   5          10m

# Describe Pod
kubectl describe pod memory-demo
# Last State:     Terminated
#   Reason:       OOMKilled
#   Exit Code:    137

# Check logs before crash
kubectl logs memory-demo --previous
```

**Fix:**
```yaml
resources:
  limits:
    memory: "1Gi"  # ← Increase limit
```

---

## 💾 Scenario 6: Disk Pressure

### Kịch bản

**Node disk đầy:**

```
Node disk: 100GB
Used: 95GB (logs, images, temp files)
Available: 5GB (< 10% threshold)
```

### Timeline

```
00:00 - Node disk: 80GB/100GB (80%)
        └── Normal
        
02:00 - Apps writing logs
        └── 85GB/100GB (85%)
        
04:00 - More logs, images pulled
        └── 90GB/100GB (90%)
        
06:00 - Critical! 95GB/100GB (95%)
        └── kubelet detects disk pressure
        
06:00 - Node marked with condition
        └── DiskPressure: True
        
06:00 - kubelet starts evicting Pods
        ├── Priority: BestEffort Pods first
        ├── Then: Burstable Pods
        └── Last: Guaranteed Pods
        
06:05 - Evicted 5 Pods
        └── Freed: 10GB disk space
        
06:05 - Disk: 85GB/100GB (85%)
        └── DiskPressure: False
        
06:10 - Evicted Pods rescheduled
        └── Scheduled to healthy Nodes ✅
```

### Disk pressure thresholds

```bash
# kubelet flags
--eviction-hard=nodefs.available<10%  # ← Hard eviction at 10%
--eviction-soft=nodefs.available<15%  # ← Soft eviction at 15%
--eviction-soft-grace-period=nodefs.available=2m  # ← Wait 2min
```

### Prevention

**1. Log rotation:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: app
    image: my-app:1.0
    volumeMounts:
    - name: logs
      mountPath: /var/log
  volumes:
  - name: logs
    emptyDir:
      sizeLimit: 1Gi  # ← Limit log size
```

**2. Image garbage collection:**
```bash
# kubelet flags
--image-gc-high-threshold=85  # ← Start GC at 85% disk usage
--image-gc-low-threshold=80   # ← Stop GC at 80% disk usage
```

**3. Regular cleanup:**
```bash
# Delete unused images
kubectl run cleanup --image=alpine --rm -it --restart=Never -- sh -c "
  docker image prune -a -f
"
```

---

## 🔄️ RestartPolicy

### 3 Options

**1. Always (Default):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  restartPolicy: Always  # ← Restart regardless of exit code
  containers:
  - name: app
    image: nginx
```

**Behavior:**
```
Exit code 0 (success) → Restart
Exit code 1 (failure) → Restart
OOMKilled → Restart
SIGKILL → Restart
```

**Use case:**
- ✅ Web servers (nginx, Apache)
- ✅ APIs (REST, gRPC)
- ✅ Long-running services

**2. OnFailure:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  restartPolicy: OnFailure  # ← Only restart on failure
  containers:
  - name: job
    image: my-batch-job:1.0
```

**Behavior:**
```
Exit code 0 (success) → DON'T restart (job done!)
Exit code 1 (failure) → Restart
```

**Use case:**
- ✅ Batch jobs (data processing, ETL)
- ✅ One-time tasks (database migration)

**3. Never:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-setup
spec:
  restartPolicy: Never  # ← Never restart
  containers:
  - name: setup
    image: my-init-script:1.0
```

**Behavior:**
```
Exit code 0 (success) → DON'T restart
Exit code 1 (failure) → DON'T restart
```

**Use case:**
- ✅ Init containers (setup, config)
- ✅ Debug Pods (troubleshooting)

### Comparison table

| RestartPolicy | Exit 0 | Exit 1 | Use case |
|---------------|--------|--------|----------|
| **Always** | Restart | Restart | Long-running services |
| **OnFailure** | ❌ | Restart | Batch jobs |
| **Never** | ❌ | ❌ | One-time tasks |

---

## 🧪 Hands-on Labs

### Lab 1: Container Crash

```bash
# Create crashy Pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: crashy
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "echo Starting...; sleep 10; exit 1"]
    # ← Crash after 10s
EOF

# Watch Pod
kubectl get pod crashy -w
```

**Expected:**
```
NAME     READY   STATUS    RESTARTS   AGE
crashy   1/1     Running   0          5s
crashy   0/1     Error     0          15s  ← Crashed!
crashy   1/1     Running   1          20s  ← Restarted!
crashy   0/1     Error     1          30s  ← Crashed again
crashy   1/1     Running   2          35s  ← Restarted again (RESTARTS=2)
```

**Check restart count:**
```bash
kubectl get pod crashy -o jsonpath='{.status.containerStatuses[0].restartCount}'
# Output: 2
```

### Lab 2: Liveness Probe

```bash
# Create unhealthy app
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: unhealthy
spec:
  containers:
  - name: app
    image: nginx
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy  # ← File doesn't exist!
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 2
EOF

# Watch Pod (will restart due to failed probe)
kubectl get pod unhealthy -w
```

**Expected:**
```
NAME        READY   STATUS    RESTARTS   AGE
unhealthy   1/1     Running   0          10s
unhealthy   1/1     Running   1          25s  ← Restarted! (probe failed)
```

**Fix by creating file:**
```bash
kubectl exec unhealthy -- touch /tmp/healthy
# Now probe succeeds! ✅
```

### Lab 3: Pod Deletion Recovery

```bash
# Create Deployment
kubectl create deployment web --image=nginx --replicas=3

# Watch Pods
kubectl get pods -l app=web -w &

# Delete one Pod
POD=$(kubectl get pods -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD

# Expected: New Pod created automatically!
```

**Output:**
```
NAME                   READY   STATUS    RESTARTS   AGE
web-12345-aaa          1/1     Running   0          1m
web-12345-bbb          1/1     Running   0          1m
web-12345-ccc          1/1     Running   0          1m
web-12345-aaa          1/1     Terminating   0      1m5s  ← Deleted
web-12345-ddd          0/1     Pending       0      0s    ← New Pod!
web-12345-ddd          1/1     Running       0      5s    ← Running!
```

### Lab 4: Simulate Node Failure

```bash
# Mark Node as unschedulable
kubectl cordon node-1

# Drain Node (evict Pods)
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# Watch Pods reschedule
kubectl get pods -o wide -w
```

**Expected:**
```
NAME      READY   STATUS    RESTARTS   AGE   NODE
web-aaa   1/1     Running   0          1m    node-1
web-aaa   1/1     Terminating  0       1m5s  node-1  ← Evicting
web-ddd   0/1     Pending      0       0s    <none>  ← New Pod
web-ddd   1/1     Running      0       5s    node-2  ← Rescheduled!
```

**Restore Node:**
```bash
kubectl uncordon node-1
```

---

## 🔧 Troubleshooting

### Issue 1: Pod stuck in CrashLoopBackOff

**Symptoms:**
```bash
kubectl get pods
# NAME    READY   STATUS             RESTARTS   AGE
# web     0/1     CrashLoopBackOff   10         15m
```

**Meaning:**
- Container keeps crashing
- kubelet backs off before restarting (exponential backoff)

**Debug:**
```bash
# Check logs
kubectl logs web
kubectl logs web --previous  # ← Logs before crash

# Describe Pod
kubectl describe pod web
# Last State:     Terminated
#   Reason:       Error
#   Exit Code:    1

# Check events
kubectl get events --sort-by='.lastTimestamp'
```

**Common causes:**

**A. Missing environment variables:**
```yaml
# Fix: Add required env vars
env:
- name: DATABASE_URL
  value: "postgres://..."
```

**B. Wrong command:**
```yaml
# Fix: Correct command
command: ["python", "app.py"]  # NOT ["python app.py"]
```

**C. Port conflict:**
```yaml
# Fix: Use different port
ports:
- containerPort: 8080  # NOT 80 (may be privileged)
```

### Issue 2: Liveness probe killing healthy container

**Symptoms:**
```bash
kubectl get pods
# NAME   READY   STATUS    RESTARTS   AGE
# web    1/1     Running   50         10m  ← Too many restarts!
```

**Cause:** Probe timing too aggressive

**Fix:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30  # ← Increase (was 5)
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 5  # ← Increase (was 3)
```

### Issue 3: Pods not rescheduling after Node failure

**Symptoms:**
```bash
kubectl get pods
# NAME   READY   STATUS    RESTARTS   AGE   NODE
# web    1/1     Unknown   0          10m   node-1  ← Node down
```

**Cause:** No controller managing Pods (bare Pod, not Deployment)

**Fix:**
```bash
# Delete stuck Pod
kubectl delete pod web --force --grace-period=0

# Create Deployment instead
kubectl create deployment web --image=nginx --replicas=3
```

---

## 💡 Best Practices

### 1. Always use controllers (Deployment, StatefulSet)

❌ **Bad:**
```yaml
apiVersion: v1
kind: Pod  # ← Bare Pod, no self-healing!
metadata:
  name: web
spec:
  containers:
  - name: app
    image: nginx
```

✅ **Good:**
```yaml
apiVersion: apps/v1
kind: Deployment  # ← Controller ensures replicas!
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: app
        image: nginx
```

### 2. Configure appropriate restartPolicy

```yaml
# Web servers
restartPolicy: Always

# Batch jobs
restartPolicy: OnFailure

# One-time tasks
restartPolicy: Never
```

### 3. Set resource limits (prevent OOMKilled)

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"  # ← Prevent unlimited memory usage
    cpu: "500m"
```

### 4. Implement health checks

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30  # ← Wait for app to start
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 5. Use PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2  # ← Always keep 2 Pods running
  selector:
    matchLabels:
      app: web
```

### 6. Run multiple replicas

```yaml
spec:
  replicas: 3  # ← Not 1! (single point of failure)
```

### 7. Spread Pods across Nodes

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web
```

---

## 🎓 Key Takeaways

### Concepts

1. **Self-Healing:** Tự động phát hiện và sửa lỗi mà không cần can thiệp thủ công
2. **Control Loop:** Liên tục so sánh desired state vs current state, tự động reconcile
3. **Container Crash:** kubelet tự động restart (based on restartPolicy)
4. **Health Check Failed:** Liveness probe fail → kubelet restart container
5. **Pod Deleted:** ReplicaSet Controller tạo Pod mới
6. **Node Failure:** Node Controller evict Pods, reschedule to healthy Nodes
7. **RestartPolicy:**
   - `Always`: Luôn restart (web servers)
   - `OnFailure`: Chỉ restart khi fail (batch jobs)
   - `Never`: Không restart (one-time tasks)

### Self-Healing Scenarios

| Scenario | Detection | Action | Recovery Time |
|----------|-----------|--------|---------------|
| Container crash | kubelet | Restart container | ~5s |
| Health check fail | Liveness probe | Restart container | ~30s |
| Pod deleted | ReplicaSet Controller | Create new Pod | ~10s |
| Node failure | Node Controller | Evict + reschedule | ~5min |
| OOMKilled | Linux OOM killer | Restart container | ~5s |
| Disk pressure | kubelet | Evict Pods | ~5min |

### Commands

```bash
# Watch Pod restart
kubectl get pod <name> -w

# Check restart count
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].restartCount}'

# View logs before crash
kubectl logs <name> --previous

# Describe Pod (see events)
kubectl describe pod <name>

# Delete Pod (test self-healing)
kubectl delete pod <name>

# Drain Node (simulate failure)
kubectl drain <node> --ignore-daemonsets

# Restore Node
kubectl uncordon <node>
```

---

**Chúc mừng!** Bạn đã hiểu về **Self-Healing** trong Kubernetes! 🎉

👉 [**8.2. Health Checks**](./02-health-checks.md)

---

[⬅️ 7.3. StorageClass](../07-storage/03-storage-classes.md) | [⬆️ Phần 8](./README.md) | [➡️ 8.2. Health Checks](./02-health-checks.md) | [🏠 Mục Lục](../README.md)

