# 2.1. Tổng Quan Kiến Trúc Kubernetes

> Hiểu big picture của K8s architecture

---

## 🎯 Mục Tiêu

- Hiểu kiến trúc tổng thể của Kubernetes cluster
- Nắm được master-worker model
- Biết cách các components communicate
- Hiểu design principles của K8s

---

## 🏗️ Kiến Trúc Tổng Thể

### Kubernetes Cluster = Control Plane + Worker Nodes

```
┌────────────────────────────────────────────┐
│         KUBERNETES CLUSTER                 │
│                                            │
│  ┌─────────────────────────────────────┐  │
│  │      CONTROL PLANE (Master)         │  │ ← "Bộ não"
│  │  • API Server                       │  │   Ra quyết định
│  │  • etcd                             │  │   Quản lý state
│  │  • Scheduler                        │  │   Không chạy app
│  │  • Controller Manager               │  │
│  └─────────────────────────────────────┘  │
│                    ▲                       │
│                    │                       │
│                    │ (communicate)         │
│                    ▼                       │
│  ┌─────────────────────────────────────┐  │
│  │         WORKER NODES                │  │ ← "Người lao động"
│  │  ┌──────────┐  ┌──────────┐         │  │   Chạy app thực tế
│  │  │ Node 1   │  │ Node 2   │  ...    │  │   Execute workloads
│  │  │ (Server) │  │ (Server) │         │  │
│  │  └──────────┘  └──────────┘         │  │
│  └─────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

---

## 🏢 Ví Dụ Thực Tế: Công Ty

Hãy tưởng tượng Kubernetes cluster như một **công ty**:

### 🏛️ Control Plane = Văn Phòng Điều Hành

**Vai trò:**
- Ra quyết định chiến lược
- Quản lý tài nguyên
- Giám sát hoạt động
- KHÔNG làm việc trực tiếp với khách hàng

**Thành viên:**
- **CEO (API Server):** Người nhận mọi requests, điều phối
- **Kế toán (etcd):** Lưu trữ mọi thông tin tài chính, nhân sự
- **HR (Scheduler):** Phân công nhân viên vào dự án
- **Giám sát viên (Controller Manager):** Đảm bảo mọi việc đúng kế hoạch

### 👷 Worker Nodes = Nhân Viên Thực Thi

**Vai trò:**
- Làm việc thực tế
- Phục vụ khách hàng
- Báo cáo về văn phòng

**Mỗi nhân viên (Node) có:**
- **Trưởng nhóm (kubelet):** Quản lý công việc trong nhóm
- **Điện thoại viên (kube-proxy):** Điều hướng cuộc gọi
- **Công cụ làm việc (Container Runtime):** Docker, containerd...

---

## 🔄 Communication Flow

### Scenario: Deploy một ứng dụng

```
1. User → kubectl apply -f app.yaml
   "Tôi muốn deploy 3 replicas của app"

2. kubectl → API Server (Control Plane)
   "Nhận yêu cầu, validate, lưu vào etcd"

3. API Server → etcd
   "Lưu desired state: 3 replicas của app"

4. Controller Manager → API Server (polling)
   "Có gì mới không?"
   API Server: "Cần 3 replicas app, hiện có 0"

5. Controller Manager → API Server
   "Tạo 3 Pods"

6. Scheduler → API Server (watch)
   "Có Pods cần schedule?"
   API Server: "Có 3 Pods pending"

7. Scheduler → API Server
   "Pod 1 → Node A, Pod 2 → Node B, Pod 3 → Node C"
   (Based on resources available)

8. kubelet (trên Node A) → API Server (polling)
   "Có công việc cho tôi không?"
   API Server: "Chạy Pod 1"

9. kubelet → Container Runtime
   "Pull image và start container"

10. kubelet → API Server
    "Pod 1 đang running"

11. API Server → etcd
    "Update current state: 1/3 Pods running"

... Lặp lại cho Node B và Node C ...

12. Current state = Desired state (3/3 running) ✅
```

### Key Points

1. **Mọi thứ đi qua API Server:** Single point of entry
2. **etcd = Source of truth:** Lưu mọi state
3. **Controllers watch API Server:** Liên tục giám sát
4. **kubelet pulls work:** Không phải push
5. **Declarative:** Bạn khai báo "muốn gì", K8s tự xử lý "làm thế nào"

---

## 🎨 Design Principles Của Kubernetes

### 1. **Declarative Configuration**

**Imperative (Traditional):**
```bash
# Bạn chỉ đạo từng bước
1. Create container 1
2. Wait for it to be ready
3. Create container 2
4. Configure load balancer
5. Update DNS
...
```

**Declarative (Kubernetes way):**
```yaml
# Bạn khai báo desired state
desired_state:
  app: web
  replicas: 3
  version: v1.2

