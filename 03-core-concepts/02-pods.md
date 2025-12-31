# 3.2. Pod - Đơn Vị Cơ Bản Nhất

> Hiểu Pod - building block của mọi thứ trong Kubernetes

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **Pod là gì** và **tại sao cần Pod**
- ✅ Phân biệt **Single-container vs Multi-container Pod**
- ✅ Hiểu **Pod lifecycle** đầy đủ
- ✅ **Tạo và quản lý Pods** với kubectl
- ✅ **Debug Pods** khi có vấn đề

---

## 📦 Pod Là Gì?

### Định Nghĩa

**Pod** = Đơn vị nhỏ nhất và cơ bản nhất trong Kubernetes, chứa một hoặc nhiều containers.

### TẠI SAO Cần Pod? (Không Chỉ Container?)

**Vấn đề:** Containers cần shared context

```
Scenario: Logging Sidecar Pattern

Main App Container:
├── Writes logs → /var/log/app.log
└── Needs logs shipped to central storage

Logging Container:
├── Reads from /var/log/app.log
├── Ships to Elasticsearch
└── Cần share filesystem với App!

→ Cần cách để 2 containers:
  1. Share volumes
  2. Share network (localhost)
  3. Lifecycle cùng nhau
  
→ POD giải quyết vấn đề này!
```

### Pod = Wrapper cho Container(s)

```
KHÔNG CÓ POD (không thể trong K8s):
[Container1]    [Container2]
   ↓ ❌           ↓ ❌
Không share được volumes/network

VỚI POD:
┌──────────────────────────────┐
│          POD                 │
│  ┌──────────┐ ┌──────────┐  │
│  │Container1│ │Container2│  │
│  └──────────┘ └──────────┘  │
│         ↓           ↓        │
│    Shared: volumes,          │
│           network,            │
│           IPC, etc.          │
└──────────────────────────────┘
```

---

## 🏗️ Pod Anatomy

### Pod Structure

```
┌─────────────────────────────────────────────────────┐
│                     POD                             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Pod Metadata:                                      │
│  ├─ Name: webapp-abc123                            │
│  ├─ Namespace: production                          │
│  ├─ Labels: {app: webapp, version: v1}            │
│  └─ UID: 12345678-1234-1234-1234-123456789012     │
│                                                     │
│  Pod IP: 10.244.1.5                                │
│  Node: worker-1                                     │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │  Init Containers (optional)                   │ │
│  │  ┌─────────────────┐                          │ │
│  │  │ init-1          │ Run first, sequential    │ │
│  │  │ (setup/migrate) │                          │ │
│  │  └─────────────────┘                          │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │  Application Containers                       │ │
│  │  ┌─────────────────┐  ┌──────────────────┐   │ │
│  │  │ nginx           │  │ log-shipper      │   │ │
│  │  │ Port: 80        │  │ (sidecar)        │   │ │
│  │  │ CPU: 500m       │  │ CPU: 100m        │   │ │
│  │  │ RAM: 256Mi      │  │ RAM: 128Mi       │   │ │
│  │  └─────────────────┘  └──────────────────┘   │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  Shared Resources:                                  │
│  ┌───────────────────────────────────────────────┐ │
│  │  Volumes:                                     │ │
│  │  ├─ /var/log → emptyDir (shared logs)        │ │
│  │  └─ /app/config → configMap                  │ │
│  │                                               │ │
│  │  Network:                                     │ │
│  │  └─ localhost (containers can talk)          │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 1️⃣ Single-Container Pod

### Ví Dụ: Simple Web Server

**YAML:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: dev
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
      name: http
    resources:
      requests:
        cpu: 100m      # 0.1 CPU core
        memory: 128Mi  # 128 MiB RAM
      limits:
        cpu: 500m      # Max 0.5 CPU
        memory: 256Mi  # Max 256 MiB RAM
    env:
    - name: ENVIRONMENT
      value: "development"
```

**Tạo Pod:**

```bash
# Save YAML to file
cat > nginx-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
EOF

# Create Pod
kubectl apply -f nginx-pod.yaml

# Output:
# pod/nginx-pod created

# Check Pod status
kubectl get pods

# Output:
# NAME        READY   STATUS    RESTARTS   AGE
# nginx-pod   1/1     Running   0          30s

# Get detailed info
kubectl get pods -o wide

# Output:
# NAME        READY   STATUS    RESTARTS   AGE   IP           NODE       
# nginx-pod   1/1     Running   0          1m    10.244.1.5   worker-1
```

