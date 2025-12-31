# 3.1. Cluster và Nodes

> Hiểu building blocks cơ bản nhất của Kubernetes

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **Cluster và Node** là gì
- ✅ Phân biệt **Master Node vs Worker Node**
- ✅ Biết **Node lifecycle và management**
- ✅ **Setup local cluster** để practice
- ✅ Sử dụng **kubectl commands** cơ bản

---

## 🏢 Cluster Là Gì?

### Định Nghĩa

**Kubernetes Cluster** = Tập hợp servers (Nodes) làm việc cùng nhau như một hệ thống thống nhất.

### Giải Thích Bằng Ví Dụ

**Cluster giống như một tòa nhà văn phòng:**

```
🏢 TÒA NHÀ VĂN PHÒNG (Cluster)
├── 🎯 Tầng Điều Hành (Master Nodes / Control Plane)
│   ├── CEO Office (API Server)
│   ├── CFO Office (etcd)
│   ├── HR Office (Scheduler)
│   └── Operations Office (Controllers)
│
└── 👷 Các Tầng Làm Việc (Worker Nodes)
    ├── Tầng 2: Developer teams
    ├── Tầng 3: QA teams
    ├── Tầng 4: DevOps teams
    └── Tầng 5+: More teams...
```

### Cluster Components

```
┌─────────────────────────────────────────────┐
│          KUBERNETES CLUSTER                 │
├─────────────────────────────────────────────┤
│                                             │
│  Master Nodes (Control Plane):             │
│  ├─ master-1 (HA setup)                    │
│  ├─ master-2 (HA setup)                    │
│  └─ master-3 (HA setup)                    │
│                                             │
│  Worker Nodes (Application layer):         │
│  ├─ worker-1                                │
│  ├─ worker-2                                │
│  ├─ worker-3                                │
│  ├─ worker-4                                │
│  └─ ... more workers ...                   │
│                                             │
└─────────────────────────────────────────────┘
```

---

## 🖥️ Node Là Gì?

### Định Nghĩa

**Node** = Một server (physical hoặc virtual) trong Kubernetes Cluster.

### Loại Nodes

**1. Master Node (Control Plane Node)**
```
Vai trò: Điều khiển và quản lý cluster
Chạy: Control Plane components
  ├── kube-apiserver
  ├── etcd
  ├── kube-scheduler
  └── kube-controller-manager

Số lượng: 
├── 1 node: OK cho dev/test
├── 3 nodes: Recommended cho production (HA)
└── 5 nodes: Large production clusters

Tài nguyên: Ít hơn Workers (không chạy apps)
```

**2. Worker Node**
```
Vai trò: Chạy application workloads
Chạy: Node components + Application Pods
  ├── kubelet
  ├── kube-proxy
  ├── Container Runtime
  └── Application Pods

Số lượng: Nhiều (10, 100, 1000+ nodes)
Tài nguyên: Nhiều CPU/RAM cho applications
```

---

## 🔧 Node Components

### Worker Node Anatomy

```
┌────────────────────────────────────────────────┐
│           WORKER NODE (worker-1)               │
├────────────────────────────────────────────────┤
│                                                │
│  System Info:                                  │
│  ├─ Hostname: worker-1                         │
│  ├─ IP: 192.168.1.101                          │
│  ├─ OS: Ubuntu 22.04                           │
│  ├─ CPU: 8 cores                               │
│  ├─ RAM: 32 GB                                 │
│  └─ Disk: 500 GB SSD                           │
│                                                │
│  K8s Components:                               │
│  ┌──────────────────────────────────────────┐ │
│  │  kubelet (v1.28.0)                       │ │
│  │  • PID: 1234                             │ │
│  │  • Managing 15 Pods                      │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │  kube-proxy (v1.28.0)                    │ │
│  │  • Mode: iptables                        │ │
│  │  • Managing 50 Services                  │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │  Container Runtime: containerd           │ │
│  │  • Version: 1.7.0                        │ │
│  │  • Running 25 containers                 │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  Pods:                                         │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐            │
│  │Pod1 │ │Pod2 │ │Pod3 │ │... │            │
│  └─────┘ └─────┘ └─────┘ └─────┘            │
│                                                │
└────────────────────────────────────────────────┘
```

