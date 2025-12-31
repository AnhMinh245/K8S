# 2.1. Tổng Quan Kiến Trúc Kubernetes

> Hiểu big picture trước khi đi vào chi tiết từng thành phần

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **kiến trúc tổng thể** của Kubernetes
- ✅ Phân biệt **Control Plane** và **Worker Node**
- ✅ Hiểu **communication flow** giữa các components
- ✅ Biết **vai trò** của từng thành phần chính

---

## 🏗️ Kubernetes Cluster - Big Picture

### Cluster Là Gì?

**Kubernetes Cluster** = Tập hợp servers làm việc cùng nhau như một hệ thống thống nhất.

### Giải Thích Bằng Ví Dụ

**Cluster giống như một công ty:**

```
🏢 Công Ty (Cluster)
├── 🎯 Phòng Giám Đốc (Control Plane)
│   ├── CEO (API Server) - Nhận mọi yêu cầu
│   ├── CFO (etcd) - Quản lý dữ liệu/trạng thái
│   ├── HR Manager (Scheduler) - Phân công nhân viên
│   └── Operations Manager (Controller Manager) - Đảm bảo mọi thứ chạy đúng
│
└── 👷 Phòng Sản Xuất (Worker Nodes)
    ├── Worker Node 1 - Nhân viên làm việc
    ├── Worker Node 2 - Nhân viên làm việc  
    └── Worker Node 3 - Nhân viên làm việc
```

---

## 📐 Kiến Trúc Tổng Thể

### Diagram Cluster Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────────────────────────────────────────────┐    │
│  │         CONTROL PLANE (Bộ Não)                     │    │
│  │         Master Components                          │    │
│  ├────────────────────────────────────────────────────┤    │
│  │                                                    │    │
│  │  ┌──────────────┐  ┌──────────────┐             │    │
│  │  │ API Server   │  │    etcd      │             │    │
│  │  │ (Điểm vào)   │  │ (Database)   │             │    │
│  │  └──────┬───────┘  └──────────────┘             │    │
│  │         │                                         │    │
│  │  ┌──────┴────────┐  ┌──────────────┐            │    │
│  │  │  Scheduler    │  │ Controller   │            │    │
│  │  │ (Phân công)   │  │  Manager     │            │    │
│  │  └───────────────┘  │ (Giám sát)   │            │    │
│  │                     └──────────────┘            │    │
│  └────────────────────────────────────────────────────┘    │
│                           ↕                                 │
│              [Communication via API]                        │
│                           ↕                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │         WORKER NODES (Thợ Làm Việc)               │    │
│  │         Data Plane                                 │    │
│  ├────────────────────────────────────────────────────┤    │
│  │                                                    │    │
│  │  ┌─────────────────────────────────────────┐     │    │
│  │  │  NODE 1                                 │     │    │
│  │  │  ┌─────────────────────────────────┐   │     │    │
│  │  │  │ kubelet (Agent)                 │   │     │    │
│  │  │  │ kube-proxy (Network)            │   │     │    │
│  │  │  │ Container Runtime (Docker)      │   │     │    │
│  │  │  ├─────────────────────────────────┤   │     │    │
│  │  │  │  PODS (Applications)            │   │     │    │
│  │  │  │  ┌────┐ ┌────┐ ┌────┐          │   │     │    │
│  │  │  │  │Pod1│ │Pod2│ │Pod3│          │   │     │    │
│  │  │  │  └────┘ └────┘ └────┘          │   │     │    │
│  │  │  └─────────────────────────────────┘   │     │    │
│  │  └─────────────────────────────────────────┘     │    │
│  │                                                    │    │
│  │  ┌─────────────────────────────────────────┐     │    │
│  │  │  NODE 2                                 │     │    │
│  │  │  ┌─────────────────────────────────┐   │     │    │
│  │  │  │ kubelet + kube-proxy + Runtime  │   │     │    │
│  │  │  │  PODS: ┌────┐ ┌────┐            │   │     │    │
│  │  │  │        │Pod4│ │Pod5│            │   │     │    │
│  │  │  │        └────┘ └────┘            │   │     │    │
│  │  │  └─────────────────────────────────┘   │     │    │
│  │  └─────────────────────────────────────────┘     │    │
│  │                                                    │    │
│  │  ┌─────────────────────────────────────────┐     │    │
│  │  │  NODE 3                                 │     │    │
│  │  │  ┌─────────────────────────────────┐   │     │    │
│  │  │  │ kubelet + kube-proxy + Runtime  │   │     │    │
│  │  │  │  PODS: ┌────┐ ┌────┐ ┌────┐    │   │     │    │
│  │  │  │        │Pod6│ │Pod7│ │Pod8│    │   │     │    │
│  │  │  │        └────┘ └────┘ └────┘    │   │     │    │
│  │  │  └─────────────────────────────────┘   │     │    │
│  │  └─────────────────────────────────────────┘     │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘

