# 2.3. Worker Node - Nơi Chạy Workload

> Hiểu cách Worker Node hoạt động và chạy Pods

---

## 🎯 Mục Tiêu

- Hiểu các components trên Worker Node
- Biết cách kubelet quản lý Pods
- Nắm được Pod lifecycle
- Hiểu networking trên Node

---

## 🖥️ Worker Node Architecture

```
┌──────────────────────────────────────────────┐
│          WORKER NODE (Minion)                │
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │           Pods (Workloads)             │ │
│  │  ┌─────────┐  ┌─────────┐  ┌────────┐ │ │
│  │  │ Pod 1   │  │ Pod 2   │  │ Pod 3  │ │ │
│  │  │┌───────┐│  │┌───────┐│  │┌──────┐│ │ │
│  │  ││App    ││  ││Nginx  ││  ││Redis ││ │ │
│  │  │└───────┘│  │└───────┘│  │└──────┘│ │ │
│  │  └─────────┘  └─────────┘  └────────┘ │ │
│  └────────────────────────────────────────┘ │
│          ▲                                   │
│          │ (manages)                         │
│  ┌───────┴──────────────────────────────┐   │
│  │      kubelet (Node Agent)            │   │ ← Main component
│  │  • Watch for Pod assignments         │   │
│  │  • Run containers via CRI            │   │
│  │  • Report status to API Server       │   │
│  │  • Health checks                     │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │      kube-proxy                      │   │ ← Networking
│  │  • Network rules (iptables/IPVS)    │   │
│  │  • Service load balancing            │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │   Container Runtime                  │   │ ← Run containers
│  │   (Docker / containerd / CRI-O)      │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  OS: Linux (usually)                         │
└──────────────────────────────────────────────┘
```

---

## 1️⃣ kubelet

### 🎯 Vai Trò

**kubelet = Node agent, "Trưởng ca" tại mỗi Node**

- ✅ **Nhận Pod assignments** từ API Server
- ✅ **Manage Pod lifecycle** (start, stop, restart)
- ✅ **Health checks** (liveness, readiness probes)
- ✅ **Resource monitoring** (CPU, memory usage)
- ✅ **Report status** về API Server

### 🔄 kubelet Workflow

```
┌──────────────────────────────────────────┐
│ kubelet running on Node                  │
└───────────────┬──────────────────────────┘
                │
                │ (poll/watch)
                ▼
┌──────────────────────────────────────────┐
│ API Server: "Any Pods for this Node?"   │
└───────────────┬──────────────────────────┘
                │
                │ "Yes, run Pod X"
                ▼
┌──────────────────────────────────────────┐
│ kubelet:                                 │
│  1. Pull container image                │
│  2. Create container via Runtime        │
│  3. Start container                     │
│  4. Setup networking                    │
│  5. Mount volumes                       │
└───────────────┬──────────────────────────┘
                │
                │ (continuous monitoring)
                ▼
┌──────────────────────────────────────────┐
│ Health Checks:                           │
│  • Liveness probe → Restart if failed   │
│  • Readiness probe → Route traffic or not│
└───────────────┬──────────────────────────┘
                │
                │ (report status)
                ▼
┌──────────────────────────────────────────┐
│ API Server: "Pod X is Running"          │
│             "Pod Y failed health check" │
└──────────────────────────────────────────┘
```

### 🏢 Ví Dụ Thực Tế

**kubelet = Trưởng ca trong nhà máy**

```
Nhiệm vụ:
  1. Nhận công việc từ văn phòng (API Server)
     "Hôm nay ca sáng cần làm 3 sản phẩm"
  
  2. Chuẩn bị nguyên liệu (pull images)
     "Lấy vật liệu từ kho"
  
  3. Bắt đầu sản xuất (start containers)
     "Khởi động máy móc"
  
  4. Giám sát liên tục (health checks)
     "Kiểm tra máy móc có hoạt động tốt không"
  
  5. Báo cáo tiến độ (report status)
     "Sản phẩm 1 hoàn thành, sản phẩm 2 đang làm..."
  
  6. Xử lý sự cố (restart failed containers)
     "Máy 2 hỏng → Restart"
```

### 🔧 kubelet Responsibilities

#### Pod Lifecycle Management
```
Pod lifecycle phases:
  Pending   → kubelet pulls image, prepares
  Running   → Container(s) running
  Succeeded → Completed successfully (Job/CronJob)
  Failed    → Container(s) failed
  Unknown   → Can't determine state
```

#### Health Checks
**1. Liveness Probe:**
```yaml
# Check if container is alive
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  
# If failed 3 times → Restart container
```

**2. Readiness Probe:**
```yaml
# Check if container ready to serve traffic
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  
# If failed → Remove from Service endpoints
```

**3. Startup Probe:**
```yaml
# For slow-starting containers
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30  # 30 * 10s = 5 minutes timeout
  periodSeconds: 10
```