---

## 🚀 Setup Local Cluster

### Option 1: Minikube (Recommended cho beginners)

**Install Minikube:**

```bash
# macOS
brew install minikube

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Windows (with Chocolatey)
choco install minikube

# Verify installation
minikube version
```

**Start Cluster:**

```bash
# Start với default settings
minikube start

# Output:
# 😄  minikube v1.32.0 on Darwin 13.5
# ✨  Using the docker driver based on existing profile
# 👍  Starting control plane node minikube in cluster minikube
# 🚜  Pulling base image ...
# 🔥  Creating docker container (CPUs=2, Memory=4096MB) ...
# 🐳  Preparing Kubernetes v1.28.0 on Docker 24.0.6 ...
# 🔗  Configuring bridge CNI (Container Networking Interface) ...
# 🔎  Verifying Kubernetes components...
# 🌟  Enabled addons: storage-provisioner, default-storageclass
# 🏄  Done! kubectl is now configured to use "minikube" cluster

# Check cluster status
minikube status

# Output:
# minikube
# type: Control Plane
# host: Running
# kubelet: Running
# apiserver: Running
# kubeconfig: Configured

# Get cluster info
kubectl cluster-info

# Output:
# Kubernetes control plane is running at https://127.0.0.1:32768
# CoreDNS is running at https://127.0.0.1:32768/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

**Minikube useful commands:**

```bash
# Stop cluster (giữ data)
minikube stop

# Delete cluster (xóa hết)
minikube delete

# SSH vào node
minikube ssh

# Open dashboard
minikube dashboard

# Check resource usage
minikube addons enable metrics-server
kubectl top nodes
```

---

### Option 2: kind (Kubernetes in Docker)

**Install kind:**

```bash
# macOS/Linux
brew install kind

# Linux (binary)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Verify
kind version
```

**Create Cluster:**

```bash
# Simple single-node cluster
kind create cluster

# Multi-node cluster
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF

# List clusters
kind get clusters

# Get cluster info
kubectl cluster-info --context kind-kind

# Delete cluster
kind delete cluster
```

---

### Option 3: Docker Desktop (Easiest cho Windows/Mac)

```
1. Install Docker Desktop
2. Settings → Kubernetes → Enable Kubernetes
3. Apply & Restart
4. kubectl cluster-info (should work!)
```

---

## 📊 Working với Nodes

### View Nodes

```bash
# List all nodes
kubectl get nodes

# Output:
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   5m    v1.28.0

# Detailed node info
kubectl get nodes -o wide

# Output:
# NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
# minikube   Ready    control-plane   5m    v1.28.0   192.168.49.2   <none>        Ubuntu 22.04.3 LTS   5.15.0-91-generic   containerd://1.7.10
```

### Describe Node

```bash
# Get detailed information về một node
kubectl describe node minikube

# Output (sample):
Name:               minikube
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=minikube
                    kubernetes.io/os=linux
                    minikube.k8s.io/commit=5883c09216182566a63dff4c326a6fc9ed2982ff
                    minikube.k8s.io/name=minikube
                    minikube.k8s.io/primary=true
                    minikube.k8s.io/updated_at=2024_01_01T10_00_00_0700
                    minikube.k8s.io/version=v1.32.0
                    node-role.kubernetes.io/control-plane=
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: unix:///var/run/containerd/containerd.sock
                    node.alpha.kubernetes.io/ttl: 0
CreationTimestamp:  Mon, 01 Jan 2024 10:00:00 +0700
Taints:             <none>

