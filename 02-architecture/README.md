# 📘 Phần 2: Architecture - Kiến Trúc Kubernetes

> Hiểu sâu cách Kubernetes hoạt động bên trong

---

## 🎯 Mục Tiêu Phần Này

Sau khi hoàn thành Phần 2, bạn sẽ:

✅ **Hiểu kiến trúc tổng thể** của Kubernetes Cluster  
✅ **Nắm rõ vai trò** của từng component  
✅ **Biết cách components tương tác** với nhau  
✅ **Troubleshoot được** architecture-level issues  
✅ **Có foundation vững** để làm việc với K8s  

---

## 📚 Nội Dung

### [2.1. Tổng Quan Kiến Trúc](./01-overview.md) ⭐⭐⭐⭐⭐

**Thời gian:** 45-60 phút

**Nội dung:**
- Kubernetes Cluster là gì
- Big picture: Control Plane + Worker Nodes
- Kiến trúc tổng thể (detailed diagram)
- Control Plane vs Worker Nodes
- Communication flow overview
- Design principles (tại sao kiến trúc này?)

**Key Concepts:**
```
✓ Cluster = Control Plane + Worker Nodes
✓ Separation of concerns (brain vs workers)
✓ Declarative API pattern
✓ Watch & reconciliation loop
✓ API-driven architecture
```

**Bài Tập:**
- Identify components (Control Plane hay Worker?)
- Vẽ flow: kubectl create deployment
- Trace self-healing workflow

---

### [2.2. Control Plane - Bộ Não](./02-control-plane.md) ⭐⭐⭐⭐⭐

**Thời gian:** 60-90 phút (CỰC KỲ QUAN TRỌNG!)

**Nội dung:**
- **API Server**: Cổng vào duy nhất, Authentication/Authorization
- **etcd**: Database của cluster, backup strategies
- **Scheduler**: Phân công Pods to Nodes (filtering + scoring)
- **Controller Manager**: Reconciliation loops, multiple controllers
- **Cloud Controller Manager**: Cloud integration

**Key Concepts:**
```
✓ API Server = Central hub, etcd gateway
✓ etcd = Single source of truth, must backup!
✓ Scheduler = Smart placement (resources, constraints)
✓ Controllers = Desired state = Actual state
✓ Watch pattern = Event-driven architecture
```

**Deep Dives:**
- Authentication & Authorization flow
- etcd data structure
- Scheduler scoring algorithm
- Controller reconciliation loop
- Complete deployment flow (10 steps)

**Commands:**
```bash
# Monitor Control Plane
kubectl get pods -n kube-system
kubectl logs -n kube-system kube-apiserver-<node>
kubectl get --raw /metrics

# etcd backup
etcdctl snapshot save backup.db
```

---

### [2.3. Worker Nodes - Nơi Chạy Apps](./03-worker-nodes.md) ⭐⭐⭐⭐⭐

**Thời gian:** 60-75 phút

**Nội dung:**
- **kubelet**: Node agent, Pod lifecycle management
- **kube-proxy**: Network proxy, Service implementation
- **Container Runtime**: Docker/containerd/CRI-O
- Node components interaction
- Troubleshooting Node issues

**Key Concepts:**
```
✓ kubelet = Node supervisor (manage Pods)
✓ kube-proxy = Network manager (iptables/IPVS)
✓ Container Runtime = Execute containers
✓ CRI = Container Runtime Interface
✓ Linux kernel features (namespaces, cgroups)
```

**Deep Dives:**
- kubelet Pod lifecycle (8 steps)
- kube-proxy iptables vs IPVS
- containerd architecture
- Container isolation mechanisms
- Complete Pod startup flow (11 steps)

**Troubleshooting:**
```bash
# Node issues
kubectl get nodes
kubectl describe node <node>
sudo systemctl status kubelet

# Network issues
kubectl get svc
kubectl get endpoints
sudo iptables-save | grep KUBE

# Container runtime
sudo crictl ps
sudo crictl logs <container-id>
```

---

## 🗺️ Learning Path

### Recommended Order

```
1. Start: README.md (file này)
   ↓
2. 2.1. Overview (big picture)
   ↓  
3. 2.2. Control Plane (deep dive)
   ↓
4. 2.3. Worker Nodes (execution)
   ↓
5. Next: Phần 3 - Core Concepts
```

### Cách Học Hiệu Quả

**1. Visualize Architecture**
```
✓ Vẽ lại diagrams bằng tay
✓ Annotate với notes riêng
✓ So sánh với analogies (công ty, nhà hàng)
```

