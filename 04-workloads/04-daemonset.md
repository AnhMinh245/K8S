# 4.4. DaemonSet - Chạy Trên Mọi Node

> DaemonSet đảm bảo 1 Pod chạy trên mỗi Node (hoặc một nhóm Nodes)

---

## 🎯 DaemonSet Là Gì?

**DaemonSet** = Controller chạy exactly 1 Pod trên mỗi Node

```
Cluster:
  Node 1 → DaemonSet tạo Pod-1
  Node 2 → DaemonSet tạo Pod-2
  Node 3 → DaemonSet tạo Pod-3

Add Node 4 → DaemonSet tự động tạo Pod-4
Remove Node 2 → Pod-2 tự động deleted
```

---

## 🏢 Ví Dụ Thực Tế

**DaemonSet = Camera an ninh tại mỗi tầng**

```
Tòa nhà 10 tầng:
  Tầng 1 → Camera 1
  Tầng 2 → Camera 2
  ...
  Tầng 10 → Camera 10

Yêu cầu: MỌI tầng phải có 1 camera

Thêm tầng 11 → Tự động lắp Camera 11
Phá bỏ tầng 5 → Camera 5 tự động removed
```

---

## 📝 DaemonSet YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  labels:
    app: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
          hostPort: 9100  # Expose on Node's port
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      tolerations:  # Run on master nodes too
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
```

---

## 🎯 Use Cases

### 1. Monitoring Agents

**Prometheus Node Exporter**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  template:
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter
        
Purpose:
  • Collect metrics from EVERY Node
  • CPU, memory, disk, network stats
  • No Node left unmonitored
```

---

### 2. Log Collectors

**Fluentd, Filebeat**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  template:
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log

Purpose:
  • Collect logs from all Nodes
  • Ship to central logging (ELK, Loki)
  • Every Node's logs captured
```

---

### 3. Network Plugins

**Calico, Weave, Flannel**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
spec:
  template:
    spec:
      hostNetwork: true
      containers:
      - name: calico-node
        image: calico/node:latest

Purpose:
  • Network driver on every Node
  • Pod-to-Pod communication
  • Network policies enforcement
```

---

### 4. Storage Daemons

**Ceph, GlusterFS**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ceph-osd
spec:
  template:
    spec:
      containers:
      - name: osd
        image: ceph/ceph:latest

Purpose:
  • Distributed storage daemon
  • Each Node contributes storage
```

---

### 5. Security Agents

**Antivirus, Intrusion Detection**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: security-agent
spec:
  template:
    spec:
      containers:
      - name: agent
        image: security/agent:latest

Purpose:
  • Security monitoring on all Nodes
  • Threat detection
  • Compliance scanning
```

---

## 🔄 DaemonSet Behavior

### Node Add/Remove

```
Initial cluster: 3 Nodes
  Node 1, 2, 3 → Each has DaemonSet Pod

Add Node 4:
  DaemonSet Controller detects new Node
  → Automatically creates Pod on Node 4

Remove Node 2:
  Node 2 goes down
  → Pod on Node 2 automatically deleted
  → No replacement (Node is gone)
```

---

### Node Selector

**Run DaemonSet only on specific Nodes:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-monitor
spec:
  template:
    spec:
      nodeSelector:
        gpu: "true"  # Only Nodes with label gpu=true
      containers:
      - name: gpu-monitor
        image: nvidia/gpu-monitor

Result:
  • Only GPU Nodes get this DaemonSet Pod
  • Non-GPU Nodes ignored
```

---

### Node Affinity

**Advanced node selection:**

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/worker
                operator: Exists
              - key: kubernetes.io/os
                operator: In
                values:
                - linux

Result:
  • Only worker Nodes
  • Only Linux OS
```

---

### Tolerations

**Run on tainted Nodes (e.g., master):**

```yaml
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      - key: node.kubernetes.io/unschedulable
        effect: NoSchedule

Result:
  • Can run on master Nodes (usually tainted)
  • Can run on unschedulable Nodes
  • Essential for system DaemonSets
```

---

## ⚙️ Update Strategies

### 1. RollingUpdate (Default)

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # Max Pods unavailable during update
```

**Process:**
```
Update image: v1 → v2

Step 1: Update Pod on Node 1
  • Terminate old Pod
  • Create new Pod with v2
  • Wait for Ready

Step 2: Update Pod on Node 2
  • Same process

...

Continue until all Nodes updated
```

**maxUnavailable:**
```yaml
maxUnavailable: 1      # Update 1 Node at a time (safe)
maxUnavailable: "30%"  # Update 30% of Nodes at once (faster)
```

---

### 2. OnDelete

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

**Behavior:**
```
Update DaemonSet template → Nothing happens