USER/DEVELOPER
      ↓
   kubectl
      ↓
  API Server (Control Plane)
      ↓
  Worker Nodes (Pods running)
```

---

## 🎯 Control Plane vs Worker Nodes

### So Sánh Chi Tiết

| Đặc Điểm | Control Plane | Worker Nodes |
|----------|---------------|--------------|
| **Vai trò** | Bộ não - Ra quyết định | Thợ làm việc - Thực hiện công việc |
| **Số lượng** | 1-3 nodes (HA) | Nhiều nodes (10, 100, 1000+) |
| **Chạy gì?** | K8s system components | Application Pods |
| **Quan trọng** | Critical - Chết = cluster chết | Important - Chết 1 node OK |
| **Tài nguyên** | Ít CPU/RAM | Nhiều CPU/RAM cho apps |
| **Expose** | API Server public | Thường private |

---

## 🧠 Control Plane - Bộ Não

### Thành Phần Chính

**1. API Server (kube-apiserver)**
```
Vai trò: Cổng vào duy nhất của cluster
Ví dụ: Receptionist tại công ty

Làm gì:
├── Nhận tất cả requests (kubectl, dashboard, etc.)
├── Authenticate & authorize
├── Validate requests
└── Forward đến components khác

Communication:
User → kubectl → API Server → Other components
```

**2. etcd**
```
Vai trò: Database của cluster
Ví dụ: Kho lưu trữ hồ sơ công ty

Lưu gì:
├── Cluster state
├── Configuration data
├── Pods đang chạy ở đâu
├── Services có những Endpoints nào
└── Mọi thông tin về cluster

Đặc điểm:
✓ Distributed key-value store
✓ Consistent và highly-available
✓ Chỉ API Server có thể đọc/ghi
```

**3. Scheduler (kube-scheduler)**
```
Vai trò: Quyết định Pod chạy ở Node nào
Ví dụ: HR Manager phân công nhân viên

Làm gì:
1. Watch for new Pods (chưa assign Node)
2. Chọn Node phù hợp nhất dựa trên:
   ├── Resource available (CPU, RAM)
   ├── Node constraints (taints, tolerations)
   ├── Affinity/Anti-affinity rules
   └── Data locality
3. Assign Pod → Node
```

**4. Controller Manager (kube-controller-manager)**
```
Vai trò: Đảm bảo desired state = actual state
Ví dụ: Operations Manager giám sát mọi thứ

Gồm nhiều controllers:
├── Node Controller: Watch nodes health
├── Replication Controller: Đảm bảo đủ số Pods
├── Endpoints Controller: Populate Endpoints
├── Service Account Controller: Tạo default accounts
└── Nhiều controllers khác...

Logic: Continuous reconciliation loop
Desired: 3 Pods
Actual: 2 Pods (1 crashed)
→ Controller: Tạo thêm 1 Pod!
```

---

## 👷 Worker Nodes - Thợ Làm Việc

### Thành Phần Chính

**1. kubelet**
```
Vai trò: Agent chạy trên mỗi Node
Ví dụ: Supervisor của mỗi nhân viên