**2. Trace Flows**
```
✓ Follow từng bước trong workflows
✓ Vẽ arrows giữa components
✓ Hiểu "tại sao" mỗi step
```

**3. Hands-on Later**
```
✓ Hiểu concepts trước
✓ Setup cluster ở Phần 3
✓ Practice commands sau khi hiểu theory
```

---

## 🏗️ Architecture Summary

### Complete Picture

```
┌─────────────────────────────────────────────────────────┐
│                 KUBERNETES CLUSTER                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  USER/DEVELOPER                                         │
│       ↓                                                 │
│    kubectl                                              │
│       ↓                                                 │
│  ┌──────────────────────────────────────────────────┐  │
│  │  CONTROL PLANE (Brain)                           │  │
│  │                                                  │  │
│  │  API Server ← etcd (Database)                   │  │
│  │      ↕                                           │  │
│  │  Scheduler + Controllers                         │  │
│  └──────────────────────────────────────────────────┘  │
│       ↕ (Commands & Status)                            │
│  ┌──────────────────────────────────────────────────┐  │
│  │  WORKER NODES (Workers)                          │  │
│  │                                                  │  │
│  │  Node 1, 2, 3...                                │  │
│  │  ├── kubelet (Pod manager)                      │  │
│  │  ├── kube-proxy (Network)                       │  │
│  │  ├── Container Runtime                          │  │
│  │  └── PODS (Applications)                        │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Location | Responsibility |
|-----------|----------|----------------|
| **API Server** | Control Plane | Cổng vào, authn/authz, etcd gateway |
| **etcd** | Control Plane | Database, store all state |
| **Scheduler** | Control Plane | Assign Pods to Nodes |
| **Controllers** | Control Plane | Reconcile desired vs actual state |
| **kubelet** | Worker Node | Manage Pods on Node |
| **kube-proxy** | Worker Node | Network proxy, Services |
| **Container Runtime** | Worker Node | Run containers |

---

## 🎓 Self-Assessment

### Checkpoint: Bạn Đã Hiểu Chưa?

**1. Vẽ kiến trúc K8s từ memory**
<details>
<summary>Check points</summary>

Should include:
- Control Plane: API Server, etcd, Scheduler, Controllers
- Worker Nodes: kubelet, kube-proxy, Container Runtime
- Arrows showing communication
- User/kubectl at top
</details>

**2. Giải thích flow: kubectl create deployment**
<details>
<summary>Check answer</summary>

```
1. kubectl → API Server
2. API Server validates, writes to etcd
3. Deployment Controller creates ReplicaSet
4. ReplicaSet Controller creates Pods
5. Scheduler assigns Pods to Nodes
6. kubelet on Nodes pulls images, starts containers
7. Pods running!
```
</details>

**3. Component nào làm gì?**

Match:
1. API Server
2. etcd
3. Scheduler
4. ReplicaSet Controller
5. kubelet
6. kube-proxy

Tasks:
a. Maintain Pod replicas
b. Store cluster state
c. Run containers
d. Assign Pods to Nodes
e. Implement Services
f. Validate requests

<details>
<summary>Check answers</summary>

1. API Server → f. Validate requests
2. etcd → b. Store cluster state
3. Scheduler → d. Assign Pods to Nodes
4. ReplicaSet Controller → a. Maintain Pod replicas
5. kubelet → c. Run containers
6. kube-proxy → e. Implement Services
</details>

**4. Troubleshooting scenarios**

**Scenario A:** Node shows NotReady
<details>
<summary>Debug steps</summary>

```bash
1. kubectl describe node <node>
2. Check kubelet: systemctl status kubelet
3. Check logs: journalctl -u kubelet
4. Check disk/memory: df -h, free -m
5. Check network connectivity
```
</details>

**Scenario B:** Pod stuck Pending
<details>
<summary>Debug steps</summary>

```bash
1. kubectl describe pod <pod>
2. Check events: "insufficient cpu" / "no nodes available"
3. kubectl get nodes (enough resources?)
4. Check node taints/tolerations
5. Check PVC if uses storage
```
</details>

---

## 🎯 Key Takeaways - Phần 2

### 10 Điều Quan Trọng Nhất

**1. Cluster Architecture**
```
Cluster = Control Plane (brain) + Worker Nodes (workers)
Separation of concerns = Scalable + Maintainable
```

**2. API Server = Central Hub**
```
All communication goes through API Server
Only component talking to etcd
Authentication, Authorization, Validation
```

**3. etcd = Critical Database**
```
Stores ALL cluster state
Distributed, consistent
MUST backup regularly!
```

**4. Scheduler = Smart Placement**
```
Filter (can run?) → Score (best fit?) → Bind
Consider: resources, constraints, affinity
Optimize cluster utilization
```

**5. Controllers = Self-Healing**
```
Watch loop: Desired vs Actual
Automatic reconciliation
Multiple controllers for different resources
```

**6. kubelet = Pod Manager**
```
Agent on each Node
Manages Pod lifecycle
Reports status to API Server
```

**7. kube-proxy = Service Implementation**
```
Maintains network rules (iptables/IPVS)
Enables Service abstraction
Load balances to Pods
```

**8. Container Runtime**
```
containerd recommended (not Docker!)
CRI interface với kubelet
Manages container lifecycle
```

**9. Watch Pattern Everywhere**
```
Components watch API Server for changes
Event-driven architecture
React to state changes
```

**10. Declarative API**
```
Describe desired state
K8s makes it happen
Reconciliation = continuous convergence
```

---

## 📚 Thuật Ngữ Quan Trọng

| Thuật Ngữ | Ý Nghĩa |
|-----------|---------|
| **Control Plane** | Bộ não của cluster (master components) |
| **Worker Node** | Server chạy application Pods |
| **API Server** | Cổng vào duy nhất, REST API |
| **etcd** | Distributed key-value database |
| **Scheduler** | Component phân công Pods to Nodes |
| **Controller** | Component reconcile desired vs actual |
| **kubelet** | Node agent, manage Pods |
| **kube-proxy** | Network proxy, implement Services |
| **CRI** | Container Runtime Interface |
| **Reconciliation** | Process make actual = desired |

---

## ❓ FAQs

**Q: Có thể có nhiều Control Plane nodes không?**
```
A: Có! (High Availability setup)
- Multiple API Server instances (load balanced)
- Multiple Controller Manager (leader election)
- Multiple Scheduler (leader election)
- etcd cluster (3 or 5 nodes)

