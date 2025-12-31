# 2.3. Worker Nodes - Nơi Chạy Applications

> Deep dive vào Worker Nodes - nơi workloads thực sự chạy

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **chi tiết các components** trên Worker Node
- ✅ Biết **cách kubelet quản lý Pods**
- ✅ Hiểu **networking trong Node** (kube-proxy)
- ✅ Troubleshoot được **Node-level issues**

---

## 👷 Worker Node Components

### Tổng Quan

```
WORKER NODE = NƠI APPLICATIONS CHẠY

┌────────────────────────────────────────────────┐
│            WORKER NODE                         │
├────────────────────────────────────────────────┤
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │  1. kubelet                              │ │
│  │     • Node agent                         │ │
│  │     • Manage Pods lifecycle              │ │
│  │     • Report to API Server               │ │
│  └─────────────┬────────────────────────────┘ │
│                │                              │
│  ┌─────────────┴────────────────────────────┐ │
│  │  2. kube-proxy                           │ │
│  │     • Network proxy                      │ │
│  │     • Service load balancing             │ │
│  │     • Maintain iptables rules            │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │  3. Container Runtime                    │ │
│  │     • Docker / containerd / CRI-O        │ │
│  │     • Pull images, run containers        │ │
│  │     • Manage container lifecycle         │ │
│  └─────────────┬────────────────────────────┘ │
│                │                              │
│  ┌─────────────┴────────────────────────────┐ │
│  │  PODS (Running Applications)             │ │
│  │  ┌──────┐  ┌──────┐  ┌──────┐           │ │
│  │  │ Pod1 │  │ Pod2 │  │ Pod3 │           │ │
│  │  │┌────┐│  │┌────┐│  │┌────┐│           │ │
│  │  ││App ││  ││App ││  ││App ││           │ │
│  │  │└────┘│  │└────┘│  │└────┘│           │ │
│  │  └──────┘  └──────┘  └──────┘           │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │  Operating System (Linux)                │ │
│  │  • Kernel                                │ │
│  │  • cgroups (resource limits)             │ │
│  │  • namespaces (isolation)                │ │
│  └──────────────────────────────────────────┘ │
│                                                │
└────────────────────────────────────────────────┘
```

---

## 1️⃣ kubelet - Node Agent

### Vai Trò: Supervisor Của Node

**kubelet = Foreman/Supervisor tại construction site**

```
kubelet responsibilities:
├── Watch API Server for Pods assigned to this Node
├── Ensure containers are running in Pods
├── Mount volumes
├── Run health checks (liveness, readiness)
├── Report Pod & Node status to API Server
└── Execute lifecycle hooks
```

### Kubelet Workflow

**Complete lifecycle management:**

```
┌─────────────────────────────────────────────────────────┐
│  KUBELET POD LIFECYCLE MANAGEMENT                       │
└─────────────────────────────────────────────────────────┘

1. WATCH API SERVER
   ↓
   "New Pod assigned to me!"
   Pod: webapp-abc123
   Image: nginx:latest
   ↓

2. CHECK POD STATUS
   ↓
   Pod not exist locally? → Need to create
   ↓

3. PULL IMAGE (via Container Runtime)
   ↓
   kubelet → Container Runtime: "Pull nginx:latest"
   Container Runtime → Docker Hub: Download image
   ↓
   [====== Downloading ======] 100%
   Image stored: /var/lib/containerd/...
   ↓

4. CREATE CONTAINER(S)
   ↓
   kubelet → Container Runtime: "Create container"
   Parameters:
   ├── Image: nginx:latest
   ├── Command: [nginx, -g, daemon off;]
   ├── Env vars: DB_HOST=postgres
   ├── Volumes: /data → emptyDir
   ├── Resources: CPU 500m, RAM 256Mi
   └── Network: Pod network
   ↓

5. START CONTAINER
   ↓
   Container Runtime: Start container
   Container ID: abc123def456
   Container state: Running
   ↓

6. MONITOR HEALTH
   ↓
   Every 10s:
   ├── Liveness probe: GET http://localhost/health
   │   Response 200 OK → Healthy ✓
   ├── Readiness probe: GET http://localhost/ready
   │   Response 200 OK → Ready ✓
   └── Container state: Running
   ↓

7. REPORT STATUS to API Server
   ↓
   kubelet → API Server:
   {
     "podIP": "10.244.1.5",
     "phase": "Running",
     "conditions": [
       {"type": "Ready", "status": "True"},
       {"type": "ContainersReady", "status": "True"}
     ],
     "containerStatuses": [
       {
         "name": "nginx",
         "state": {"running": {"startedAt": "..."}},
         "ready": true
       }
     ]
   }
   ↓

8. CONTINUOUS MONITORING
   ↓
   Every iteration:
   ├── Check container still running
   ├── Run health probes
   ├── Monitor resource usage
   ├── Report status
   └── React to changes (Pod deleted? Stop containers!)
```