Làm gì:
├── Communicate với API Server
├── Watch for Pods assigned to this Node
├── Start/Stop containers (via Container Runtime)
├── Monitor Pod health
├── Report status về API Server
└── Mount volumes

Đặc điểm:
✓ Primary "node agent"
✓ Chạy trên MỌI node (kể cả control plane)
✓ Không manage containers không được K8s tạo
```

**2. kube-proxy**
```
Vai trò: Network proxy
Ví dụ: Nhân viên bưu điện - Forward thư/gói hàng

Làm gì:
├── Maintain network rules trên Node
├── Enable Pod-to-Pod communication
├── Implement Service abstraction
├── Load balance traffic đến Pods
└── Support ClusterIP, NodePort, LoadBalancer

Implementation modes:
├── iptables (phổ biến)
├── IPVS (performance cao hơn)
└── userspace (legacy)
```

**3. Container Runtime**
```
Vai trò: Chạy containers
Ví dụ: Máy móc để nhân viên làm việc

Options:
├── Docker (phổ biến nhất)
├── containerd (lightweight)
├── CRI-O (OCI-compliant)
└── Others...

Làm gì:
├── Pull images từ registry
├── Start/stop containers
├── Manage container lifecycle
└── Resource isolation
```

---

## 🔄 Communication Flow - Luồng Hoạt Động

### Scenario: Deploy Application

**Step-by-step workflow:**

```
┌──────────────────────────────────────────────────────────┐
│  1. USER CREATES DEPLOYMENT                              │
└──────────────────────────────────────────────────────────┘
$ kubectl create deployment webapp --image=nginx --replicas=3
                    ↓
┌──────────────────────────────────────────────────────────┐
│  2. kubectl → API SERVER                                 │
│     "Tạo Deployment với 3 Pods"                          │
└──────────────────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────────┐
│  3. API SERVER                                           │
│     ✓ Authenticate user                                  │
│     ✓ Authorize request                                  │
│     ✓ Validate Deployment spec                           │
│     ✓ Write to etcd: "Desired state: 3 Pods"           │
└──────────────────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────────┐
│  4. DEPLOYMENT CONTROLLER (watching API Server)          │
│     "New Deployment detected!"                           │
│     → Tạo ReplicaSet (owner của Pods)                    │
│     → API Server: "Tạo ReplicaSet với 3 Pods"          │
└──────────────────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────────┐
│  5. REPLICASET CONTROLLER                                │
│     "New ReplicaSet detected!"                           │
│     → API Server: "Tạo 3 Pods"                          │
└──────────────────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────────┐
│  6. API SERVER writes to etcd                            │
│     "3 Pods cần được tạo (status: Pending)"             │
└──────────────────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────────┐
│  7. SCHEDULER (watching for Pending Pods)                │
│     "3 Pods chưa có Node!"                               │
│                                                          │
│     Evaluate Nodes:                                      │
│     Node 1: CPU 40%, RAM 50% ✓                          │
│     Node 2: CPU 30%, RAM 40% ✓✓ (best fit!)            │
│     Node 3: CPU 60%, RAM 70% ✓                          │
│                                                          │
│     Decision:                                            │
│     ├─ Pod1 → Node 2                                     │
│     ├─ Pod2 → Node 1                                     │
│     └─ Pod3 → Node 2                                     │
│                                                          │
│     → API Server: Update Pod specs với Node assignment  │
└──────────────────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────────┐
│  8. KUBELET on Node 2 (watching API Server)              │
│     "2 Pods assigned to me!"                             │
│                                                          │
│     For each Pod:                                        │
│     1. Pull image: nginx                                 │
│     2. Create container via Container Runtime            │
│     3. Start container                                   │
│     4. Monitor container                                 │
│     5. Report status → API Server                        │
└──────────────────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────────┐
│  9. KUBELET on Node 1 (same process)                     │
│     "1 Pod assigned to me!"                              │
│     → Pull, create, start, monitor                       │
└──────────────────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────────┐
│  10. API SERVER updates etcd                             │
│      Pod1: Running on Node 2 ✅                          │
│      Pod2: Running on Node 1 ✅                          │
│      Pod3: Running on Node 2 ✅                          │
└──────────────────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────────┐
│  11. USER CHECKS STATUS                                  │
└──────────────────────────────────────────────────────────┘
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
webapp-7d8bc4c5d-abc12   1/1     Running   0          30s
webapp-7d8bc4c5d-def34   1/1     Running   0          30s
webapp-7d8bc4c5d-ghi56   1/1     Running   0          30s