Conditions:
  Type             Status  LastHeartbeatTime                 Reason                Message
  ----             ------  -----------------                 ------                -------
  MemoryPressure   False   Mon, 01 Jan 2024 10:05:00 +0700  KubeletHasSufficientMemory
  DiskPressure     False   Mon, 01 Jan 2024 10:05:00 +0700  KubeletHasNoDiskPressure
  PIDPressure      False   Mon, 01 Jan 2024 10:05:00 +0700  KubeletHasSufficientPID
  Ready            True    Mon, 01 Jan 2024 10:05:00 +0700  KubeletReady

Capacity:
  cpu:                8
  ephemeral-storage:  488236136Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             32837460Ki
  pods:               110

Allocatable:
  cpu:                8
  ephemeral-storage:  449893134758
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             32735060Ki
  pods:               110

System Info:
  Machine ID:                 abcd1234efgh5678ijkl9012mnop3456
  System UUID:                12345678-1234-1234-1234-123456789012
  Boot ID:                    abcd1234-ab12-cd34-ef56-123456789abc
  Kernel Version:             5.15.0-91-generic
  OS Image:                   Ubuntu 22.04.3 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.7.10
  Kubelet Version:            v1.28.0
  Kube-Proxy Version:         v1.28.0

Non-terminated Pods:          (8 in total)
  Namespace                   Name                                CPU Requests  Memory Requests
  ---------                   ----                                ------------  ---------------
  kube-system                 coredns-5dd5756b68-abc12            100m (1%)     70Mi (0%)
  kube-system                 etcd-minikube                       100m (1%)     100Mi (0%)
  kube-system                 kube-apiserver-minikube             250m (3%)     0 (0%)
  ... more pods ...

Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                850m (10%)  0 (0%)
  memory             470Mi (1%)  370Mi (1%)
```

---

### Node Conditions

**Node Conditions explained:**

```yaml
Conditions:
  # 1. MemoryPressure
  # False = Có đủ memory
  # True = Memory sắp hết (kubelet sẽ evict Pods)
  MemoryPressure: False

  # 2. DiskPressure
  # False = Có đủ disk space
  # True = Disk gần đầy
  DiskPressure: False

  # 3. PIDPressure
  # False = Có đủ process IDs
  # True = Too many processes
  PIDPressure: False

  # 4. Ready
  # True = Node healthy, có thể nhận Pods
  # False = Node có vấn đề
  # Unknown = kubelet không response (>40s)
  Ready: True
```

---

### Node Capacity vs Allocatable

```
Capacity: Tổng resources của Node
Allocatable: Resources available cho Pods
          (Capacity - System Reserved - Eviction Thresholds)

Example:
Capacity:
├── CPU: 8 cores
├── Memory: 32 GB
└── Pods: 110

Allocatable:
├── CPU: 7.8 cores (OS reserved 0.2)
├── Memory: 30 GB (OS + kubelet reserved 2 GB)
└── Pods: 110
```

---

## 🏷️ Node Labels

### Default Labels

```bash
# View node labels
kubectl get nodes --show-labels

# Common default labels:
kubernetes.io/arch=amd64
kubernetes.io/os=linux
kubernetes.io/hostname=worker-1
topology.kubernetes.io/zone=us-central1-a
topology.kubernetes.io/region=us-central1
node.kubernetes.io/instance-type=n1-standard-4
```

### Add Custom Labels

```bash
# Add label
kubectl label nodes worker-1 environment=production
kubectl label nodes worker-1 disktype=ssd

# Verify
kubectl get nodes --show-labels | grep worker-1

# Remove label
kubectl label nodes worker-1 disktype-
```

### Use Labels trong Pod Scheduling

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  # Schedule Pod chỉ trên nodes có label này
  nodeSelector:
    disktype: ssd
    environment: production
  containers:
  - name: nginx
    image: nginx
```

---

## 🚫 Node Taints & Tolerations

### Taints = "Đuổi" Pods khỏi Node

```bash
# Add taint (prevent Pods from scheduling)
kubectl taint nodes worker-1 dedicated=database:NoSchedule

# Taint effects:
# NoSchedule: Không schedule Pods mới
# PreferNoSchedule: Cố tránh, nhưng OK nếu cần
# NoExecute: Evict existing Pods + không schedule mới

# View taints
kubectl describe node worker-1 | grep Taints

# Remove taint
kubectl taint nodes worker-1 dedicated:NoSchedule-
```