---

## 2️⃣ Multi-Container Pod

### Pattern: Sidecar Container

**Use case:** Application + Log Shipper

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-logging
spec:
  # Shared volume
  volumes:
  - name: shared-logs
    emptyDir: {}
  
  containers:
  # Main application container
  - name: webapp
    image: myapp:v1
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
    # App writes logs to /var/log/app/app.log
  
  # Sidecar: Log shipping container
  - name: log-shipper
    image: fluent-bit:latest
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
      readOnly: true
    # Reads logs from /var/log/app/app.log
    # Ships to Elasticsearch
```

**Workflow:**

```
┌──────────────────────────────────────────────┐
│  Pod: webapp-with-logging                    │
├──────────────────────────────────────────────┤
│                                              │
│  ┌────────────────┐   ┌──────────────────┐  │
│  │  webapp        │   │  log-shipper     │  │
│  │                │   │                  │  │
│  │  writes logs   │   │  reads logs      │  │
│  │     ↓          │   │     ↓            │  │
│  └────────────────┘   └──────────────────┘  │
│         ↓                      ↓             │
│    /var/log/app/app.log (shared volume)     │
│                                ↓             │
│                          Elasticsearch       │
└──────────────────────────────────────────────┘

Benefits:
✓ Separation of concerns
✓ Reusable log-shipper
✓ App không cần biết shipping logic
```

### Pattern: Ambassador Container

**Use case:** Proxy requests to external service

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-proxy
spec:
  containers:
  # Main application
  - name: app
    image: myapp:v1
    # App calls: localhost:5432 (thinks it's DB)
  
  # Ambassador: Proxy to external DB
  - name: db-proxy
    image: cloud-sql-proxy:latest
    command:
    - /cloud_sql_proxy
    - -instances=project:region:instance=tcp:5432
    # Listens on localhost:5432
    # Forwards to Cloud SQL
```

**Network sharing trong Pod:**

```
Pod: app-with-proxy
IP: 10.244.1.5

Container: app
├── localhost:5432 → db-proxy container
└── External network: Pod IP 10.244.1.5

Container: db-proxy
├── Listen on: localhost:5432
├── Forward to: Cloud SQL (encrypted)
└── External network: Pod IP 10.244.1.5

✓ Both containers share localhost
✓ Both share same Pod IP
✓ Can communicate via 127.0.0.1
```

---

## 🔄 Pod Lifecycle

### Pod Phases

```
PENDING → RUNNING → SUCCEEDED/FAILED
   ↓         ↓            ↓
Waiting   Container   Job completed
          Running     or crashed
```

**Detailed lifecycle:**

```
1. PENDING
   Pod accepted by K8s
   ├─ Waiting for Node assignment (Scheduler)
   ├─ Pulling images
   └─ Creating containers

2. RUNNING
   Pod bound to Node
   ├─ At least 1 container is running
   ├─ Starting
   └─ Restarting (if failed)

3. SUCCEEDED
   All containers terminated successfully
   └─ Exit code 0
   └─ Will NOT restart

4. FAILED
   All containers terminated
   └─ At least 1 failed (non-zero exit code)
   └─ May restart based on restartPolicy

5. UNKNOWN
   Pod state cannot be determined
   └─ Usually: Node communication issue
```

### Container States

```
WAITING
├─ Container không start được
├─ Reasons: ImagePullBackOff, CrashLoopBackOff
└─ Check: kubectl describe pod <name>

RUNNING
├─ Container executing
├─ PID: <process-id>
└─ Started At: <timestamp>

TERMINATED
├─ Container finished or crashed
├─ Exit Code: 0 (success) or non-zero (error)
├─ Reason: Completed, Error, OOMKilled
└─ Will restart if restartPolicy allows
```

### RestartPolicy

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  restartPolicy: Always  # Always | OnFailure | Never
  containers:
  - name: app
    image: myapp:v1
```

**Policies:**

```
Always (default):
├─ Always restart container khi terminate
├─ Use for: Long-running services
└─ Example: Web servers, APIs

OnFailure:
├─ Restart only if exit code != 0
├─ Use for: Jobs, batch processing
└─ Example: Data migrations

