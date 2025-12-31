# 4.4. DaemonSet - Pod Trên Mỗi Node

> Run exactly one Pod on every Node in cluster

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **DaemonSet là gì** và **TẠI SAO cần**
- ✅ Biết **use cases** phổ biến cho DaemonSet
- ✅ Deploy **system daemons** (logging, monitoring)
- ✅ Control **which Nodes** run DaemonSet Pods
- ✅ **Update và rollback** DaemonSets
- ✅ **Troubleshoot** DaemonSet issues

---

## 📦 DaemonSet Là Gì?

### Định Nghĩa

**DaemonSet** = Controller ensures một Pod copy chạy trên mỗi Node (hoặc subset of Nodes) trong cluster.

### Giải Thích Bằng Ví Dụ

**DaemonSet giống như security guards:**

```
🏢 TÒA NHÀ VĂN PHÒNG (Cluster)

DEPLOYMENT = Team làm dự án:
├── 3 developers work together
├── Có thể ở bất kỳ tầng nào
├── Không cần mỗi tầng có 1 người
└── Scale up/down based on workload

Use cases: Application servers

DAEMONSET = Security guards:
├── MỖI TẦNG (Node) phải có 1 guard
├── Tầng mới (new Node) → Deploy 1 guard automatically
├── Tầng đóng cửa (Node removed) → Guard removed
└── Always 1 guard per floor, no more, no less

Use cases: Monitoring, logging, network agents
```

---

## 🤔 TẠI SAO Cần DaemonSet?

### Problem Without DaemonSet

**Scenario: Logging agents**

```bash
# Using Deployment
kubectl create deployment log-collector --image=fluentd --replicas=3

# Problems:
❌ 3 Pods random placement (maybe all on 1 Node!)
   Node 1: 2 log-collector Pods
   Node 2: 1 log-collector Pod
   Node 3: 0 log-collector Pods ← No logging!

❌ Need logging on ALL Nodes
   → Must manually calculate replicas = number of Nodes
   → Nodes added/removed → Manual updates

❌ Pods can be evicted/rescheduled
   → Node might temporarily have 0 Pods
   → Missing logs during that time

❌ Can't guarantee 1 Pod per Node
```

### Solution: DaemonSet

```yaml
# Using DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    spec:
      containers:
      - name: fluentd
        image: fluentd

# Benefits:
✓ Exactly 1 Pod per Node
  Node 1: 1 log-collector Pod
  Node 2: 1 log-collector Pod
  Node 3: 1 log-collector Pod

✓ Automatic scaling
  Node 4 added → Pod automatically created
  Node 2 removed → Pod automatically deleted

✓ Guaranteed coverage
  Every Node has logging agent
  No gaps in log collection

✓ No manual replica management
  DaemonSet handles it automatically
```

---

## 🎯 Common Use Cases

### 1. Log Collection

**Collect logs từ mọi Node:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset
        volumeMounts:
        - name: varlog
          mountPath: /var/log
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

**Why DaemonSet?**
- Need to collect logs từ ALL Nodes
- Each Node's logs stored locally on Node
- Agent must run on each Node to access logs

---

### 2. Monitoring Agents

**Collect metrics từ mọi Node:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
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
        image: prom/node-exporter
        ports:
        - containerPort: 9100
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      hostNetwork: true  # Access Node's network
      hostPID: true      # Access Node's processes
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
```

**Why DaemonSet?**
- Monitor CPU, memory, disk của mỗi Node
- Metrics specific to each Node
- Need agent on every Node

---

### 3. Network Plugins (CNI)

**Network overlay agents:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-proxy
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: kube-proxy
  template:
    spec:
      containers:
      - name: kube-proxy
        image: k8s.gcr.io/kube-proxy
      hostNetwork: true  # Access Node network
```

**Examples:**
- kube-proxy (K8s networking)
- Calico, Flannel, Weave (CNI plugins)
- Load balancer agents

**Why DaemonSet?**
- Network configuration needed on every Node
- Pods on each Node need network access
- Can't function without agent on Node

---

### 4. Storage Plugins

**Storage drivers:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: csi-driver
spec:
  template:
    spec:
      containers:
      - name: csi-node
        image: my-csi-driver
        volumeMounts:
        - name: plugin-dir
          mountPath: /var/lib/kubelet/plugins
      volumes:
      - name: plugin-dir
        hostPath:
          path: /var/lib/kubelet/plugins