### Kubelet Configuration

**Key kubelet parameters:**

```bash
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# API Server connection
clusterDomain: cluster.local
clusterDNS:
  - 10.96.0.10

# Pod management
podCIDR: 10.244.1.0/24
maxPods: 110

# Resource management
cpuManagerPolicy: none
memoryManagerPolicy: None

# Health checks
nodeStatusUpdateFrequency: 10s
nodeStatusReportFrequency: 5m

# Eviction thresholds
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"

# Image management
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
```

### Kubelet Interaction với Container Runtime

**CRI (Container Runtime Interface):**

```
kubelet ←--CRI API--→ Container Runtime
                      ├── Docker (via dockershim - deprecated)
                      ├── containerd ✓ (recommended)
                      ├── CRI-O ✓
                      └── Others...

CRI Operations:
├── RunPodSandbox (create Pod network namespace)
├── CreateContainer (create container in Pod)
├── StartContainer (start container)
├── StopContainer (stop container)
├── RemoveContainer (cleanup)
└── ListContainers (list running containers)
```

**Example: Create Pod via CRI**

```
1. kubelet → CRI: RunPodSandbox
   Create Pod network namespace
   Assign Pod IP: 10.244.1.5
   ↓

2. kubelet → CRI: CreateContainer
   Container spec: {
     image: "nginx:latest",
     command: ["nginx"],
     env: [...],
     mounts: [...]
   }
   ↓

3. CRI → containerd: Create container
   containerd creates container
   Returns containerID: abc123
   ↓

4. kubelet → CRI: StartContainer(abc123)
   CRI → containerd: Start
   Container running!
   ↓

5. kubelet monitors container
   Continuously checks health
```

---

## 2️⃣ kube-proxy - Network Proxy

### Vai Trò: Network Traffic Manager

**kube-proxy = Traffic cop/Router trong Node**

```
kube-proxy làm gì:
├── Maintain network rules trên Node
├── Enable Service abstraction
├── Load balance traffic to Pods
├── Implement ClusterIP, NodePort, LoadBalancer
└── Handle Pod-to-Pod communication
```

### Service Implementation

**Cách kube-proxy implement Services:**

```yaml
# Service definition
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

**kube-proxy tạo rules:**

**Mode 1: iptables (default, phổ biến nhất)**

```bash
# kube-proxy tạo iptables rules

# Service ClusterIP: 10.96.100.50
# Backend Pods:
# - Pod1: 10.244.1.5:8080
# - Pod2: 10.244.2.8:8080
# - Pod3: 10.244.1.9:8080

# iptables rules (simplified):
iptables -A KUBE-SERVICES \
  -d 10.96.100.50/32 -p tcp --dport 80 \
  -j KUBE-SVC-WEBAPP

# Load balance to 3 Pods
iptables -A KUBE-SVC-WEBAPP \
  -m statistic --mode random --probability 0.33 \
  -j KUBE-SEP-POD1

iptables -A KUBE-SVC-WEBAPP \
  -m statistic --mode random --probability 0.50 \
  -j KUBE-SEP-POD2

iptables -A KUBE-SVC-WEBAPP \
  -j KUBE-SEP-POD3

# DNAT to actual Pod IPs
iptables -A KUBE-SEP-POD1 -j DNAT --to-destination 10.244.1.5:8080
iptables -A KUBE-SEP-POD2 -j DNAT --to-destination 10.244.2.8:8080
iptables -A KUBE-SEP-POD3 -j DNAT --to-destination 10.244.1.9:8080
```

**Traffic flow:**

```
Application calls Service:
curl http://webapp:80
    ↓