Never:
├─ Never restart
├─ Use for: One-time tasks
└─ Example: Database backups
```

---

## 🩺 Pod Health Checks

### Probes (Kiểm Tra Sức Khỏe)

**3 types of probes:**

```
1. Liveness Probe
   Question: "Is container alive?"
   If fail: Kill container, restart

2. Readiness Probe  
   Question: "Is container ready for traffic?"
   If fail: Remove from Service endpoints

3. Startup Probe
   Question: "Has container started?"
   If fail: Keep retrying, then kill if timeout
```

### Liveness Probe Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: app
    image: myapp:v1
    ports:
    - containerPort: 8080
    
    livenessProbe:
      httpGet:
        path: /healthz        # Health check endpoint
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 15 # Wait 15s after start
      periodSeconds: 10       # Check every 10s
      timeoutSeconds: 5       # Timeout after 5s
      failureThreshold: 3     # Fail after 3 attempts
      successThreshold: 1     # Success after 1 attempt
```

**Liveness Probe types:**

```yaml
# 1. HTTP GET
livenessProbe:
  httpGet:
    path: /health
    port: 8080

# 2. TCP Socket
livenessProbe:
  tcpSocket:
    port: 3306

# 3. Exec Command
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
```

### Readiness Probe Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: app
    image: myapp:v1
    ports:
    - containerPort: 8080
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```

**Liveness vs Readiness:**

```
Scenario: Pod starting up

0-10s: Container starting
       Liveness: Not checked yet (initialDelay)
       Readiness: Not checked yet

10-30s: App warming up (loading cache, DB connections)
        Liveness: ✓ Pass (app alive)
        Readiness: ✗ Fail (not ready for traffic yet)
        → Pod NOT in Service endpoints

30s+: App ready
      Liveness: ✓ Pass
      Readiness: ✓ Pass
      → Pod added to Service endpoints
      → Receiving traffic!