```

**Examples:**
- CSI node plugins
- GlusterFS, Ceph clients
- Cloud provider storage drivers

---

## 📝 DaemonSet YAML

### Basic Example

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
  labels:
    app: monitoring
spec:
  # Selector to match Pods
  selector:
    matchLabels:
      app: monitoring-agent
  
  # Pod template
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
      - name: agent
        image: monitoring-agent:v1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        ports:
        - containerPort: 9090
          name: metrics
```

### With hostPath Volumes

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
        image: fluentd:v1
        volumeMounts:
        - name: varlog
          mountPath: /var/log  # Mount Node's /var/log
        - name: containers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log  # Node's filesystem
      - name: containers
        hostPath:
          path: /var/lib/docker/containers
```

---

## 🎮 Hands-On: Working với DaemonSets

### Create DaemonSet

```yaml
# daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemon
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx-daemon
  template:
    metadata:
      labels:
        app: nginx-daemon
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

```bash
# Create DaemonSet
kubectl apply -f daemonset.yaml

# Output:
# daemonset.apps/nginx-daemon created

# Check DaemonSet
kubectl get daemonset
# or shorter
kubectl get ds

# Output:
# NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
# nginx-daemon   3         3         3       3            3           <none>          30s
#                ↑         ↑
#                Number of Nodes

# Check Pods (one per Node!)
kubectl get pods -o wide

# Output:
# NAME                 READY   STATUS    NODE      IP
# nginx-daemon-abc12   1/1     Running   node-1    10.244.1.5
# nginx-daemon-def34   1/1     Running   node-2    10.244.2.8
# nginx-daemon-ghi56   1/1     Running   node-3    10.244.3.9

# Exactly 1 Pod per Node!
```

---

### Test Node Addition

```bash
# Add new Node to cluster (minikube example)
minikube node add

# DaemonSet automatically creates Pod on new Node!
kubectl get pods -o wide

# Output:
# NAME                 READY   STATUS    NODE      IP
# nginx-daemon-abc12   1/1     Running   node-1    10.244.1.5
# nginx-daemon-def34   1/1     Running   node-2    10.244.2.8
# nginx-daemon-ghi56   1/1     Running   node-3    10.244.3.9
# nginx-daemon-jkl78   1/1     Running   node-4    10.244.4.10  ← NEW!

# Pod automatically created on node-4!
```

---

### Node Selector

**Run DaemonSet only on specific Nodes:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      # Only run on Nodes với label disktype=ssd
      nodeSelector:
        disktype: ssd
      containers:
      - name: monitor
        image: ssd-monitor:v1
```

```bash
# Label some Nodes
kubectl label nodes node-1 disktype=ssd
kubectl label nodes node-2 disktype=ssd

# Create DaemonSet
kubectl apply -f ssd-monitor-daemonset.yaml

# Pods only on node-1 và node-2!
kubectl get pods -o wide
# NAME                 READY   STATUS    NODE
# ssd-monitor-abc12    1/1     Running   node-1
# ssd-monitor-def34    1/1     Running   node-2
# (No Pod on node-3 - doesn't have disktype=ssd label)

# Add label to node-3
kubectl label nodes node-3 disktype=ssd

# Pod automatically created on node-3!
kubectl get pods -o wide
# ssd-monitor-ghi56    1/1     Running   node-3  ← NEW!
```

---

### Tolerations cho Master Nodes

**By default: DaemonSet Pods DON'T run on Master Nodes**

```bash
# Check Master Node taints
kubectl describe node master-node | grep Taints

# Output:
# Taints: node-role.kubernetes.io/master:NoSchedule

# Master has taint → Regular Pods can't schedule
```

**Run DaemonSet on Master:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-everywhere
spec:
  selector:
    matchLabels:
      app: monitoring
  template:
    metadata:
      labels:
        app: monitoring
    spec:
      # Tolerate Master Node taint
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      # Also tolerate control-plane taint (newer K8s)
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: agent
        image: monitoring-agent:v1
```