DNS resolves: webapp → 10.96.100.50 (ClusterIP)
    ↓
Kernel hits iptables rules:
DNAT to one of backend Pods (random)
    ↓
Options (33% / 33% / 34%):
├── 10.244.1.5:8080 (Pod1)
├── 10.244.2.8:8080 (Pod2)
└── 10.244.1.9:8080 (Pod3)
    ↓
Traffic reaches Pod
    ↓
Pod processes request
    ↓
Response back to caller
```

**Mode 2: IPVS (IP Virtual Server - better performance)**

```bash
# kube-proxy creates IPVS virtual server

# Install ipvsadm to view
ipvsadm -Ln

# Output:
IP Virtual Server version 1.2.1
TCP  10.96.100.50:80 rr  (round-robin)
  -> 10.244.1.5:8080      Masq    1      0          0
  -> 10.244.2.8:8080      Masq    1      0          0
  -> 10.244.1.9:8080      Masq    1      0          0

# Load balancing algorithms:
# - rr (round-robin)
# - lc (least connection)
# - dh (destination hashing)
# - sh (source hashing)
```

**IPVS advantages:**
```
vs iptables:
✓ Better performance (O(1) vs O(n))
✓ More load balancing algorithms
✓ Better for large clusters (1000+ services)
✗ Requires kernel modules
```

### NodePort Implementation

**Example:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # Exposed on all Nodes
```

**kube-proxy rules:**

```bash
# Listen on NodePort 30080 on ALL Node IPs
# Node1 IP: 192.168.1.10
# Node2 IP: 192.168.1.11
# Node3 IP: 192.168.1.12

# iptables rules:
iptables -A KUBE-NODEPORTS \
  -p tcp --dport 30080 \
  -j KUBE-SVC-WEBAPP

# Then same load balancing as ClusterIP
```

**Traffic flow:**

```
External user:
curl http://192.168.1.10:30080
    ↓
Hits Node1 on port 30080
    ↓
iptables DNAT to Pod IP
    ↓
Pod on ANY node (could be Node2!)
    ↓
Response back through Node1
```

---

## 3️⃣ Container Runtime

### Vai Trò: Chạy Containers

**Container Runtime = Engine chạy containers**

```
Container Runtime options:
├── Docker (legacy, deprecated in K8s 1.24+)
├── containerd ✓ (recommended, default in many distros)
├── CRI-O ✓ (lightweight, OCI-compliant)
└── Others (kata, gVisor for security)
```

### containerd Architecture

```
┌─────────────────────────────────────────────┐
│              CONTAINERD                     │
├─────────────────────────────────────────────┤
│                                             │
│  ┌──────────────────────────────────────┐  │
│  │  CRI Plugin (gRPC)                   │  │
│  │  • Implements CRI                    │  │
│  │  • Interface for kubelet             │  │
│  └────────────┬─────────────────────────┘  │
│               ↓                             │
│  ┌──────────────────────────────────────┐  │
│  │  containerd Core                     │  │
│  │  • Image management                  │  │
│  │  • Container lifecycle               │  │
│  │  • Snapshots                         │  │
│  └────────────┬─────────────────────────┘  │
│               ↓                             │
│  ┌──────────────────────────────────────┐  │
│  │  runc (OCI Runtime)                  │  │
│  │  • Low-level container runtime       │  │
│  │  • Create & run containers           │  │
│  │  • cgroups & namespaces              │  │
│  └──────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────┐
│         LINUX KERNEL                        │
│  • Namespaces (isolation)                   │
│  • cgroups (resource limits)                │
│  • Capabilities                             │
└─────────────────────────────────────────────┘
```

### Container Lifecycle

**From image to running container:**