During operation:
├─ Liveness fail → Kill & restart Pod
└─ Readiness fail → Stop sending traffic (but don't kill)
```

---

## 🎮 Hands-On: Working với Pods

### Create Pod

**Method 1: Imperative (command line)**

```bash
# Create Pod directly
kubectl run nginx --image=nginx

# With port
kubectl run nginx --image=nginx --port=80

# With labels
kubectl run nginx --image=nginx --labels="app=nginx,env=dev"

# With env vars
kubectl run nginx --image=nginx --env="ENV=production"

# Dry-run (không tạo, chỉ show YAML)
kubectl run nginx --image=nginx --dry-run=client -o yaml
```

**Method 2: Declarative (YAML file) - RECOMMENDED**

```bash
# Create YAML
cat > my-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: myapp
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
EOF

# Apply
kubectl apply -f my-pod.yaml

# Update (edit file, then)
kubectl apply -f my-pod.yaml
```

### Get Pods

```bash
# List all Pods
kubectl get pods

# With more details
kubectl get pods -o wide

# With labels
kubectl get pods --show-labels

# Filter by label
kubectl get pods -l app=nginx
kubectl get pods -l app=nginx,env=dev

# All namespaces
kubectl get pods --all-namespaces
kubectl get pods -A

# Watch mode (live updates)
kubectl get pods --watch
kubectl get pods -w

# Output formats
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o jsonpath='{.items[0].status.podIP}'
```

### Describe Pod

```bash
# Detailed information
kubectl describe pod nginx-pod

# Output includes:
# - Name, Namespace, Node
# - Labels, Annotations
# - Status, IP
# - Containers (image, ports, resources)
# - Conditions
# - Volumes
# - Events (VERY USEFUL for debugging!)
```

### Pod Logs

```bash
# View logs
kubectl logs nginx-pod

# Follow logs (like tail -f)
kubectl logs -f nginx-pod

# Previous container logs (if crashed)
kubectl logs nginx-pod --previous

# Multi-container Pod (specify container)
kubectl logs nginx-pod -c nginx
kubectl logs nginx-pod -c log-shipper

# Last 100 lines
kubectl logs nginx-pod --tail=100

# Logs since 1 hour ago
kubectl logs nginx-pod --since=1h

# Timestamps
kubectl logs nginx-pod --timestamps
```

### Execute Commands trong Pod

```bash
# Run command
kubectl exec nginx-pod -- ls /usr/share/nginx/html

# Interactive shell
kubectl exec -it nginx-pod -- /bin/bash

# Now you're inside container!
root@nginx-pod:/# ls
root@nginx-pod:/# ps aux
root@nginx-pod:/# curl localhost
root@nginx-pod:/# exit

# Multi-container: specify container
kubectl exec -it my-pod -c nginx -- /bin/bash
```

### Port Forward

```bash
# Forward local port to Pod
kubectl port-forward nginx-pod 8080:80

# Output:
# Forwarding from 127.0.0.1:8080 -> 80
# Forwarding from [::1]:8080 -> 80

# Now access: http://localhost:8080
# (in another terminal or browser)

curl http://localhost:8080
# → Response from Pod!

# Stop with Ctrl+C
```

### Delete Pod

```bash
# Delete by name
kubectl delete pod nginx-pod

# Delete from file
kubectl delete -f my-pod.yaml

# Delete with label selector
kubectl delete pods -l app=nginx

# Delete all Pods in namespace
kubectl delete pods --all

# Force delete (immediate)
kubectl delete pod nginx-pod --force --grace-period=0
```

---

## 🐛 Troubleshooting Pods

### Common Issues

**Issue 1: ImagePullBackOff**

```bash
$ kubectl get pods
NAME    READY   STATUS             RESTARTS   AGE
mypod   0/1     ImagePullBackOff   0          2m

# Debug
$ kubectl describe pod mypod

# Events:
# Failed to pull image "nginx:wrongtag": rpc error: code = Unknown 
# desc = Error response from daemon: manifest for nginx:wrongtag not found

# Fixes:
1. Check image name/tag spelling
2. Check image exists in registry
3. Check imagePullSecrets if private registry
```

**Issue 2: CrashLoopBackOff**

```bash
$ kubectl get pods
NAME    READY   STATUS             RESTARTS   AGE
mypod   0/1     CrashLoopBackOff   5          5m

# Container keeps crashing and restarting

# Debug steps:
# 1. Check logs
kubectl logs mypod
kubectl logs mypod --previous  # Logs from crashed container

# 2. Describe Pod
kubectl describe pod mypod
# Look at: Last State, Exit Code, Reason

# 3. Check events
kubectl get events --field-selector involvedObject.name=mypod

# Common causes:
# - Application error (check logs)
# - Missing environment variables
# - Liveness probe too aggressive
# - OOMKilled (out of memory)
```

**Issue 3: Pending**

```bash
$ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
mypod   0/1     Pending   0          10m

# Pod not scheduled to any Node

# Debug:
$ kubectl describe pod mypod

# Events might show:
# "0/3 nodes are available: insufficient cpu"
# "0/3 nodes are available: node(s) had taints"
# "persistentvolumeclaim not found"

# Fixes:
# 1. Add more nodes or reduce resource requests
# 2. Check node taints and add tolerations
# 3. Create PersistentVolume or PVC
```

**Issue 4: OOMKilled**

```bash
$ kubectl describe pod mypod

# Last State:
#   Terminated
#     Reason: OOMKilled
#     Exit Code: 137

# Container killed by Out Of Memory killer

# Fix: Increase memory limits
spec:
  containers:
  - name: app
    resources:
      limits:
        memory: 1Gi  # Increased from 512Mi
```

### Debugging Commands Cheat Sheet

```bash
# Get Pod status
kubectl get pods
kubectl get pods -o wide
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl logs <pod-name> -c <container-name>

# Execute commands
kubectl exec <pod-name> -- <command>
kubectl exec -it <pod-name> -- /bin/bash

# Check events
kubectl get events
kubectl get events --field-selector involvedObject.name=<pod-name>

# Check resource usage
kubectl top pod <pod-name>

# Port forward for testing
kubectl port-forward <pod-name> <local-port>:<pod-port>

# Debug with ephemeral container (K8s 1.23+)
kubectl debug <pod-name> -it --image=busybox

# Copy files
kubectl cp <pod-name>:/path/to/file ./local-file
kubectl cp ./local-file <pod-name>:/path/to/file
```

---

## 🎓 Kiểm Tra Hiểu Biết

**1. Tại sao cần Pod thay vì chỉ Container?**
<details>
<summary>Xem đáp án</summary>

Pod provides shared context cho containers:
- **Shared network**: localhost communication
- **Shared volumes**: File sharing
- **Shared IPC**: Inter-process communication
- **Lifecycle together**: Start/stop cùng nhau

Use cases:
- Sidecar pattern (logging, monitoring)
- Ambassador pattern (proxy)
- Adapter pattern (standardize output)
</details>

**2. Liveness Probe vs Readiness Probe?**
<details>
<summary>Xem đáp án</summary>

**Liveness Probe:**
- Question: "Is container alive?"
- Fail action: Kill & restart container
- Use case: Detect deadlocks, hung processes

**Readiness Probe:**
- Question: "Is container ready for traffic?"
- Fail action: Remove from Service (but keep running)
- Use case: Warm-up period, temporary unavailability

Example:
- App starting: Liveness ✓, Readiness ✗ (warming up)
- App overloaded: Liveness ✓, Readiness ✗ (stop traffic)
- App deadlock: Liveness ✗ (restart), Readiness N/A
</details>

**3. Multi-container Pod patterns?**
<details>
<summary>Xem đáp án</summary>

**1. Sidecar:**
- Main + Helper container
- Example: App + Log shipper
- Share: Logs volume

**2. Ambassador:**
- Main + Proxy container
- Example: App + DB proxy
- Share: Network (localhost)

**3. Adapter:**
- Main + Adapter container
- Example: App + Metrics formatter
- Share: Data transformation

All patterns leverage Pod's shared context!
</details>

---

## 💪 Bài Tập Thực Hành

### Bài 1: Create và Manage Simple Pod

```bash
# 1. Create Pod
kubectl run my-nginx --image=nginx

# 2. Check status
kubectl get pods

# 3. Get Pod IP
kubectl get pods -o wide

# 4. Access Pod (port-forward)
kubectl port-forward my-nginx 8080:80

# (In browser: http://localhost:8080)

# 5. View logs
kubectl logs my-nginx

# 6. Execute command
kubectl exec my-nginx -- nginx -v

# 7. Interactive shell
kubectl exec -it my-nginx -- /bin/bash
# (Inside: ls, cat /etc/nginx/nginx.conf, exit)

# 8. Delete
kubectl delete pod my-nginx
```

### Bài 2: Pod với Health Checks

```yaml
# health-check-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80
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
```

```bash
# Apply
kubectl apply -f health-check-pod.yaml

# Watch Pod status
kubectl get pods -w

# Check probe status
kubectl describe pod webapp | grep -A 5 "Liveness\|Readiness"

# Delete
kubectl delete -f health-check-pod.yaml
```

### Bài 3: Debug CrashLoopBackOff

```yaml
# Create a Pod that will crash
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "exit 1"]  # Always fails!
```

```bash
# Create
kubectl apply -f crash-pod.yaml