### Tolerations = Pods "chịu" được Taint

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
spec:
  # Tolerate taint để schedule được lên node
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
  containers:
  - name: postgres
    image: postgres:14
```

**Use case:** Dedicate nodes cho specific workloads (databases, GPU jobs)

---

## 🎓 Kiểm Tra Hiểu Biết

**1. Cluster vs Node - khác nhau như thế nào?**
<details>
<summary>Xem đáp án</summary>

- **Cluster**: Tập hợp nhiều Nodes
- **Node**: Một server trong Cluster

Analogy:
- Cluster = Công ty
- Node = Nhân viên trong công ty
</details>

**2. Master Node vs Worker Node?**
<details>
<summary>Xem đáp án</summary>

**Master Node:**
- Vai trò: Điều khiển cluster
- Chạy: Control Plane components
- Số lượng: 1-5 (HA)

**Worker Node:**
- Vai trò: Chạy applications
- Chạy: kubelet, kube-proxy, Pods
- Số lượng: Nhiều (scalable)
</details>

**3. Node Conditions - Ready=False nghĩa là gì?**
<details>
<summary>Xem đáp án</summary>

Node có vấn đề, không thể nhận Pods mới:
- kubelet không healthy
- Disk/Memory pressure
- Network issues
- Container runtime problems

Cần troubleshoot ngay!
</details>

---

## 💪 Bài Tập Thực Hành

### Bài 1: Setup và Explore Cluster

```bash
# 1. Start minikube cluster
minikube start

# 2. Check cluster info
kubectl cluster-info

# 3. Get nodes
kubectl get nodes

# 4. Describe node
kubectl describe node minikube

# Questions:
# - How many nodes?
# - What K8s version?
# - How much CPU/RAM?
# - How many Pods can run?
```

<details>
<summary>Xem sample answers</summary>

```bash
# Nodes: 1 (single-node cluster)
# Version: v1.28.0
# CPU: 8 cores
# RAM: 32 GB
# Max Pods: 110
```
</details>

### Bài 2: Node Labels

```bash
# 1. Add labels
kubectl label nodes minikube environment=dev
kubectl label nodes minikube tier=frontend

# 2. Verify labels
kubectl get nodes --show-labels

# 3. Remove a label
kubectl label nodes minikube tier-

# 4. Verify again
kubectl get nodes --show-labels
```

### Bài 3: Explore System Pods

```bash
# 1. Get all Pods trong kube-system namespace
kubectl get pods -n kube-system

# 2. Describe một system Pod
kubectl describe pod -n kube-system coredns-<hash>

# 3. Check logs
kubectl logs -n kube-system <pod-name>

# Questions:
# - Các system Pods nào đang chạy?
# - Chúng chạy trên node nào?
# - Có healthy không?
```

---

## 🎯 Key Takeaways

1. **Cluster = Nhiều Nodes**
   - Master Nodes (Control Plane)
   - Worker Nodes (Applications)

2. **Node = Server**
   - Physical hoặc Virtual machine
   - Chạy K8s components + Pods

3. **Setup Local Cluster**
   - Minikube: Easiest cho beginners
   - kind: Fast, multi-node
   - Docker Desktop: Integrated

4. **Node Management**
   - Labels: Organize và select nodes
   - Taints: Control scheduling
   - Conditions: Health status

5. **kubectl Basics**
   - `get nodes`: List nodes
   - `describe node`: Details
   - `label nodes`: Add labels

---

## 🚀 Tiếp Theo

Cluster đã ready! Giờ học về Pods - đơn vị cơ bản nhất!

**Next:** [3.2. Pods →](./02-pods.md)

---

[⬅️ Phần 2: Architecture](../02-architecture/README.md) | [🏠 Mục Lục](../README.md) | [📂 Phần 3: Core Concepts](./README.md) | [➡️ 3.2. Pods](./02-pods.md)