# K8s tự động làm mọi thứ để đạt được state này
```

**Ví dụ thực tế:**
```
Imperative: "Đi thẳng 500m, rẽ phải, qua cầu, rẽ trái..."
Declarative: "Đi đến địa chỉ 123 Main St" → GPS tự tính đường
```

---

### 2. **Control Loop (Reconciliation Loop)**

**Cách hoạt động:**
```
loop forever:
  current_state = get_from_etcd()
  desired_state = get_from_user()
  
  if current_state != desired_state:
    take_action_to_match()
  
  sleep(interval)
```

**Ví dụ:**
```
Desired: 3 Pods running
Current: 2 Pods running (1 Pod crashed)

Controller detects difference:
  → Creates 1 new Pod
  
Current: 3 Pods running ✅
```

**Tương tự:**
- **Thermostat:** Desired temp = 22°C, current = 20°C → Turn on heater
- **Cruise control:** Desired speed = 100km/h, current = 95km/h → Accelerate

---

### 3. **API-Driven Architecture**

**Everything is an API call:**
```
User action          → API call
Controller logic     → API call
Monitoring           → API call
External tools       → API call
```

**Benefits:**
- ✅ **Extensible:** Easy to add new features
- ✅ **Programmatic:** Automate everything
- ✅ **Consistent:** Same interface for everything
- ✅ **Observable:** Audit all actions

---

### 4. **Distributed System**

**Kubernetes là distributed system:**

**Characteristics:**
- Multiple machines work together
- No single point of failure (với HA setup)
- Eventual consistency
- Network partitions possible

**Challenges:**
- More complex than single-server
- Network issues
- Split-brain scenarios
- Debugging harder

**Solutions K8s provides:**
- Leader election (etcd, API server)
- Health checks
- Retry logic
- Graceful degradation

---

### 5. **Modularity & Extensibility**

**K8s is modular:**

```
Core K8s:
  ├─ API Server
  ├─ Scheduler
  ├─ Controller Manager
  └─ ...

Pluggable components:
  ├─ Container Runtime (Docker, containerd, CRI-O)
  ├─ Network Plugin (Calico, Flannel, Weave)
  ├─ Storage Plugin (EBS, GCE PD, NFS)
  └─ Custom Controllers (Operators)
```

**Extension points:**
- **CRI (Container Runtime Interface):** Swap container runtime
- **CNI (Container Network Interface):** Swap network solution
- **CSI (Container Storage Interface):** Swap storage provider
- **Custom Resources:** Extend K8s API
- **Operators:** Custom automation logic

---

## 📦 Components Overview

### Control Plane Components

| Component | Vai Trò | Ví Dụ Thực Tế |
|-----------|---------|---------------|
| **API Server** | Gateway, authentication, authorization | CEO công ty, nhận mọi requests |
| **etcd** | Distributed key-value store | Database, sổ sách công ty |
| **Scheduler** | Assign Pods to Nodes | HR phân công nhân viên |
| **Controller Manager** | State reconciliation | Giám sát viên đảm bảo KPI |
| **Cloud Controller** | Cloud integration | Liên lạc với cloud providers |

### Node Components

| Component | Vai Trò | Ví Dụ Thực Tế |
|-----------|---------|---------------|
| **kubelet** | Node agent, manage Pods | Trưởng nhóm quản lý công việc |
| **kube-proxy** | Network proxy, load balancing | Tổng đài viên điều hướng calls |
| **Container Runtime** | Run containers | Công cụ thực tế làm việc |

---

## 🔍 Deep Dive: Control Plane vs Worker Node

### Control Plane (Master)

**Đặc điểm:**
- ❌ **KHÔNG chạy application workloads** (by default)
- ✅ **Chạy management components**
- ✅ **Ra quyết định**
- ✅ **Lưu trữ state**

**High Availability:**
- Production: 3 hoặc 5 master nodes
- Etcd cluster: 3, 5, hoặc 7 members
- API Server: Active-active (load balanced)
- Scheduler, Controller Manager: Active-passive (leader election)

**Hardware requirements (production):**
```
CPU: 4 cores minimum
RAM: 8 GB minimum
Disk: SSD, fast I/O (for etcd)
Network: Low latency
```

---

### Worker Node

**Đặc điểm:**
- ✅ **Chạy application workloads** (Pods)
- ❌ **KHÔNG ra quyết định**
- ✅ **Nhận lệnh từ Control Plane**
- ✅ **Báo cáo status**

**Scaling:**
- Có thể có hàng trăm/ngàn worker nodes
- Horizontal scaling
- Heterogeneous: Nodes có thể khác size

**Hardware requirements:**
```
CPU: Tùy workload (2-64 cores)
RAM: Tùy workload (4-256 GB)
Disk: For logs, temp storage
Network: High bandwidth
```

---

## 🌐 Network Architecture

### Kubernetes Network Model

**Principles:**
1. **Every Pod gets its own IP address**
   - No NAT between Pods
   - Flat network space

2. **Pods can communicate with all other Pods**
   - Without NAT
   - Across nodes

3. **Nodes can communicate with all Pods**
   - Without NAT

4. **The IP a Pod sees itself as is the same IP others see it as**

**Implementation:**
```
┌────────────────────────────────────────┐
│            Node 1                      │
│  Pod A (IP: 10.1.1.5)                  │
│  Pod B (IP: 10.1.1.6)                  │
└────────────────────────────────────────┘
            │
            │ (Network fabric - CNI plugin)
            │