#### Resource Management
```yaml
resources:
  requests:    # Minimum guaranteed
    cpu: "500m"      # 0.5 CPU core
    memory: "512Mi"  # 512 MB
  limits:      # Maximum allowed
    cpu: "1000m"     # 1 CPU core
    memory: "1Gi"    # 1 GB
```

**kubelet enforces:**
- CPU throttling when limit exceeded
- Container killed (OOMKilled) if memory limit exceeded

#### Volume Management
```
kubelet mounts volumes:
  - emptyDir: Create temporary directory
  - hostPath: Mount from Node filesystem
  - PVC: Mount cloud disk (EBS, GCE PD, etc.)
  - ConfigMap/Secret: Mount as files
```

### 📊 Status Reporting

**kubelet reports to API Server:**
```
Node status:
  - CPU, memory, disk capacity
  - Running Pods
  - Node conditions (Ready, DiskPressure, MemoryPressure)
  
Pod status:
  - Container states (Running, Waiting, Terminated)
  - Restart counts
  - Resource usage
```

---

## 2️⃣ kube-proxy

### 🎯 Vai Trò

**kube-proxy = Network proxy trên mỗi Node**

- ✅ **Service networking** (ClusterIP, NodePort)
- ✅ **Load balancing** giữa Pod replicas
- ✅ **Network rules** (iptables, IPVS, userspace)

### 🏢 Ví Dụ Thực Tế

**kube-proxy = Tổng đài viên**

```
Service "web" có 3 Pods backend:
  - Pod A: 10.1.1.5:8080
  - Pod B: 10.1.1.6:8080
  - Pod C: 10.1.1.7:8080

Service ClusterIP: 10.0.0.100:80

kube-proxy tạo network rules:
  Request đến 10.0.0.100:80
  → Random pick 1 trong 3 Pods
  → Forward request
  
Giống tổng đài viên:
  - Khách gọi số tổng đài duy nhất
  - Tổng đài chuyển đến nhân viên rảnh
```

### 🔧 kube-proxy Modes

#### 1. iptables Mode (Default, phổ biến nhất)
```bash
# kube-proxy creates iptables rules
# Example: Service "web" with ClusterIP 10.0.0.100

# Rule 1: Intercept traffic to ClusterIP
iptables -A KUBE-SERVICES \
  -d 10.0.0.100/32 -p tcp --dport 80 \
  -j KUBE-SVC-WEB

# Rule 2: Load balance to Pods (random)
iptables -A KUBE-SVC-WEB -m statistic --mode random --probability 0.33 \
  -j KUBE-SEP-POD-A  # 10.1.1.5:8080

iptables -A KUBE-SVC-WEB -m statistic --mode random --probability 0.50 \
  -j KUBE-SEP-POD-B  # 10.1.1.6:8080

iptables -A KUBE-SVC-WEB \
  -j KUBE-SEP-POD-C  # 10.1.1.7:8080 (default)
```

**Pros:**
- Mature, stable
- Low overhead

**Cons:**
- Doesn't scale well (1000s of services)
- No real load balancing (just random)

#### 2. IPVS Mode (Recommended for large clusters)
```
Uses Linux IPVS (IP Virtual Server):
  - Better performance
  - More load balancing algorithms:
    • Round-robin
    • Least connection
    • Source hashing
  - Scales to 10,000+ services
```

#### 3. userspace Mode (Legacy, deprecated)
```
kube-proxy itself proxies traffic:
  - Performance overhead
  - Not recommended
```

### 🌐 Service Types Handling

**ClusterIP:**
```
Service: 10.0.0.100 (cluster-internal IP)
kube-proxy: Create rules to forward to Pod IPs
```

**NodePort:**
```
Service: 10.0.0.100 + NodePort 30080
kube-proxy: 
  - Listen on Node IP:30080
  - Forward to Pod IPs
  
Access from outside:
  http://NodeIP:30080 → Pod
```

**LoadBalancer:**
```
Cloud LB: 203.0.113.5
  ↓
NodePort: 30080 (created automatically)
  ↓
kube-proxy forwards to Pods
```

---

## 3️⃣ Container Runtime

### 🎯 Vai Trò

**Container Runtime = "Động cơ" chạy containers**

- ✅ **Pull images** từ registry
- ✅ **Create containers** từ images
- ✅ **Start/Stop containers**
- ✅ **Isolate containers** (namespaces, cgroups)

### 🔄 CRI (Container Runtime Interface)

**Kubernetes không phụ thuộc Docker:**

```
┌──────────────┐
│   kubelet    │
└──────┬───────┘
       │ (CRI - gRPC API)
       │
       ▼
┌──────────────────────────────────┐
│   Container Runtime              │
│   • Docker (via dockershim)      │ ← Deprecated
│   • containerd ⭐                 │ ← Recommended
│   • CRI-O                        │
└──────────────────────────────────┘
```

### 🐳 Runtime Options