```bash
# Apply
kubectl apply -f monitoring-daemonset.yaml

# Pods on ALL Nodes including Master!
kubectl get pods -o wide
# NAME                       READY   STATUS    NODE
# monitoring-abc12           1/1     Running   master-node  ← On Master!
# monitoring-def34           1/1     Running   worker-1
# monitoring-ghi56           1/1     Running   worker-2
```

---

### Update DaemonSet

**Update strategy:**

```yaml
spec:
  updateStrategy:
    type: RollingUpdate  # Default
    rollingUpdate:
      maxUnavailable: 1  # Max Pods down during update
```

**Update image:**

```bash
# Update image
kubectl set image daemonset/nginx-daemon nginx=nginx:1.22

# Pods updated one Node at a time
kubectl get pods -w

# Output:
# nginx-daemon-abc12   1/1     Terminating   node-1
# nginx-daemon-abc12   0/1     ContainerCreating   node-1
# nginx-daemon-abc12   1/1     Running   node-1
# nginx-daemon-def34   1/1     Terminating   node-2  ← Next Node
# ...

# Check status
kubectl rollout status daemonset/nginx-daemon

# Output:
# daemon set "nginx-daemon" successfully rolled out
```

**OnDelete strategy:**

```yaml
spec:
  updateStrategy:
    type: OnDelete  # Manual control
```

```bash
# Update DaemonSet
kubectl set image daemonset/nginx-daemon nginx=nginx:1.22

# Pods NOT updated automatically!

# Manually delete Pods to trigger update
kubectl delete pod nginx-daemon-abc12
# Pod recreated with new image

kubectl delete pod nginx-daemon-def34
# Pod recreated with new image
```

---

### Delete DaemonSet

```bash
# Delete DaemonSet
kubectl delete daemonset nginx-daemon

# All Pods deleted
kubectl get pods -l app=nginx-daemon
# No resources found

# DaemonSet deleted
kubectl get daemonset nginx-daemon
# Error from server (NotFound)
```

---

## 🔧 Advanced Configurations

### hostNetwork: true

**Use Node's network namespace:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: network-monitor
spec:
  template:
    spec:
      hostNetwork: true  # Use Node's network
      containers:
      - name: monitor
        image: network-monitor:v1
        ports:
        - containerPort: 9100  # Binds to Node's port 9100!
```

**Use cases:**
- Network monitoring tools
- kube-proxy
- CNI plugins

**⚠️ Warning:** Pod exposes ports directly on Node!

---

### hostPID và hostIPC

**Access Node's processes:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: system-monitor
spec:
  template:
    spec:
      hostPID: true   # See Node's processes
      hostIPC: true   # Access Node's IPC
      containers:
      - name: monitor
        image: system-monitor:v1
```

**Use cases:**
- System monitoring (CPU, memory per process)
- Debugging tools
- Security scanning

---

### Privileged Containers

**Run với elevated privileges:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: privileged-agent
spec:
  template:
    spec:
      containers:
      - name: agent
        image: privileged-agent:v1
        securityContext:
          privileged: true  # Full access to Node
        volumeMounts:
        - name: dev
          mountPath: /dev
      volumes:
      - name: dev
        hostPath:
          path: /dev
```

**Use cases:**
- Storage drivers
- Network plugins
- System-level tools

**⚠️ Security risk:** Use only when necessary!

---

## 🐛 Troubleshooting DaemonSets

### Issue 1: Pods Not on All Nodes

```bash
$ kubectl get daemonset
NAME           DESIRED   CURRENT   READY
nginx-daemon   5         3         3

# Only 3/5 Nodes have Pods

# Check Pod status
$ kubectl get pods -l app=nginx-daemon -o wide

# Describe DaemonSet
$ kubectl describe daemonset nginx-daemon

# Events might show:
# Warning  FailedCreate  Pod nginx-daemon-xyz is forbidden: 
# unable to validate against any pod security policy

