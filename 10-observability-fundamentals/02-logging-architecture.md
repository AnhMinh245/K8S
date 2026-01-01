# 10.2. Logging Architecture trong Kubernetes

> Hiểu cách K8s handles logs và log collectors hoạt động

---

## 🎯 K8s Logging Model

```
┌────────────────────────────────────────┐
│         Application Container          │
│  • Writes to stdout/stderr             │
│  • Container runtime captures          │
└────────────────┬───────────────────────┘
                 ↓
┌────────────────────────────────────────┐
│      Container Runtime (containerd)    │
│  • Stores logs: /var/log/pods/        │
│  • JSON format with metadata           │
└────────────────┬───────────────────────┘
                 ↓
┌────────────────────────────────────────┐
│        kubelet (symlinks)              │
│  • /var/log/containers/                │
│  • Format: <pod>_<namespace>_<container>-<id>.log │
└────────────────┬───────────────────────┘
                 ↓
┌────────────────────────────────────────┐
│    Log Collector (DaemonSet)           │
│  • Fluentd, Filebeat, Datadog Agent    │
│  • Reads logs from /var/log/           │
│  • Enriches with K8s metadata          │
│  • Forwards to backend                 │
└────────────────────────────────────────┘
```

---

## 📁 1. Log Storage Locations

### Container Logs Path

```bash
# Logs stored on Node
/var/log/pods/<namespace>_<pod-name>_<pod-uid>/<container-name>/

# Example:
/var/log/pods/default_web-abc123_1234-5678/nginx/0.log
/var/log/pods/default_web-abc123_1234-5678/nginx/1.log (rotated)
```

### Symlinks for Easy Access

```bash
# kubelet creates symlinks
/var/log/containers/<pod>_<namespace>_<container>-<container-id>.log

# Example:
/var/log/containers/web-abc123_default_nginx-abc123def456.log
  → symlink to /var/log/pods/.../nginx/0.log
```

---

## 📝 2. Log Format

### JSON Format

**Container runtime writes logs in JSON:**

```json
{
  "log": "2024-01-15 10:30:00 INFO Request processed successfully\n",
  "stream": "stdout",
  "time": "2024-01-15T10:30:00.123456789Z"
}
```

**Multi-line logs:**
```json
{
  "log": "Exception in thread \"main\" java.lang.NullPointerException\n",
  "stream": "stderr",
  "time": "2024-01-15T10:30:01.123456789Z"
}
{
  "log": "    at com.example.Main.main(Main.java:10)\n",
  "stream": "stderr",
  "time": "2024-01-15T10:30:01.123456790Z"
}
```

---

## 🔧 3. Accessing Logs

### kubectl logs

```bash
# View logs
kubectl logs <pod-name>

# Specific container in multi-container Pod
kubectl logs <pod-name> -c <container-name>

# Follow logs (tail -f)
kubectl logs -f <pod-name>

# Previous container (after restart)
kubectl logs <pod-name> --previous

# Last N lines
kubectl logs <pod-name> --tail=100

# Since timestamp
kubectl logs <pod-name> --since=1h

# All Pods with label
kubectl logs -l app=web --all-containers=true
```

### Log Limitations

```
kubectl logs limitations:
  • Only current + previous container
  • No history beyond that
  • Lost if Node dies
  • No aggregation across Pods
  • No search/filter capabilities

→ Need centralized logging solution!
```

---

## 🚀 4. Log Collection Patterns

### Pattern 1: Node-Level Logging (DaemonSet)

**Pattern phổ biến nhất cho Datadog, Dynatrace, ELK:**

```yaml
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
        image: fluent/fluentd-kubernetes-daemonset:v1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

**How it works:**
```
1. DaemonSet deploys collector on every Node
2. Collector reads from /var/log/containers/
3. Parses JSON format
4. Enriches with K8s metadata (from API)
5. Forwards to backend (Datadog, Elasticsearch, etc.)
```

---

### Pattern 2: Sidecar Container

**Sidecar reads app logs and forwards:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  # Main application
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  
  # Sidecar log forwarder
  - name: log-forwarder
    image: fluent/fluentd:v1
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
      readOnly: true
  
  volumes:
  - name: logs
    emptyDir: {}
```