```
1. PULL IMAGE
   containerd → Registry (Docker Hub):
   "Download nginx:latest"
   ↓
   Layers downloaded:
   ├── Layer 1: base OS
   ├── Layer 2: nginx binary
   ├── Layer 3: config files
   └── Layer 4: entry point
   ↓
   Stored in: /var/lib/containerd/...
   ↓

2. CREATE CONTAINER
   containerd creates container config:
   {
     "image": "nginx:latest",
     "rootfs": "/var/lib/containerd/...",
     "env": ["PATH=/usr/bin"],
     "cmd": ["nginx", "-g", "daemon off;"],
     "mounts": [...],
     "namespaces": [
       {"type": "pid"},
       {"type": "network"},
       {"type": "mount"},
       {"type": "uts"}
     ],
     "cgroups": {
       "cpu": {"quota": 50000},  # 500m CPU
       "memory": {"limit": 268435456}  # 256Mi
     }
   }
   ↓

3. START CONTAINER
   runc creates:
   ├── New namespaces (isolation)
   ├── cgroups (resource limits)
   ├── Root filesystem (from image layers)
   └── Network namespace
   ↓
   Execute entrypoint: nginx -g "daemon off;"
   ↓
   Container PID: 12345
   Container running!
   ↓

4. MONITOR
   containerd monitors:
   ├── Process state (running/stopped)
   ├── Resource usage (CPU, RAM)
   └── Exit code if stopped
```

---

## 🔄 Node Components Interaction

### Complete Flow: Pod Startup

```
┌────────────────────────────────────────────────────┐
│  POD STARTUP ON WORKER NODE                       │
└────────────────────────────────────────────────────┘

1. SCHEDULER assigns Pod to Node
   API Server: Pod.spec.nodeName = "worker-node-1"
   ↓

2. KUBELET on worker-node-1 watches API Server
   "New Pod assigned to me!"
   Pod: webapp-abc123
   ↓

3. KUBELET → CONTAINER RUNTIME (via CRI)
   "Create Pod sandbox"
   ↓
   Container Runtime:
   ├── Create network namespace
   ├── Setup Pod network (assign IP)
   ├── Pod IP: 10.244.1.5
   └── Return sandbox ID
   ↓

4. KUBELET → CONTAINER RUNTIME
   "Pull image: nginx:latest"
   ↓
   Container Runtime downloads image
   [====== Download 100% ======]
   ↓

5. KUBELET → CONTAINER RUNTIME
   "Create container in sandbox"
   ↓
   Container Runtime:
   ├── Create container from image
   ├── Mount volumes
   ├── Set environment variables
   ├── Configure resources (CPU, RAM limits)
   └── Return container ID
   ↓

6. KUBELET → CONTAINER RUNTIME
   "Start container"
   ↓
   Container Runtime starts container
   Container state: Running
   ↓

7. KUBELET configures KUBE-PROXY
   (via updating Endpoints)
   ↓

8. KUBE-PROXY updates iptables/IPVS
   Add Pod IP to Service backend pool
   Service: webapp
   Backend Pods:
   ├── 10.244.1.5:8080 ← NEW!
   ├── 10.244.2.8:8080
   └── 10.244.1.9:8080
   ↓

9. KUBELET runs health checks
   Liveness: GET http://10.244.1.5:8080/health
   Response: 200 OK ✓
   ↓
   Readiness: GET http://10.244.1.5:8080/ready  
   Response: 200 OK ✓
   ↓

10. KUBELET → API SERVER
    "Pod ready!"
    Pod status:
    ├── phase: Running
    ├── conditions: Ready=True
    ├── podIP: 10.244.1.5
    └── containerStatuses: [...ready...]
    ↓

11. Pod fully operational! ✅
    Receiving traffic from Service
```

---

## 🔍 Troubleshooting Worker Nodes

### Common Issues & Solutions

**Issue 1: Node NotReady**

```bash
# Check Node status
kubectl get nodes
NAME           STATUS     ROLES    AGE   VERSION
worker-node-1  NotReady   <none>   5d    v1.28.0

# Describe Node
kubectl describe node worker-node-1
# Look at conditions and events

# Common causes:
# 1. kubelet not running
ssh worker-node-1
sudo systemctl status kubelet
sudo systemctl restart kubelet

# 2. Network issue
ping 8.8.8.8  # Check internet
# Check CNI plugin

# 3. Resource exhaustion
df -h  # Disk full?
free -m  # Memory pressure?
```

**Issue 2: Pod Stuck in Pending**