# Possible causes:
1. Node taints (Pods can't tolerate)
2. NodeSelector mismatch
3. Resource constraints
4. Pod Security Policy blocking

# Debug steps:
kubectl describe nodes  # Check taints
kubectl get nodes --show-labels  # Check labels
```

---

### Issue 2: Pods CrashLoopBackOff

```bash
$ kubectl get pods -l app=log-collector
NAME                  READY   STATUS             RESTARTS   AGE
log-collector-abc12   0/1     CrashLoopBackOff   5          5m

# DaemonSet Pod crashing on some Nodes

# Check logs
$ kubectl logs log-collector-abc12

# Possible causes:
1. hostPath volume doesn't exist on Node
2. Permission issues (need privileged: true)
3. Resource limits too low
4. Application configuration error

# Fix examples:
# - Add privileged: true if needed
# - Adjust resource limits
# - Fix hostPath paths
```

---

### Issue 3: Can't Delete DaemonSet

```bash
$ kubectl delete daemonset nginx-daemon
# Hangs...

# Check finalizers
$ kubectl get daemonset nginx-daemon -o yaml | grep finalizers

# Force delete if stuck
$ kubectl delete daemonset nginx-daemon --force --grace-period=0

# Or patch to remove finalizers
$ kubectl patch daemonset nginx-daemon -p '{"metadata":{"finalizers":[]}}' --type=merge
```

---

## 🎓 Kiểm Tra Hiểu Biết

**1. DaemonSet vs Deployment - khi nào dùng gì?**
<details>
<summary>Xem đáp án</summary>

**DaemonSet:**
- Use when: Need exactly 1 Pod per Node
- Examples: Logging, monitoring, network agents
- Scaling: Automatic (# Pods = # Nodes)

**Deployment:**
- Use when: Need N replicas (can be anywhere)
- Examples: Web apps, APIs, workers
- Scaling: Manual or HPA

**Rule:** If need "every Node" → DaemonSet. Otherwise → Deployment.
</details>

**2. Làm sao run DaemonSet on Master Nodes?**
<details>
<summary>Xem đáp án</summary>

Add tolerations:

```yaml
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
```

Master Nodes have NoSchedule taints by default. Tolerations allow Pods to schedule despite taints.
</details>

**3. DaemonSet có auto-scale khi add Nodes không?**
<details>
<summary>Xem đáp án</summary>

**YES! Automatic.**

```
Cluster: 3 Nodes → DaemonSet: 3 Pods

Add Node 4 → DaemonSet automatically: 4 Pods
Remove Node 2 → DaemonSet automatically: 3 Pods

No manual intervention needed!
```

This is THE reason DaemonSets exist - automatic per-Node deployment.
</details>

---

## 💪 Bài Tập Thực Hành

### Bài 1: Deploy Monitoring DaemonSet

```yaml
# monitoring-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      containers:
      - name: monitor
        image: busybox
        command: ['sh', '-c', 'while true; do hostname; sleep 30; done']
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
```

```bash
# 1. Deploy
kubectl apply -f monitoring-daemonset.yaml

# 2. Verify (1 Pod per Node)
kubectl get pods -o wide

# 3. Check logs from different Pods
kubectl logs -l app=node-monitor --tail=5

# 4. Update image
kubectl set image daemonset/node-monitor monitor=busybox:1.35

# 5. Watch rollout
kubectl rollout status daemonset/node-monitor

# 6. Cleanup
kubectl delete daemonset node-monitor
```

---

## 🎯 Key Takeaways

1. **DaemonSet = One Pod Per Node**
   - Exactly 1 Pod on each Node
   - Automatic scaling với cluster

2. **Use Cases**
   - Logging agents (Fluentd)
   - Monitoring (Node Exporter)
   - Network (CNI plugins, kube-proxy)
   - Storage (CSI drivers)

3. **Node Selection**
   - nodeSelector: Run on specific Nodes
   - tolerations: Run on tainted Nodes (Master)
   - Default: All worker Nodes

4. **Updates**
   - RollingUpdate: Automatic, one Node at a time
   - OnDelete: Manual control

5. **vs Deployment**
   - DaemonSet: Per-Node daemons
   - Deployment: N replicas anywhere

---

## 🚀 Tiếp Theo

DaemonSet mastered! Next: Jobs & CronJobs - batch workloads!

**Next:** [4.5. Jobs & CronJobs →](./05-jobs-cronjobs.md)

---

[⬅️ 4.3. StatefulSet](./03-statefulset.md) | [🏠 Mục Lục](../README.md) | [📂 Phần 4: Workloads](./README.md) | [➡️ 4.5. Jobs](./05-jobs-cronjobs.md)