#### 1. containerd (Recommended)
```
Lightweight, native K8s runtime
  - Graduated from Docker
  - Industry standard
  - Used by Docker itself

Installation:
  Most managed K8s (EKS, GKE, AKS) use containerd
```

#### 2. CRI-O
```
Designed specifically for K8s
  - Lightweight
  - OCI-compliant
  - Minimal

Use case:
  - Red Hat OpenShift
  - Embedded systems
```

#### 3. Docker (via dockershim - DEPRECATED)
```
Legacy support:
  - dockershim removed in K8s 1.24+
  - Use containerd instead

Migration:
  Docker images still work!
  Only runtime changes, not image format
```

---

## 📦 Pod Lifecycle On Node

### Full Flow

```
1. Scheduler assigns Pod to Node
   API Server: Update Pod.spec.nodeName = "node-1"

2. kubelet (on node-1) detects assignment
   kubelet watches: "New Pod for me!"

3. kubelet pulls image
   kubelet → Container Runtime: "Pull nginx:1.21"
   Runtime → Docker Hub: Download image

4. kubelet creates Pod sandbox
   Setup network namespace, IPC, etc.

5. kubelet starts init containers (if any)
   Wait for init containers to complete

6. kubelet starts main containers
   Container Runtime: Start nginx container

7. kubelet runs readiness probe
   Wait until app is ready

8. kubelet updates Pod status
   kubelet → API Server: "Pod is Running"

9. Continuous monitoring
   - Liveness probe every 10s
   - Resource usage monitoring
   - Log collection

10. Pod termination (when deleted)
    1. Send SIGTERM to containers
    2. Wait gracefully (default 30s)
    3. Send SIGKILL if still running
    4. Cleanup resources
```

### 🔍 Example: Pod Startup

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  initContainers:
  - name: init-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db:3306; do sleep 1; done']
  
  containers:
  - name: app
    image: nginx:1.21
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      periodSeconds: 5
```

**Execution:**
```
1. kubelet pulls busybox image
2. kubelet starts init-db container
3. init-db waits for DB (may take 30s)
4. init-db exits successfully
5. kubelet pulls nginx:1.21 image
6. kubelet starts app container
7. Wait 30s (initialDelaySeconds)
8. Start liveness probe (every 10s)
9. Start readiness probe (every 5s)
10. When readiness passes → Add to Service
```

---

## 🔗 Node Components Interaction

```
┌─────────────────────────────────────────┐
│          Control Plane                  │
│                                         │
│  ┌────────────┐                         │
│  │ API Server │                         │
│  └─────┬──────┘                         │
└────────┼────────────────────────────────┘
         │
         │ (1) Watch for Pod assignments
         │ (2) Report Node/Pod status
         │
┌────────┼────────────────────────────────┐
│ ┌──────▼──────┐    Worker Node          │
│ │   kubelet   │                         │
│ └──────┬──────┘                         │
│        │                                │
│        │ (3) Create Pod                 │
│        ▼                                │
│ ┌────────────────┐                      │
│ │Container Runtime│                     │
│ └────────┬───────┘                      │
│          │                              │
│          │ (4) Start containers         │
│          ▼                              │
│      ┌────────┐                         │
│      │  Pods  │                         │
│      └────────┘                         │
│          ▲                              │
│          │                              │
│          │ (5) Network traffic          │
│    ┌─────┴──────┐                       │
│    │ kube-proxy │                       │
│    └────────────┘                       │
│        ↑                                │
│        │ (6) Watch Services/Endpoints   │
└────────┼────────────────────────────────┘
         │
┌────────┼────────────────────────────────┐
│  ┌─────▼─────┐   Control Plane          │
│  │ API Server│                          │
│  └───────────┘                          │
└─────────────────────────────────────────┘
```

---

## 🎓 Key Takeaways

1. **kubelet:** Node agent, manages Pods, health checks
2. **kube-proxy:** Network proxy, Service load balancing
3. **Container Runtime:** Actual container execution (containerd recommended)
4. **Pod lifecycle:** Pull → Create → Start → Monitor → Report
5. **Health probes:** Liveness (restart), Readiness (traffic routing)
6. **Async communication:** Components watch API Server
7. **Node = Worker:** Executes workloads, reports status

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. kubelet làm gì khi được assign một Pod?
2. Sự khác biệt giữa liveness và readiness probe?
3. kube-proxy làm gì với Service?
4. Container Runtime nào được recommend?
5. Vẽ flow từ khi Pod được assign đến khi Running

---

## 🚀 Tiếp Theo

Bạn đã hoàn thành **Phần 2: Architecture**! 🎉

👉 [**Phần 3: Core Concepts - Khái Niệm Cốt Lõi**](../03-core-concepts/README.md)

---

[⬅️ 2.2. Control Plane](./02-control-plane.md) | [⬆️ Phần 2](./README.md) | [🏠 Mục Lục Chính](../README.md)