→ Survive Control Plane node failures!
```

**Q: Worker Node có thể là Control Plane không?**
```
A: Technically yes, nhưng NOT recommended production

Single-node cluster:
✓ Learning/development (minikube, kind)
✗ Production

Separation Control Plane + Workers:
✓ Better resource allocation
✓ Better security
✓ Better scaling
```

**Q: Tại sao không let components talk to etcd directly?**
```
A: Security và consistency!

If direct access:
❌ No validation
❌ No authorization  
❌ Inconsistent updates
❌ Hard to audit

Via API Server:
✓ Centralized validation
✓ Fine-grained RBAC
✓ Audit logging
✓ Consistent interface
```

**Q: iptables vs IPVS - nên dùng gì?**
```
A: Depends on scale

iptables (default):
✓ Stable, mature
✓ Good < 1000 services
✗ Performance degrades với nhiều services

IPVS:
✓ Better performance (O(1))
✓ More load balancing algorithms
✓ Good for large scale (1000+ services)
✗ Requires kernel modules
```

**Q: Có thể thay đổi Scheduler logic không?**
```
A: Có! Multiple options:

1. Custom Scheduler
   - Viết scheduler riêng
   - Deploy alongside default
   - Specify trong Pod: schedulerName: custom

2. Scheduler Extenders
   - Extend default scheduler
   - Add custom filter/score logic

3. Scheduler Framework
   - Plugin architecture (K8s 1.15+)
   - Write plugins for custom logic
```

---

## 🔧 Setup Next Steps

**Bạn đã hiểu architecture!**

Next step: Hands-on với K8s

**Phần 3: Core Concepts** sẽ bao gồm:
```
├── Setup local cluster (minikube/kind)
├── First kubectl commands
├── Create Pods, Deployments
├── Understand Namespaces, Labels
└── Hands-on practice!
```

---

## 📖 Resources Bổ Sung

### Documentation
- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)

### Deep Dives
- [How Kubernetes Scheduler Works](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [etcd Documentation](https://etcd.io/docs/)

### Videos
- "Kubernetes Deconstructed" - CoreOS
- "A Deep Dive Into Kubernetes Internals" - KubeCon talks

---

## 🚀 Tiếp Theo

**Completed:** Architecture fundamentals ✅

**Next:** [Phần 3: Core Concepts →](../03-core-concepts/README.md)

Bắt đầu hands-on:
- Setup local cluster
- Tạo first Pods
- kubectl commands
- Namespaces và Labels

Let's get practical! 🎯

---

[⬅️ Phần 1: Introduction](../01-introduction/README.md) | [🏠 Mục Lục Chính](../README.md) | [➡️ Phần 3: Core Concepts](../03-core-concepts/README.md)