```bash
# Describe Pod
kubectl describe pod webapp-abc123

# Check events:
# "0/3 nodes are available: insufficient cpu"
# → Need more resources or scale cluster

# "0/3 nodes are available: node(s) had taints"
# → Check node taints
kubectl describe node worker-node-1 | grep Taints
# Add tolerations to Pod if needed
```

**Issue 3: Container Keeps Crashing**

```bash
# Check Pod logs
kubectl logs webapp-abc123
kubectl logs webapp-abc123 --previous  # Previous crash

# Check container status
kubectl describe pod webapp-abc123
# Look at: LastState, Reason (OOMKilled? CrashLoopBackOff?)

# Common fixes:
# - Increase memory limits (if OOMKilled)
# - Fix application errors (check logs)
# - Adjust liveness probe (if too aggressive)
```

**Issue 4: Network Problems**

```bash
# Check Pod can reach Service
kubectl exec webapp-abc123 -- curl http://backend-service

# Check DNS
kubectl exec webapp-abc123 -- nslookup backend-service

# Check kube-proxy
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system kube-proxy-xyz

# Check iptables rules
sudo iptables-save | grep KUBE
```

### Debugging Commands

```bash
# Node level
kubectl get nodes
kubectl describe node <node-name>
kubectl top node <node-name>

# Pod level
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/bash

# Container runtime
# (SSH to node first)
sudo crictl ps  # List containers
sudo crictl logs <container-id>
sudo crictl inspect <container-id>

# kubelet
sudo systemctl status kubelet
sudo journalctl -u kubelet -f

# Network
kubectl get svc
kubectl get endpoints
sudo iptables-save | grep <service-name>
```

---

## 🎓 Kiểm Tra Hiểu Biết

**1. kubelet làm gì khi Pod assigned to Node?**
<details>
<summary>Xem đáp án</summary>

1. Watch API Server, detect new Pod
2. Pull container image (via Container Runtime)
3. Create Pod sandbox (network namespace)
4. Create container(s) from image
5. Start container(s)
6. Run health checks (liveness, readiness)
7. Report Pod status to API Server
8. Continuously monitor và restart nếu cần
</details>

**2. kube-proxy implement Service như thế nào?**
<details>
<summary>Xem đáp án</summary>

**iptables mode (default):**
- Tạo iptables rules
- DNAT Service ClusterIP → Pod IPs
- Random load balancing giữa Pods
- Update rules khi Pods thay đổi

**IPVS mode:**
- Tạo IPVS virtual servers
- Better performance (O(1) lookup)
- More load balancing algorithms
- Better cho large scale
</details>

**3. Container Runtime nào recommended?**
<details>
<summary>Xem đáp án</summary>

**containerd** ✓ (recommended)
- Lightweight
- Industry standard
- Default trong nhiều distros
- Good performance

**CRI-O** ✓ (alternative)
- Lightweight
- OCI-compliant
- Designed specifically for K8s

**Docker** ❌ (deprecated)
- Removed in K8s 1.24+
- Too heavy for K8s
- Use containerd instead
</details>

---

## 🎯 Key Takeaways

1. **kubelet = Node Supervisor**
   - Manage Pod lifecycle
   - Report to API Server
   - Execute health checks

2. **kube-proxy = Network Manager**
   - Implement Services
   - Load balancing
   - iptables or IPVS

3. **Container Runtime = Container Engine**
   - containerd recommended
   - Pull images, run containers
   - CRI interface với kubelet

4. **Isolation via Linux Kernel**
   - Namespaces (PID, Network, Mount)
   - cgroups (CPU, Memory limits)
   - Secure container isolation

5. **Troubleshooting Workflow**
   - Check Node status
   - Check Pod describe/logs
   - Check kubelet logs
   - Check network connectivity

---

## 🚀 Tiếp Theo

Hoàn thành Phần 2 - Architecture!

**Next:** [Phần 3: Core Concepts →](../03-core-concepts/README.md)

Ở phần tiếp theo, chúng ta sẽ học các concepts cốt lõi: Pod, Namespace, Labels, Selectors.

---

[⬅️ 2.2. Control Plane](./02-control-plane.md) | [🏠 Mục Lục Chính](../README.md) | [📂 Phần 2: Architecture](./README.md) | [➡️ Phần 3: Core Concepts](../03-core-concepts/README.md)
