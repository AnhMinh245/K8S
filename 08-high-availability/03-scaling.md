# 8.3. Scaling - Tự Động Mở Rộng

> Tự động mở rộng/thu nhỏ ứng dụng dựa trên nhu cầu thực tế

---

## 📖 Mục Lục

1. [Scaling là gì?](#-scaling-là-gì)
2. [Manual Scaling](#-manual-scaling)
3. [Horizontal Pod Autoscaler (HPA)](#-horizontal-pod-autoscaler-hpa)
4. [Vertical Pod Autoscaler (VPA)](#-vertical-pod-autoscaler-vpa)
5. [Cluster Autoscaler](#-cluster-autoscaler)
6. [HPA Metrics Deep Dive](#-hpa-metrics-deep-dive)
7. [HPA Behavior Configuration](#-hpa-behavior-configuration)
8. [Hands-on Labs](#-hands-on-labs)
9. [Troubleshooting](#-troubleshooting)
10. [Best Practices](#-best-practices)

---

## 🤔 Scaling là gì?

### Định nghĩa

**Scaling** là khả năng tự động điều chỉnh tài nguyên để đáp ứng nhu cầu:
- 📈 **Scale Up/Out:** Tăng tài nguyên khi traffic tăng
- 📉 **Scale Down/In:** Giảm tài nguyên khi traffic giảm
- 💰 **Cost optimization:** Chỉ trả tiền cho tài nguyên thực sự cần
- 🚀 **Performance:** Đảm bảo response time tốt

### Vấn đề nếu không có Auto-scaling

**❌ Scenario: E-commerce Black Friday**

```
Normal traffic: 1000 req/s → 10 Pods (OK)

Black Friday:
09:00 - Traffic: 50,000 req/s → 10 Pods (overwhelmed!)
        ├── CPU: 98% (maxed out)
        ├── Response time: 10 seconds (slow!)
        ├── Many requests timeout ❌
        └── Lost sales! 💸

Manual scaling:
09:30 - Admin wakes up, scales to 100 Pods
10:00 - Pods running, performance restored
        └── But: Lost 1 hour of sales! 🔥

After Black Friday:
23:00 - Traffic: 1000 req/s (back to normal)
        ├── Still running 100 Pods (99% idle!)
        └── Wasting money on 90 unused Pods! 💸
```

**✅ With Auto-scaling:**

```
Normal traffic: 1000 req/s → 10 Pods (OK)

Black Friday:
09:00 - Traffic: 50,000 req/s
09:01 - HPA detects CPU > 70%
09:02 - HPA scales to 100 Pods (automatic!)
09:05 - All Pods running, performance restored ✅
        └── Only 5 minutes of degraded performance!

After Black Friday:
23:00 - Traffic: 1000 req/s (back to normal)
23:01 - HPA detects CPU < 30%
23:06 - HPA scales down to 10 Pods (automatic!)
        └── No wasted resources! ✅
```

### 2 Types of Scaling

**Horizontal Scaling (Scale Out/In):**
```
┌─────────┐                  ┌─────────┐ ┌─────────┐ ┌─────────┐
│  Pod 1  │  → Scale out →   │  Pod 1  │ │  Pod 2  │ │  Pod 3  │
│ 100% CPU│                  │ 40% CPU │ │ 40% CPU │ │ 40% CPU │
└─────────┘                  └─────────┘ └─────────┘ └─────────┘

More Pods! (horizontal)
```

**Vertical Scaling (Scale Up/Down):**
```
┌─────────────┐              ┌─────────────┐
│    Pod 1    │              │    Pod 1    │
│ cpu: 100m   │  → Scale →   │ cpu: 500m   │
│ mem: 128Mi  │     up       │ mem: 512Mi  │
│ (100% used) │              │ (30% used)  │
└─────────────┘              └─────────────┘

Bigger Pod! (vertical)
```

---

## 🖱️ Manual Scaling

### Basic Commands

```bash
# Scale Deployment
kubectl scale deployment web --replicas=10

# Scale ReplicaSet
kubectl scale replicaset web-abc123 --replicas=5

# Scale StatefulSet
kubectl scale statefulset mysql --replicas=3

# Scale using manifest
kubectl scale -f deployment.yaml --replicas=20

# Scale multiple resources
kubectl scale deployment web api db --replicas=5
```

### Verify scaling

```bash
# Watch Pods being created
kubectl get pods -w

# Check Deployment replicas
kubectl get deployment web
# NAME   READY   UP-TO-DATE   AVAILABLE   AGE
# web    10/10   10           10          5m

# Describe Deployment
kubectl describe deployment web
# Replicas:  10 desired | 10 updated | 10 total | 10 available
```

### Use cases

**✅ Khi nào dùng manual scaling:**
- 🗓️ **Scheduled events:** Traffic spike đã biết trước (product launch, sale)
- 🧪 **Testing:** Load testing, stress testing
- 📊 **Known patterns:** Traffic patterns hàng ngày/tuần
- 🚀 **Quick fix:** Response nhanh trước khi HPA kích hoạt

**Example:**

```bash
# Product launch at 10:00 AM
# Pre-scale 30 minutes before
09:30 $ kubectl scale deployment web --replicas=50

# After launch (traffic stabilizes)
12:00 $ kubectl scale deployment web --replicas=20
```

---

## 📊 Horizontal Pod Autoscaler (HPA)

### HPA là gì?

**HPA** tự động điều chỉnh số lượng Pods dựa trên metrics (CPU, memory, custom).

### Basic Example

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2         # ← Minimum Pods (HA)
  maxReplicas: 10        # ← Maximum Pods (cost control)
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # ← Target: 70% CPU
```

**Cách hoạt động:**

```
Hiện tại: 2 Pods, CPU: 85% (> 70% target)
    ↓
HPA tính toán: cần 3 Pods (85% / 70% * 2 ≈ 2.4 → làm tròn lên 3)
    ↓
HPA scale Deployment lên 3 replicas
    ↓
Pod mới khởi động
    ↓
CPU: 60% (< 70% target) ✅
```

### Create HPA (Command)

```bash
# Simple HPA (CPU only)
kubectl autoscale deployment web \
  --min=2 \
  --max=10 \
  --cpu-percent=70

# Verify
kubectl get hpa
```

**Output:**
```
NAME      REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
web-hpa   Deployment/web   45%/70%   2         10        3          5m
                           ↑ Current/Target
```

### HPA Algorithm

**Formula:**

```
desiredReplicas = ceil[currentReplicas * (currentMetric / targetMetric)]
```

**Example:**

```
Current: 5 Pods
Current CPU: 85%
Target CPU: 70%

desiredReplicas = ceil[5 * (85 / 70)]
                = ceil[5 * 1.214]
                = ceil[6.07]
                = 7 Pods

HPA scales to 7 Pods! ✅
```

### Timeline Example

```
00:00 - Deployment created, replicas: 2
        └── CPU: 30% (normal traffic)

00:00 - HPA created (target: 70%)
        └── No action needed (30% < 70%)

01:00 - Traffic spike!
        ├── CPU: 85% (> 70% target)
        └── Current: 2 Pods

01:00 - HPA calculates
        └── desiredReplicas = ceil[2 * (85/70)] = 3

01:01 - HPA updates Deployment
        └── replicas: 2 → 3

01:02 - New Pod created
        └── Status: Running, Ready: 1/1

01:03 - Traffic distributed to 3 Pods
        └── CPU: 60% (< 70% target) ✅

01:30 - More traffic!
        ├── CPU: 90% (> 70%)
        └── desiredReplicas = ceil[3 * (90/70)] = 4

01:31 - HPA scales to 4 Pods
        └── CPU: 65% ✅

02:00 - Traffic decreases
        ├── CPU: 40% (< 70%)
        └── desiredReplicas = ceil[4 * (40/70)] = 3

02:05 - HPA waits (stabilization window: 5 min)
        └── "Maybe temporary drop..."

07:05 - Still low (40%), scale down
        └── replicas: 4 → 3

07:10 - CPU: 55% ✅
```

### Prerequisites

**❗ REQUIRED: Resource requests must be set!**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: app
        image: my-app:1.0
        resources:
          requests:
            cpu: 200m      # ← REQUIRED for CPU-based HPA!
            memory: 256Mi  # ← REQUIRED for memory-based HPA!
          limits:
            cpu: 500m
            memory: 512Mi
```

**Why?** HPA calculates utilization:

```
CPU Utilization = (Current CPU usage / CPU request) * 100%

Example:
Current usage: 150m
Request: 200m
Utilization = (150 / 200) * 100% = 75%
```

---

## 📐 Vertical Pod Autoscaler (VPA)

### VPA là gì?

**VPA** tự động điều chỉnh **requests/limits** CPU và memory cho Pods.

### Khi nào dùng VPA vs HPA?

**HPA (Horizontal):**
- ✅ Stateless apps (web servers, APIs)
- ✅ Có thể chạy nhiều replicas
- ✅ Scale nhanh

**VPA (Vertical):**
- ✅ Stateful apps (databases, caches)
- ✅ Không thể scale horizontal dễ dàng
- ✅ Cần right-size resources

### VPA Example

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Auto"  # ← Auto update (recreate Pods)
    # updateMode: "Off"      # Just recommend, don't apply
    # updateMode: "Initial"  # Only set at Pod creation
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2000m
        memory: 2Gi
```

### Điều gì xảy ra?

**Trạng thái ban đầu:**

```yaml
resources:
  requests:
    cpu: 100m      # ← Initial guess
    memory: 128Mi
```

**Sau 1 tuần monitoring:**

```
VPA quan sát:
├── CPU: Thực tế dùng 300-400m (liên tục)
└── Memory: Thực tế dùng 350-450Mi (liên tục)

VPA đề xuất:
├── cpu: 100m → 500m (tăng lên!)
└── memory: 128Mi → 512Mi (tăng lên!)
```

**VPA áp dụng (updateMode: Auto):**

```
1. Update Pod spec:
   resources:
     requests:
       cpu: 500m      # ← Đã cập nhật!
       memory: 512Mi  # ← Đã cập nhật!

2. Recreate Pods với resources mới
3. Pods mới có right-sized resources! ✅
```

### VPA Update Modes

**Auto:**
```yaml
updateMode: "Auto"
# ✅ Tự động update và recreate Pods
# ⚠️  Gây downtime (Pod restart)
# Dùng với: Multiple replicas + PodDisruptionBudget
```

**Off:**
```yaml
updateMode: "Off"
# ✅ Chỉ đề xuất, không áp dụng
# Use case: Review thủ công trước khi áp dụng
```

**Initial:**
```yaml
updateMode: "Initial"
# ✅ Chỉ set resources khi tạo Pod
# ⚠️  Không update Pods đang chạy
# Use case: Ít disruption hơn, nhưng thích nghi chậm hơn
```

### Kiểm tra VPA recommendations

```bash
# Xem VPA status
kubectl get vpa web-vpa -o yaml

# Kiểm tra recommendations
kubectl describe vpa web-vpa
```

**Output:**

```yaml
status:
  recommendation:
    containerRecommendations:
    - containerName: app
      lowerBound:
        cpu: 250m
        memory: 256Mi
      target:           # ← VPA applies this!
        cpu: 500m
        memory: 512Mi
      uncappedTarget:
        cpu: 750m       # ← Uncapped recommendation
        memory: 768Mi
      upperBound:
        cpu: 1000m
        memory: 1Gi
```

### ⚠️ HPA + VPA Conflict!

**❌ DON'T use HPA and VPA on the same metric:**

```yaml
# HPA scales based on CPU
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      averageUtilization: 70

# VPA adjusts CPU requests
# ↑ CONFLICT! Both changing CPU!
```

**Problem:**

```
HPA: "CPU is 80%, scale to 5 Pods!"
VPA: "CPU is 80%, increase CPU request to 500m!"
    ↓
Both act simultaneously → unpredictable behavior! 🔥
```

**✅ Solution: Use different metrics**

```yaml
# HPA: Scale based on memory
hpa:
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        averageUtilization: 80

# VPA: Adjust CPU only
vpa:
  resourcePolicy:
    containerPolicies:
    - containerName: app
      mode: Auto
      controlledResources: ["cpu"]  # ← Only CPU, not memory!
```

---

## 🌐 Cluster Autoscaler

### Cluster Autoscaler là gì?

**Cluster Autoscaler** tự động thêm hoặc xóa **Nodes** khỏi cluster.

### Khi nào được trigger?

**Scale Up (Thêm Nodes):**
```
HPA scale lên 50 Pods
    ↓
Scheduler cố gắng assign Pods
    ↓
❌ Không đủ CPU/memory trên Nodes hiện tại!
    ↓
Pods bị stuck ở trạng thái "Pending"
    ↓
Cluster Autoscaler phát hiện
    ↓
Thêm Nodes mới (qua cloud provider API)
    ↓
Pods được schedule lên Nodes mới ✅
```

**Scale Down (Xóa Nodes):**
```
Traffic giảm
    ↓
HPA scale xuống còn 10 Pods
    ↓
Một số Nodes bị underutilized (< 50% usage)
    ↓
Cluster Autoscaler phát hiện
    ↓
Evict Pods an toàn khỏi Node underutilized
    ↓
Xóa Node (qua cloud provider API)
    ↓
Tiết kiệm tiền! 💰
```

### Timeline Example

```
09:00 - Cluster: 3 Nodes, 30 Pods
        ├── Node 1: 10 Pods (80% CPU)
        ├── Node 2: 10 Pods (80% CPU)
        └── Node 3: 10 Pods (80% CPU)

09:00 - Traffic spike!
        └── HPA scales web Deployment: 10 → 50 replicas

09:01 - 40 new Pods created
        └── Status: Pending (no Node can fit them!)
        
09:01 - Cluster Autoscaler detects
        └── "Need more Nodes!"

09:02 - Cluster Autoscaler calls cloud API
        └── "AWS: Create 2 new m5.xlarge instances"

09:05 - New Nodes join cluster
        ├── Node 4: Ready
        └── Node 5: Ready

09:06 - Scheduler assigns Pending Pods
        ├── Node 4: 20 Pods
        └── Node 5: 20 Pods

09:10 - All 50 Pods running! ✅
        └── Cluster: 5 Nodes, 50 Pods

15:00 - Traffic drops
        └── HPA scales: 50 → 10 replicas

15:05 - Cluster status:
        ├── Node 1: 3 Pods (30% CPU)
        ├── Node 2: 3 Pods (30% CPU)
        ├── Node 3: 3 Pods (30% CPU)
        ├── Node 4: 1 Pod  (10% CPU) ← Underutilized!
        └── Node 5: 0 Pods (0% CPU)  ← Empty!

15:15 - Cluster Autoscaler (after 10min wait)
        └── "Node 5 is empty, remove it!"

15:16 - Node 5 removed
        └── Save money! 💰

15:30 - Cluster Autoscaler
        └── "Node 4 underutilized, move Pod to Node 1-3"

15:31 - Pod evicted from Node 4, rescheduled to Node 2

15:35 - Node 4 removed
        └── Cluster: 3 Nodes, 10 Pods (back to normal!)
```

### Cloud Provider Setup

**AWS EKS:**

```yaml
# cluster-autoscaler.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - name: cluster-autoscaler
        image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.27.0
        command:
        - ./cluster-autoscaler
        - --cloud-provider=aws
        - --namespace=kube-system
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
```

**GCP GKE:**

```bash
# Enable Cluster Autoscaler on node pool
gcloud container clusters update my-cluster \
  --enable-autoscaling \
  --min-nodes=3 \
  --max-nodes=10 \
  --node-pool=default-pool
```

**Azure AKS:**

```bash
# Enable Cluster Autoscaler
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10
```

---

## 📈 HPA Metrics Deep Dive

### 1. Resource Metrics (CPU, Memory)

**CPU-based:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # ← 70% of CPU request
```

**Memory-based:**

```yaml
metrics:
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 80  # ← 80% of memory request
```

**Both CPU and Memory:**

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 80

# ↑ HPA scales based on HIGHEST utilization!
# If CPU: 60%, Memory: 85% → Scale based on Memory (85%)
```

### 2. Custom Metrics (Application-specific)

**HTTP requests per second:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"  # ← 1000 req/s per Pod
```

**Cách hoạt động:**

```
Metric: http_requests_total = 1000

Được enriched với K8s metadata:
  • pod_name: web-abc123
  • namespace: production
  • app: web
  • version: v1.2.3
  • tier: frontend
  • environment: prod
  • node: node-1
  • cluster: prod-cluster
  • team: platform (từ annotation)

Query trong Datadog/Dynatrace:
  "Hiển thị http_requests_total với environment=prod VÀ tier=frontend"
```

### 3. External Metrics (Queue depth, etc.)

**RabbitMQ queue depth:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker
  minReplicas: 1
  maxReplicas: 50
  metrics:
  - type: External
    external:
      metric:
        name: rabbitmq_queue_depth
        selector:
          matchLabels:
            queue: "tasks"
      target:
        type: AverageValue
        averageValue: "30"  # ← 30 messages per Pod
```

**Ví dụ:**

```
Queue: 500 messages
Hiện tại: 5 worker Pods
Average mỗi Pod: 500 / 5 = 100 messages

Target: 30 messages mỗi Pod
desiredReplicas = ceil[5 * (100 / 30)] = 17 Pods

HPA scale lên 17 Pods! ✅
Average mới: 500 / 17 ≈ 29 messages mỗi Pod ✅
```

---

## ⚙️ HPA Behavior Configuration

### Hành Vi Mặc Định

**Vấn đề với default:**

```
Traffic spike → Scale up ngay lập tức
Traffic drop → Scale down ngay lập tức
    ↓
Flapping! (scale up/down liên tục)
    ↓
Pod churn, instability! 🔥
```

### Stabilization Windows

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0  # ← Scale up immediately (default)
      policies:
      - type: Percent
        value: 100       # ← Can double Pods
        periodSeconds: 15  # ← Every 15s
      - type: Pods
        value: 4         # ← Or add 4 Pods
        periodSeconds: 15
      selectPolicy: Max  # ← Use policy that adds MORE Pods
    
    scaleDown:
      stabilizationWindowSeconds: 300  # ← Wait 5 minutes before scale down!
      policies:
      - type: Percent
        value: 50        # ← Max 50% scale down at once
        periodSeconds: 60  # ← Every 60s
      - type: Pods
        value: 2         # ← Or remove 2 Pods
        periodSeconds: 60
      selectPolicy: Min  # ← Use policy that removes FEWER Pods
```

### Scale Up Example

```
00:00 - Current: 2 Pods, CPU: 85%
        └── desiredReplicas = ceil[2 * (85/70)] = 3

00:00 - Policy 1 (Percent): 2 * 100% = 2 (can add up to 2)
        Policy 2 (Pods): Can add 4 Pods
        selectPolicy: Max → Add 2 Pods (limited by Percent policy)

00:01 - Current: 4 Pods (2+2)

00:01 - Still high CPU: 80%
        └── desiredReplicas = ceil[4 * (80/70)] = 5

00:01 - (wait 15s before next scale)

00:16 - Policy 1: 4 * 100% = 4 (can add up to 4)
        Policy 2: Can add 4 Pods
        selectPolicy: Max → Add 4 Pods

00:17 - Current: 8 Pods (4+4)
        └── CPU: 60% ✅ (below target)
```

### Scale Down Example

```
10:00 - Current: 10 Pods, CPU: 40%
        └── desiredReplicas = ceil[10 * (40/70)] = 6

10:00 - HPA wants to scale down to 6 Pods
        └── But: stabilizationWindowSeconds: 300

10:00 - HPA waits... "Maybe temporary drop?"

10:05 - Still low (CPU: 40%)

15:05 - 5 minutes passed, OK to scale down now!

15:05 - Policy 1 (Percent): 10 * 50% = 5 (remove max 5)
        Policy 2 (Pods): Remove 2 Pods
        selectPolicy: Min → Remove 2 Pods (more conservative)

15:06 - Current: 8 Pods (10-2)

15:06 - CPU: 50% (still low)
        └── desiredReplicas = 6 (want to remove 2 more)

15:06 - (wait 60s before next scale)

16:06 - Remove 2 more Pods

16:07 - Current: 6 Pods
        └── CPU: 65% ✅ (close to target)
```

---

## 🧪 Hands-on Labs

### Lab 1: Setup HPA with CPU metric

**Step 1: Install Metrics Server**

```bash
# Check if Metrics Server installed
kubectl get deployment metrics-server -n kube-system

# If not, install:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl top nodes
kubectl top pods
```

**Step 2: Create Deployment with resource requests**

```yaml
# web-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
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
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m      # ← REQUIRED!
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

```bash
kubectl apply -f web-deployment.yaml
```

**Step 3: Create HPA**

```bash
# Create HPA
kubectl autoscale deployment web \
  --min=2 \
  --max=10 \
  --cpu-percent=50

# Or using YAML:
kubectl apply -f - <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF
```

**Step 4: Check HPA status**

```bash
kubectl get hpa web-hpa
```

**Output:**
```
NAME      REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
web-hpa   Deployment/web   0%/50%    2         10        2          1m
```

**Step 5: Generate load**

```bash
# Get Service URL
kubectl get svc web

# In another terminal, generate load
kubectl run -it --rm load-generator \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://web; done"
```

**Step 6: Watch HPA scale**

```bash
# Watch HPA
kubectl get hpa web-hpa -w

# Watch Pods
kubectl get pods -l app=web -w
```

**Expected:**
```
NAME      REFERENCE        TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
web-hpa   Deployment/web   0%/50%     2         10        2          2m
web-hpa   Deployment/web   75%/50%    2         10        2          3m  ← High CPU!
web-hpa   Deployment/web   75%/50%    2         10        3          3m  ← Scaled to 3!
web-hpa   Deployment/web   55%/50%    2         10        3          4m
web-hpa   Deployment/web   48%/50%    2         10        3          5m  ← Stable
```

**Step 7: Stop load and watch scale down**

```bash
# Stop load generator (Ctrl+C)

# Watch scale down (takes ~5 minutes)
kubectl get hpa web-hpa -w
```

### Lab 2: HPA with custom metrics (HTTP requests)

**Yêu cầu:** Đã cài Prometheus + Prometheus Adapter

```bash
# Install Prometheus Adapter
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus-adapter prometheus-community/prometheus-adapter
```

**Application with metrics:**

```yaml
# app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: my-api-with-metrics:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
```

**HPA with custom metric:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"  # ← 100 req/s per Pod
```

```bash
kubectl apply -f api-hpa.yaml

# Check
kubectl get hpa api-hpa
```

---

## 🔧 Troubleshooting

### Issue 1: HPA shows "unknown" targets

**Symptoms:**

```bash
kubectl get hpa
# NAME      REFERENCE        TARGETS         MINPODS   MAXPODS   REPLICAS
# web-hpa   Deployment/web   <unknown>/70%   2         10        2
```

**Causes:**

**A. Metrics Server not installed**

```bash
# Check
kubectl get deployment metrics-server -n kube-system
# Error: deployments.apps "metrics-server" not found

# Fix: Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**B. No resource requests defined**

```bash
kubectl get deployment web -o yaml | grep -A 5 resources
# (empty or no requests)

# Fix: Add resource requests
kubectl set resources deployment web \
  --requests=cpu=200m,memory=256Mi \
  --limits=cpu=500m,memory=512Mi
```

**C. Pods not ready**

```bash
kubectl get pods -l app=web
# NAME       READY   STATUS    RESTARTS   AGE
# web-abc    0/1     Running   0          1m  ← Not ready!

# Fix: Check Pod logs, fix readiness probe
```

### Issue 2: HPA not scaling up

**Symptoms:**

```bash
# CPU is 95%, but still 2 Pods!
kubectl get hpa
# NAME      REFERENCE        TARGETS    MINPODS   MAXPODS   REPLICAS
# web-hpa   Deployment/web   95%/70%    2         10        2
```

**Debug:**

```bash
# Check HPA events
kubectl describe hpa web-hpa
```

**Causes:**

**A. maxReplicas reached**

```
Events:
  Warning  TooManyReplicas  HPA reached maximum replicas (10)
```

**Fix:**
```bash
kubectl patch hpa web-hpa -p '{"spec":{"maxReplicas":20}}'
```

**B. Insufficient cluster resources**

```
Events:
  Warning  FailedGetResourceMetric  unable to get metrics for resource cpu
```

**Fix:**
```bash
# Check Node resources
kubectl top nodes

# May need Cluster Autoscaler or add Nodes manually
```

**C. Scale-up cooldown**

```bash
# HPA has default cooldown: 3 minutes between scale-ups
# Wait and check again
```

### Issue 3: HPA flapping (scale up/down repeatedly)

**Symptoms:**

```bash
kubectl get hpa web-hpa -w
# web-hpa   Deployment/web   65%/70%    2    10    3     5m
# web-hpa   Deployment/web   72%/70%    2    10    4     5m  ← Scale up
# web-hpa   Deployment/web   68%/70%    2    10    4     6m
# web-hpa   Deployment/web   69%/70%    2    10    3     7m  ← Scale down
# web-hpa   Deployment/web   73%/70%    2    10    4     8m  ← Scale up again!
```

**Fix: Add stabilization window**

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300  # ← Wait 5 minutes
    policies:
    - type: Percent
      value: 50
      periodSeconds: 60
```

```bash
kubectl apply -f web-hpa-with-behavior.yaml
```

---

## 💡 Best Practices

### 1. Always set resource requests

❌ **Bad:**
```yaml
containers:
- name: app
  image: my-app
  # No resource requests! ❌
```

✅ **Good:**
```yaml
containers:
- name: app
  image: my-app
  resources:
    requests:
      cpu: 200m      # ← REQUIRED for HPA!
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

### 2. Set reasonable min/max replicas

❌ **Bad:**
```yaml
minReplicas: 1   # ← No HA!
maxReplicas: 1000  # ← Cost explosion!
```

✅ **Good:**
```yaml
minReplicas: 2   # ← HA (minimum 2)
maxReplicas: 50  # ← Cost control (reasonable max)
```

### 3. Use stabilization windows

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0  # Fast scale up (OK for traffic spikes)
  scaleDown:
    stabilizationWindowSeconds: 300  # Slow scale down (avoid flapping)
```

### 4. Don't use HPA + VPA on same metric

❌ **Bad:**
```yaml
# HPA scales based on CPU
# VPA adjusts CPU requests
# ↑ CONFLICT!
```

✅ **Good:**
```yaml
# HPA: Scale based on memory
# VPA: Adjust CPU requests only
# ↑ No conflict!
```

### 5. Monitor HPA metrics

```bash
# Check HPA status
kubectl get hpa

# Check events
kubectl describe hpa <name>

# Check Pod metrics
kubectl top pods
```

### 6. Use PodDisruptionBudget with VPA

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2  # ← Keep 2 Pods available during VPA updates
  selector:
    matchLabels:
      app: web
```

### 7. Test scaling before production

```bash
# Load test
kubectl run -it --rm load-generator \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://web; done"

# Watch scaling
kubectl get hpa -w
kubectl get pods -w
```

---

## 🎓 Key Takeaways

### Scaling Types

1. **Manual Scaling:** `kubectl scale deployment web --replicas=10`
2. **HPA (Horizontal):** Auto-scale number of Pods (2 → 10 Pods)
3. **VPA (Vertical):** Auto-adjust CPU/memory requests (100m → 500m)
4. **Cluster Autoscaler:** Auto-add/remove Nodes (3 → 5 Nodes)

### When to Use

| Scaling Type | Use Case | Example |
|--------------|----------|---------|
| **Manual** | Scheduled events, known patterns | Product launch, Black Friday |
| **HPA** | Dynamic traffic (web apps, APIs) | E-commerce, social media |
| **VPA** | Right-size resources (databases) | PostgreSQL, Redis |
| **Cluster Autoscaler** | Dynamic cluster capacity | Cloud environments |

### HPA Requirements

1. ✅ Metrics Server installed
2. ✅ Resource requests defined
3. ✅ Pods ready and healthy
4. ✅ Sufficient cluster capacity (or Cluster Autoscaler)

### HPA Algorithm

```
desiredReplicas = ceil[currentReplicas * (currentMetric / targetMetric)]
```

### Best Practices

- ✅ Set resource requests (required for HPA)
- ✅ Reasonable min (≥2 for HA) and max (cost control)
- ✅ Stabilization windows (avoid flapping)
- ✅ Don't mix HPA + VPA on same metric
- ✅ Use PodDisruptionBudget with VPA
- ✅ Monitor and test before production

### Commands

```bash
# Manual scaling
kubectl scale deployment web --replicas=10

# Create HPA
kubectl autoscale deployment web --min=2 --max=10 --cpu-percent=70

# Check HPA
kubectl get hpa
kubectl describe hpa <name>

# Check metrics
kubectl top nodes
kubectl top pods

# Delete HPA
kubectl delete hpa <name>
```

---

**Chúc mừng!** Hoàn thành **Phần 8: High Availability & Scaling** 🎉

Bạn đã học:
- ✅ Self-Healing (tự phục hồi)
- ✅ Health Checks (probes)
- ✅ Scaling (manual, HPA, VPA, Cluster Autoscaler)

**Bạn đã hoàn thành toàn bộ kiến thức cốt lõi của Kubernetes!** 🎉🎉🎉

👉 [**Phần 9: Next Steps - Lộ Trình Tiếp Theo**](../09-next-steps/README.md)

---

[⬅️ 8.2. Health Checks](./02-health-checks.md) | [⬆️ Phần 8](./README.md) | [➡️ 9. Next Steps](../09-next-steps/README.md) | [🏠 Mục Lục](../README.md)