**Khi nào dùng:**
- App ghi logs vào files (không phải stdout)
- Need special parsing per app
- Different log destinations per app

**Cons:**
- More resource usage (sidecar per Pod)
- More complex than DaemonSet

---

### Pattern 3: Streaming Sidecar

**For multi-line logs that need special handling:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  
  # Streaming sidecar
  - name: log-streamer
    image: busybox
    command:
    - sh
    - -c
    - tail -f /var/log/app/application.log
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
      readOnly: true
  
  volumes:
  - name: logs
    emptyDir: {}
```

**Sidecar streams logs to its stdout → Captured by container runtime**

---

## 🏷️ 5. Log Metadata Enrichment

### K8s Metadata Added by Collectors

**Log collector enriches logs with K8s context:**

```json
Original log line:
  "Request processed successfully"

Enriched with K8s metadata:
{
  "log": "Request processed successfully",
  "kubernetes": {
    "pod_name": "web-abc123",
    "namespace_name": "production",
    "pod_id": "1234-5678-9abc",
    "labels": {
      "app": "web",
      "version": "v1.2.3",
      "environment": "prod"
    },
    "annotations": {
      "team": "platform"
    },
    "host": "node-1",
    "container_name": "nginx",
    "container_image": "nginx:1.21"
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**Query in Datadog:**
```
source:kubernetes
  namespace:production
  app:web
  version:v1.2.3
```

---

## 🔑 6. RBAC for Log Collectors

**ServiceAccount and permissions:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: log-collector
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: log-collector
rules:
# Get Pod metadata for enrichment
- apiGroups: [""]
  resources:
  - pods
  - namespaces
  verbs: ["get", "list", "watch"]

# Optional: Read events
- apiGroups: [""]
  resources:
  - events
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: log-collector
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: log-collector
subjects:
- kind: ServiceAccount
  name: log-collector
  namespace: logging
```

---

## 📊 7. Log Rotation & Retention

### Container Log Rotation

**kubelet automatically rotates logs:**

```yaml
# kubelet config
containerLogMaxSize: 10Mi   # Rotate after 10MB
containerLogMaxFiles: 5     # Keep 5 rotated files
```

**Example:**
```
/var/log/pods/.../nginx/0.log     (current, 10MB)
/var/log/pods/.../nginx/1.log     (rotated, 10MB)
/var/log/pods/.../nginx/2.log     (rotated, 10MB)
...
/var/log/pods/.../nginx/5.log     (cũ nhất, sẽ bị xóa)
```

### Node Disk Pressure

**If Node disk full:**
```
1. kubelet detects disk pressure
2. Marks Node as DiskPressure
3. Evicts Pods if necessary
4. Logs may be truncated

→ Important: Forward logs off-Node quickly!
```

---

## 🎯 8. Best Practices

### ✅ DO

**1. Log to stdout/stderr**
```python
# Good
print("Request processed")  # → stdout
import logging
logging.error("Error occurred")  # → stderr

# Bad (harder to collect)
with open('/var/log/app.log', 'w') as f:
    f.write("Log message")
```

**2. Structured logging (JSON)**
```json
// Good - Easy to parse
{"level":"info","msg":"Request processed","user_id":123,"duration_ms":45}

// Bad - Hard to parse
"INFO Request processed for user 123 in 45ms"
```

**3. Include request IDs**
```json
{"level":"info","request_id":"abc-123","msg":"Request started"}
{"level":"info","request_id":"abc-123","msg":"Query executed","duration":10}
{"level":"info","request_id":"abc-123","msg":"Request completed"}
```

**4. Set appropriate log levels**
```
ERROR: 5-10% of logs
WARN: 10-20%
INFO: 60-70%
DEBUG: Off in production
```

**5. Use consistent labels for filtering**
```yaml
metadata:
  labels:
    app: web
    environment: prod
    component: api
```

---

### ❌ DON'T

1. **Log sensitive data** (passwords, PII)
```python
# Bad
logging.info(f"User logged in: {username} / {password}")

# Good
logging.info(f"User logged in: {username}")
```

2. **Excessive logging** (every request details)
```python
# Bad - 1000 req/s = 1000 log lines/s
logging.debug(f"Processing request {req}")

# Good - Sample or aggregate
if random.random() < 0.01:  # 1% sampling
    logging.debug(f"Processing request {req}")
```

3. **Log to files inside container** without volume
```python
# Bad - Lost on container restart
logging.basicConfig(filename='/app/app.log')

# Good - stdout/stderr
logging.basicConfig(stream=sys.stdout)
```

4. **No log rotation** (disk fills up)

5. **Mixed stdout/stderr for normal logs**
```python
# Bad
print("Info message")  # stdout
print("Error!", file=sys.stderr)  # stderr
print("More info")  # stdout → Order mixed!

# Good - Use logging framework
logger.info("Info message")
logger.error("Error!")
```

---

## 🔍 9. Troubleshooting Logging

### Check Logs Not Appearing

**1. Verify app writes to stdout/stderr:**
```bash
kubectl exec -it <pod> -- sh
# Inside container
ls -la /proc/self/fd/
# 1 -> stdout, 2 -> stderr
```

**2. Check log files exist on Node:**
```bash
# On Node
ls -la /var/log/pods/
ls -la /var/log/containers/
```

**3. Check log collector is running:**
```bash
kubectl get daemonset -n logging
kubectl logs -n logging <log-collector-pod>
```

**4. Check RBAC permissions:**
```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:logging:log-collector
```

**5. Check log collector configuration:**
```bash
kubectl exec -it <log-collector-pod> -- cat /etc/fluent/fluent.conf
```

---

## 📈 10. Log Volume Estimation

**Calculate log volume for capacity planning:**

```
Example calculation:
  • 100 Pods
  • Mỗi Pod: 10 log lines/second
  • Mỗi line: 200 bytes trung bình
  • Retention: 7 days

Daily volume:
  100 pods × 10 lines/s × 200 bytes × 86400 seconds
  = 17.28 GB/day

Weekly retention:
  17.28 GB × 7 days = 120.96 GB

Add 50% buffer: ~180 GB storage needed
```

**Cost considerations:**
```
Datadog/Dynatrace pricing often based on:
  • GB ingested per day
  • Retention period
  • Number of indexed logs

→ Use sampling, filtering, aggregation to reduce cost!
```

---

## 🎓 Key Takeaways

1. **Logs:** App writes to stdout/stderr → Container runtime captures → /var/log/pods/
2. **kubectl logs:** Limited, need centralized logging
3. **DaemonSet pattern:** Phổ biến nhất cho log collection
4. **Metadata enrichment:** Labels/annotations → log tags
5. **RBAC:** Log collector needs read Pod/Namespace permissions
6. **Structured logging:** JSON format best for parsing
7. **Best practice:** Log to stdout, structured format, consistent labels

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. Container logs được lưu ở đâu trên Node?
2. kubectl logs có hạn chế gì?
3. DaemonSet log collector hoạt động như thế nào?
4. Tại sao nên log to stdout/stderr instead of files?
5. Log collector cần RBAC permissions gì?
6. Structured logging (JSON) có lợi ích gì?

---

## 🚀 Tiếp Theo

👉 [10.3. Labels & Annotations for Observability](./03-labels-annotations-observability.md)

---

[⬅️ 10.1. Metrics](./01-metrics-architecture.md) | [⬆️ Phần 10](./README.md) | [🏠 Mục Lục](../README.md)