Manual Pod deletion:
  kubectl delete pod <pod-name>
  → New Pod created with new template

Use case: Manual, controlled updates
```

---

## 🔧 DaemonSet Operations

### Create

```bash
kubectl apply -f daemonset.yaml

# Verify
kubectl get daemonset

# Output:
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
node-exporter   3         3         3       3            3

# DESIRED: Number of Nodes
# CURRENT: Number of Pods
# READY: Pods ready
```

### List Pods

```bash
kubectl get pods -l app=node-exporter -o wide

# Output:
NAME                  NODE     READY   STATUS
node-exporter-abc12   node-1   1/1     Running
node-exporter-def34   node-2   1/1     Running
node-exporter-ghi56   node-3   1/1     Running
```

### Update

```bash
# Update image
kubectl set image daemonset/node-exporter \
  node-exporter=prom/node-exporter:v1.3.0

# Check rollout status
kubectl rollout status daemonset/node-exporter
```

### Describe

```bash
kubectl describe daemonset node-exporter

# Shows:
# - Pods status
# - Update strategy
# - Node selector
# - Tolerations
# - Events
```

### Delete

```bash
kubectl delete daemonset node-exporter

# All DaemonSet Pods deleted from all Nodes
```

---

## 🎨 Advanced Configurations

### Host Network

```yaml
spec:
  template:
    spec:
      hostNetwork: true  # Use Node's network namespace
      
Use case:
  • Network plugins (Calico, Flannel)
  • Need to bind to Node's IP
```

---

### Host Path Volumes

```yaml
spec:
  template:
    spec:
      volumes:
      - name: logs
        hostPath:
          path: /var/log
          type: Directory
      containers:
      - name: log-collector
        volumeMounts:
        - name: logs
          mountPath: /host-logs

Use case:
  • Access Node's filesystem
  • Collect logs, metrics
```

---

### Privileged Containers

```yaml
spec:
  template:
    spec:
      containers:
      - name: privileged-daemon
        securityContext:
          privileged: true  # Full host access

Use case:
  • System-level operations
  • Storage daemons
  • Network management
  ⚠️ Security risk - use carefully
```

---

## 💡 Best Practices

### ✅ DO

1. **Resource limits**
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi

# DaemonSets run on ALL Nodes
# Over-request → Starve other workloads
```

2. **Readiness probes**
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 9100
  initialDelaySeconds: 10
```

3. **Tolerations for system DaemonSets**
```yaml
tolerations:
- key: node-role.kubernetes.io/master
  effect: NoSchedule
```

4. **Node selector for specialized DaemonSets**
```yaml
nodeSelector:
  disktype: ssd  # Only SSD Nodes
```

5. **RollingUpdate for safety**
```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
```

---

### ❌ DON'T

1. **High resource requests** → Block other Pods
2. **No resource limits** → DaemonSet can starve Node
3. **Run application workloads** → Use Deployment instead
4. **Forget tolerations** → Won't run on master/tainted Nodes
5. **No readiness probe** → Rolling update issues

---

## 📊 Monitoring DaemonSets

```bash
# Check DaemonSet status
kubectl get daemonset

# Desired = Current = Ready?
# If not, investigate:

# Check Pods
kubectl get pods -l app=<daemonset-label>

# Check events
kubectl describe daemonset <name>

# Common issues:
# - Node selector doesn't match any Nodes
# - Missing tolerations for tainted Nodes
# - Resource constraints
# - Image pull errors
```

---

## 🎓 Key Takeaways

1. **DaemonSet:** 1 Pod per Node automatically
2. **Use cases:** Monitoring, logging, networking, storage, security
3. **Auto-scaling:** Add Node → Pod created automatically
4. **Node selection:** nodeSelector, nodeAffinity, tolerations
5. **Update strategies:** RollingUpdate (safe), OnDelete (manual)
6. **Resource limits:** Essential (runs on ALL Nodes)
7. **NOT for applications:** Use Deployment for regular apps

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. DaemonSet dùng để làm gì?
2. Điều gì xảy ra khi thêm Node mới vào cluster?
3. Use cases phổ biến của DaemonSet?
4. Làm sao chỉ chạy DaemonSet trên một số Nodes?
5. Tại sao cần tolerations trong DaemonSet?
6. maxUnavailable trong RollingUpdate là gì?

---

[⬅️ 4.3. StatefulSet](./03-statefulset.md) | [➡️ 4.5. Job & CronJob](./05-jobs-cronjobs.md) | [🏠 Mục Lục Chính](../README.md)