┌────────────────────────────────────────┐
│            Node 2                      │
│  Pod C (IP: 10.1.2.5)                  │
│  Pod D (IP: 10.1.2.6)                  │
└────────────────────────────────────────┘

Pod A can directly ping Pod C at 10.1.2.5
```

**CNI Plugins (Network implementations):**
- **Calico:** Network policies, BGP
- **Flannel:** Simple overlay network
- **Weave:** Mesh network
- **Cilium:** eBPF-based, advanced

---

## 💾 Storage Architecture

**Kubernetes storage model:**

```
┌──────────────────┐
│   Application    │
│   (Container)    │
└────────┬─────────┘
         │ mount
         ▼
┌──────────────────┐
│   Volume         │ ← Abstract storage
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Storage Backend  │ ← Actual storage
│ (EBS, NFS, etc)  │
└──────────────────┘
```

**Storage types:**
1. **Ephemeral:** emptyDir, hostPath (mất khi Pod xóa)
2. **Persistent:** PV/PVC (giữ data dù Pod xóa)
3. **Projected:** ConfigMap, Secret (mount vào Pod)

---

## 🛡️ Security Architecture

### Defense in Depth

```
┌───────────────────────────────────────┐
│  Layer 1: Network Security           │ ← Network Policies
├───────────────────────────────────────┤
│  Layer 2: Authentication              │ ← Who are you?
├───────────────────────────────────────┤
│  Layer 3: Authorization (RBAC)        │ ← What can you do?
├───────────────────────────────────────┤
│  Layer 4: Admission Control           │ ← Is this allowed?
├───────────────────────────────────────┤
│  Layer 5: Pod Security                │ ← Runtime security
├───────────────────────────────────────┤
│  Layer 6: Secret Management           │ ← Encrypt data
└───────────────────────────────────────┘
```

---

## 🎓 Key Takeaways

1. **Master-Worker model:** Control Plane ra quyết định, Workers thực thi
2. **API Server = Central hub:** Mọi communication đi qua đây
3. **etcd = Source of truth:** Lưu tất cả state
4. **Declarative:** Khai báo desired state, K8s reconcile
5. **Control loops:** Liên tục so sánh current vs desired state
6. **Distributed system:** Multiple machines, no SPOF (với HA)
7. **Extensible:** Plugin architecture, dễ customize

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. Vẽ kiến trúc K8s cluster với Control Plane và Worker Nodes
2. Giải thích control loop hoạt động như thế nào
3. Tại sao mọi request phải đi qua API Server?
4. etcd lưu trữ thông tin gì?
5. Sự khác biệt chính giữa Control Plane và Worker Node?
6. Declarative vs Imperative configuration khác nhau như thế nào?

---

## 🚀 Tiếp Theo

👉 [2.2. Control Plane - Bộ Não Của Cluster](./02-control-plane.md)

Chúng ta sẽ đi sâu vào từng component của Control Plane.

---

[⬅️ Về Phần 2: Architecture](./README.md) | [🏠 Mục Lục Chính](../README.md)