✅ Deployment successful!
```

---

## 💡 Tại Sao Kiến Trúc Này?

### Design Principles

**1. Separation of Concerns**
```
Control Plane: Quyết định
Worker Nodes: Thực hiện

Lợi ích:
✓ Scale riêng biệt
✓ Failure isolation
✓ Easier maintenance
```

**2. Declarative API**
```
Bạn nói: "Tôi muốn 3 Pods"
K8s làm: "OK, để tôi lo!"

Không cần nói:
❌ "Tạo Pod1 trên Node1"
❌ "Tạo Pod2 trên Node2"
❌ "Config networking..."

K8s tự động handle tất cả!
```

**3. Watch & Reconciliation Loop**
```
Controllers liên tục:
1. Watch actual state
2. Compare với desired state
3. Take action nếu khác nhau
4. Repeat

→ Self-healing automatic!
```

**4. API-Driven**
```
Mọi interaction qua API Server:
✓ Centralized control
✓ Auditable
✓ Secure (authn/authz)
✓ Extensible
```

---

## 🎓 Kiểm Tra Hiểu Biết

### Câu Hỏi Tự Kiểm Tra

**1. Control Plane và Worker Nodes khác nhau như thế nào?**
<details>
<summary>Xem đáp án</summary>

**Control Plane (Bộ não):**
- Vai trò: Ra quyết định, điều phối
- Components: API Server, etcd, Scheduler, Controller Manager
- Chạy: K8s system components
- Quan trọng: Critical - chết = cluster chết

**Worker Nodes (Thợ):**
- Vai trò: Thực hiện công việc, chạy applications
- Components: kubelet, kube-proxy, Container Runtime
- Chạy: Application Pods
- Redundant: Chết 1 node, apps vẫn OK trên nodes khác
</details>

**2. API Server làm gì?**
<details>
<summary>Xem đáp án</summary>

API Server là cổng vào duy nhất của cluster:
1. Nhận tất cả requests (kubectl, dashboard)
2. Authenticate & authorize
3. Validate requests
4. Write/Read từ etcd
5. Forward requests đến components khác

Analogy: Receptionist/Switchboard operator
</details>

**3. Scheduler quyết định Pod chạy ở đâu dựa trên gì?**
<details>
<summary>Xem đáp án</summary>

Scheduler evaluate dựa trên:
1. **Resource availability**: CPU, RAM available trên node
2. **Resource requests**: Pod cần bao nhiêu CPU/RAM
3. **Node constraints**: Taints, tolerations, node selectors
4. **Affinity rules**: Pod muốn/không muốn chạy gần Pod nào
5. **Data locality**: Data ở đâu (optimize network)
6. **Priority**: Pod nào priority cao hơn

Choose node có score cao nhất!
</details>

**4. Vẽ flow khi bạn run `kubectl create deployment`**
<details>
<summary>Xem đáp án</summary>

```
kubectl 
  → API Server (validate, write etcd)
    → Deployment Controller (create ReplicaSet)
      → ReplicaSet Controller (create Pods)
        → API Server (write Pods to etcd)
          → Scheduler (assign Pods to Nodes)
            → kubelet on Nodes (create containers)
              → Containers running!