# Watch it crash
kubectl get pods -w

# See CrashLoopBackOff after few seconds

# Debug:
kubectl describe pod crash-pod
kubectl logs crash-pod
kubectl logs crash-pod --previous

# Understanding:
# - Container exits with code 1 (error)
# - Kubernetes restarts it
# - Crash again
# - Backoff delay increases: 10s, 20s, 40s, 80s, 160s, max 5m

# Cleanup
kubectl delete pod crash-pod
```

---

## 🎯 Key Takeaways

1. **Pod = Smallest Unit**
   - Wraps 1+ containers
   - Shared network, volumes, lifecycle

2. **Single vs Multi-Container**
   - Single: Most common
   - Multi: Sidecar, Ambassador, Adapter patterns

3. **Pod Lifecycle**
   - Pending → Running → Succeeded/Failed
   - RestartPolicy: Always, OnFailure, Never

4. **Health Checks Critical**
   - Liveness: Kill if unhealthy
   - Readiness: Stop traffic if not ready
   - Startup: Initial health check

5. **kubectl Basics**
   - `get pods`: List
   - `describe pod`: Details
   - `logs`: View logs
   - `exec`: Run commands
   - `port-forward`: Local access

---

## 🚀 Tiếp Theo

Pods học xong! Next: Organization với Namespaces!

**Next:** [3.3. Namespaces →](./03-namespaces.md)

---

[⬅️ 3.1. Cluster & Nodes](./01-cluster-and-nodes.md) | [🏠 Mục Lục](../README.md) | [📂 Phần 3: Core Concepts](./README.md) | [➡️ 3.3. Namespaces](./03-namespaces.md)