```
</details>

---

## 💪 Bài Tập Thực Hành

### Bài 1: Identify Components

**Tình huống:** Các components sau thuộc Control Plane hay Worker Node?

1. API Server
2. kubelet
3. etcd
4. kube-proxy
5. Scheduler
6. Container Runtime
7. Controller Manager

<details>
<summary>Xem đáp án</summary>

**Control Plane:**
- API Server ✓
- etcd ✓
- Scheduler ✓
- Controller Manager ✓

**Worker Node:**
- kubelet ✓
- kube-proxy ✓
- Container Runtime ✓
</details>

---

### Bài 2: Trace the Flow

**Pod crash, vẽ flow K8s tự phục hồi:**

<details>
<summary>Xem đáp án</summary>

```
1. Container crashes
   ↓
2. kubelet detects (health check fail)
   ↓
3. kubelet reports to API Server
   "Pod X on Node Y is dead"
   ↓
4. API Server writes to etcd
   "Pod X: status = Failed"
   ↓
5. ReplicaSet Controller watches API
   "Desired: 3 Pods, Actual: 2 Pods"
   "Need to create 1 Pod!"
   ↓
6. ReplicaSet Controller → API Server
   "Create new Pod"
   ↓
7. API Server writes to etcd
   "New Pod: status = Pending"
   ↓
8. Scheduler watches for Pending Pods
   "New Pod needs a Node!"
   Evaluate nodes → Choose Node Z
   ↓
9. Scheduler → API Server
   "Assign Pod to Node Z"
   ↓
10. kubelet on Node Z watches API
    "New Pod assigned to me!"
    Pull image → Create container → Start
    ↓
11. kubelet → API Server
    "Pod is Running!"
    ↓
12. Self-healing complete! ✅
```
</details>

---

## 🎯 Key Takeaways

### Ghi Nhớ 5 Điều Quan Trọng

1. **Cluster = Control Plane + Worker Nodes**
   - Control Plane: Bộ não (quyết định)
   - Worker Nodes: Thợ làm việc (thực hiện)

2. **API Server = Cổng vào duy nhất**
   - Mọi request đều qua API Server
   - Tương tác với etcd
   - Central hub

3. **etcd = Database của cluster**
   - Lưu mọi state
   - Single source of truth
   - Highly available

4. **Scheduler = Phân công thông minh**
   - Quyết định Pod chạy ở Node nào
   - Dựa trên resources, constraints, affinity

5. **Controllers = Reconciliation loops**
   - Watch desired vs actual state
   - Take action to match
   - Self-healing mechanism

---

## 📚 Thuật Ngữ Cần Nhớ

| Thuật Ngữ | Tiếng Việt | Ý Nghĩa |
|-----------|------------|---------|
| **Cluster** | Cluster | Tập hợp servers hoạt động như một hệ thống |
| **Control Plane** | Control Plane | Bộ não của cluster (master) |
| **Worker Node** | Worker Node | Server chạy application Pods |
| **API Server** | API Server | Cổng vào duy nhất của cluster |
| **etcd** | etcd | Database key-value lưu cluster state |
| **Scheduler** | Scheduler | Component phân công Pods to Nodes |
| **Controller** | Controller | Component đảm bảo desired state |
| **kubelet** | kubelet | Agent chạy trên mỗi Node |
| **kube-proxy** | kube-proxy | Network proxy trên mỗi Node |

---

## 🚀 Tiếp Theo

Bạn đã hiểu kiến trúc tổng thể của Kubernetes!

**Next:** [2.2. Control Plane - Chi Tiết →](./02-control-plane.md)

Ở phần tiếp theo, chúng ta sẽ deep dive vào từng component của Control Plane, hiểu chi tiết cách chúng hoạt động.

---

[⬅️ Phần 1: Introduction](../01-introduction/README.md) | [🏠 Mục Lục Chính](../README.md) | [📂 Phần 2: Architecture](./README.md) | [➡️ 2.2. Control Plane](./02-control-plane.md)
