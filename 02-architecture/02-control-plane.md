# 2.2. Control Plane - Bộ Não Của Cluster

> Deep dive vào từng component của Control Plane

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **chi tiết từng component** của Control Plane
- ✅ Biết **vai trò cụ thể** của mỗi component
- ✅ Hiểu **cách các components tương tác** với nhau
- ✅ Troubleshoot được **issues liên quan Control Plane**

---

## 🧠 Control Plane Components

### Tổng Quan

```
CONTROL PLANE = BỘ NÃO CỦA CLUSTER

┌────────────────────────────────────────────────┐
│            CONTROL PLANE NODE                  │
├────────────────────────────────────────────────┤
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │  1. kube-apiserver                       │ │
│  │     • Cổng vào duy nhất                  │ │
│  │     • REST API                           │ │
│  │     • Authentication & Authorization     │ │
│  └─────────────┬────────────────────────────┘ │
│                │                              │
│  ┌─────────────┴────────────────────────────┐ │
│  │  2. etcd                                 │ │
│  │     • Distributed key-value database     │ │
│  │     • Store all cluster data             │ │
│  │     • Single source of truth             │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │  3. kube-scheduler                       │ │
│  │     • Watch for unassigned Pods          │ │
│  │     • Select best Node for Pod           │ │
│  │     • Consider resources, constraints    │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │  4. kube-controller-manager              │ │
│  │     • Run multiple controllers           │ │
│  │     • Watch & reconcile state            │ │
│  │     • Ensure desired = actual            │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │  5. cloud-controller-manager (optional)  │ │
│  │     • Cloud-specific control logic       │ │
│  │     • Manage cloud resources             │ │
│  └──────────────────────────────────────────┘ │
│                                                │
└────────────────────────────────────────────────┘
```

---

## 1️⃣ API Server (kube-apiserver)

### Vai Trò: Trung Tâm Giao Tiếp

**API Server = Receptionist/Switchboard Operator của cluster**

```
USER/SYSTEM
    ↓
 API Server  ← Điểm vào DUY NHẤT
    ↓
All other components
```

### Chức Năng Chính

**1. Frontend cho Cluster**
```
Mọi interaction với cluster đều qua API Server:
├── kubectl commands
├── Dashboard
├── CI/CD tools
├── Custom controllers
└── Other K8s components

→ KHÔNG AI có thể bypass API Server!
```

**2. Authentication & Authorization**
```
Mỗi request phải:
1. Authentication: "Bạn là ai?"
   ├── Client certificates
   ├── Bearer tokens
   ├── Service account tokens
   └── OpenID Connect (OIDC)

2. Authorization: "Bạn được làm gì?"
   ├── RBAC (Role-Based Access Control)
   ├── ABAC (Attribute-Based)
   └── Webhook mode

3. Admission Control: "Request có hợp lệ không?"
   ├── Validate resources
   ├── Mutate resources (add defaults)
   └── Custom admission webhooks
```

**3. Validation**
```
API Server validates:
├── Syntax correctness (YAML structure)
├── Required fields present
├── Field values valid (e.g., CPU format: "100m")
├── References exist (e.g., ConfigMap exists)
└── Quota limits not exceeded
```

**4. etcd Gateway**
```
API Server là component DUY NHẤT giao tiếp với etcd:

Write path:
User → API Server → Validate → etcd

Read path:
User → API Server → etcd → API Server → User

Other components:
Scheduler/Controllers → API Server → etcd
(KHÔNG truy cập etcd trực tiếp!)
```

### Ví Dụ: Create Pod Request

```yaml
# user.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

**Flow chi tiết:**

```
1. User runs:
   $ kubectl create -f user.yaml
                ↓
2. kubectl sends HTTP POST to API Server:
   POST /api/v1/namespaces/default/pods
   Body: Pod spec (JSON)
                ↓
3. API Server - Authentication:
   ✓ Verify client certificate
   ✓ Identity: user@example.com
                ↓
4. API Server - Authorization (RBAC):
   Question: "Can user@example.com create Pods in default namespace?"
   Check RBAC rules...
   ✓ Allowed!
                ↓
5. API Server - Admission Control:
   • Mutating admission: Add default values
     - Add imagePullPolicy: Always
     - Add restartPolicy: Always
   • Validating admission: Check constraints
     - Resource limits OK?
     - Security policies OK?
   ✓ Valid!
                ↓
6. API Server - Validation:
   ✓ YAML syntax correct
   ✓ Required fields present
   ✓ Image name valid
   ✓ Container name unique
                ↓
7. API Server writes to etcd:
   Key: /registry/pods/default/nginx
   Value: Pod spec + metadata
   Status: Pending (chưa có Node)
                ↓
8. API Server returns to kubectl:
   HTTP 201 Created
   Body: Pod object với metadata (UID, creationTimestamp)
                ↓
9. kubectl shows:
   pod/nginx created
```

### Commands để Monitor

```bash
# Check API Server logs
kubectl logs -n kube-system kube-apiserver-<node>

# API Server metrics
kubectl get --raw /metrics | grep apiserver

# Check API Server endpoints
kubectl get --raw /api/v1 | jq
kubectl get --raw /apis | jq

# API Server health
curl -k https://localhost:6443/healthz
curl -k https://localhost:6443/readyz
```

---

## 2️⃣ etcd

### Vai Trò: Database Của Cluster

**etcd = Kho lưu trữ hồ sơ của công ty**

```
etcd lưu TẤT CẢ:
├── Pods
├── Services
├── Deployments
├── Secrets
├── ConfigMaps
├── Nodes
└── Mọi K8s resources!
```

### Đặc Điểm

**1. Distributed Key-Value Store**
```
Key-Value Database:
Key: /registry/pods/default/nginx
Value: {
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": { ... },
  "spec": { ... },
  "status": { ... }
}

Distributed:
├── Run trên multiple nodes (usually 3 or 5)
├── Use Raft consensus algorithm
├── Survive node failures
└── Always consistent
```

**2. Consistency is Key**
```
Strong Consistency Guarantee:
├── Một write được ack → Guaranteed durable
├── Reads always return latest write
├── No stale data
└── Perfect for cluster state!

Vì sao quan trọng:
- Cluster state phải accurate 100%
- Pod đang chạy ở đâu?
- Service có những Endpoints nào?
- Wrong data = disaster!
```

**3. Watch Mechanism**
```
Components "watch" API Server for changes:
(NOT etcd directly! Only API Server talks to etcd)

Example - Scheduler watches API Server:
API Server: "New Pod created (status: Pending)"
     ↓
Scheduler: "Aha! Pod cần Node. Let me assign..."
          ↓
Scheduler → API Server: "Assign Pod to Node X"
          ↓
API Server → etcd: Save binding
          ↓
API Server → kubelet (watch): "Pod assigned to you!"
          ↓
kubelet on Node X: "Aha! Let me start this Pod!"

Key Point: Components NEVER access etcd directly!
All communication goes through API Server.
```

### Data Structure trong etcd

```bash
# View etcd structure (requires etcd access)
export ETCDCTL_API=3

# List all keys
etcdctl get / --prefix --keys-only

# Sample structure:
/registry/
├── pods/
│   ├── default/
│   │   ├── nginx
│   │   └── webapp-abc123
│   └── kube-system/
│       └── coredns-xyz789
├── services/
│   └── default/
│       └── kubernetes
├── deployments/
│   └── default/
│       └── webapp
├── secrets/
│   └── default/
│       └── my-secret
└── ...

# Get specific Pod
etcdctl get /registry/pods/default/nginx
```

### Backup etcd - CỰC KỲ QUAN TRỌNG!

**TẠI SAO:** etcd chết = mất TOÀN BỘ cluster state!

```bash
# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db

# Restore từ backup
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore

# Schedule regular backups (production must-have!)
# Cron job every 6 hours
```

---

## 3️⃣ Scheduler (kube-scheduler)

### Vai Trò: Phân Công Thông Minh

**Scheduler = HR Manager phân công nhân viên vào projects**

```
Scheduler's job:
1. Watch for Pods without Node assignment (status: Pending)
2. Find best Node for each Pod
3. Assign Pod → Node (write to API Server)
```

### Scheduling Algorithm

**Step 1: Filtering (Lọc)**
```
Question: "Nodes nào CÓ THỂ chạy Pod này?"

Check:
├── Node có đủ CPU/RAM không?
│   Pod requests: 1 CPU, 2Gi RAM
│   Node available: 0.5 CPU, 1Gi RAM
│   → FILTERED OUT ❌
│
├── Node có label matching không? (nodeSelector)
│   Pod: nodeSelector: disktype=ssd
│   Node: disktype=hdd
│   → FILTERED OUT ❌
│
├── Node có taints Pod không tolerate?
│   Node: taint=dedicated:NoSchedule
│   Pod: No tolerations
│   → FILTERED OUT ❌
│
└── Pod affinity/anti-affinity satisfied?
    → Check rules...

Result: List of feasible nodes
```

**Step 2: Scoring (Chấm điểm)**
```
Question: "Node nào TỐT NHẤT trong các nodes feasible?"

Scoring criteria:
├── LeastRequestedPriority (nhiều resources available = điểm cao)
│   Node 1: 20% CPU used → Score: 80
│   Node 2: 60% CPU used → Score: 40
│
├── BalancedResourceAllocation (balanced CPU & RAM = tốt)
│   Node 1: CPU 30%, RAM 80% → Unbalanced → Score: 50
│   Node 2: CPU 60%, RAM 65% → Balanced → Score: 90
│
├── NodeAffinityPriority (match affinity preferences)
│   Preferred affinity matched → +10 points
│
└── ImageLocalityPriority (image already on node = faster)
    Image present → +5 points

Total scores:
Node 1: 145 points
Node 2: 195 points ← WINNER!
Node 3: 120 points
```

**Step 3: Binding**
```
Scheduler → API Server:
"Assign Pod X to Node 2"

API Server → etcd:
Update Pod:
  spec.nodeName: node-2
  status: Pending (still Pending!)
  
Note: Pod remains "Pending" until kubelet starts it.
Only when container running → status: Running
```

### Ví Dụ Thực Tế

**Scenario: Schedule 3 Pods**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 500m
            memory: 256Mi
```

**Cluster state:**
```
Nodes:
├── node-1: 2 CPU, 4Gi RAM (available: 1.5 CPU, 3Gi)
├── node-2: 4 CPU, 8Gi RAM (available: 3 CPU, 7Gi)
└── node-3: 2 CPU, 4Gi RAM (available: 0.3 CPU, 1Gi)
```

**Scheduler decisions:**
```
Pod 1:
├── Filter: node-1 ✓, node-2 ✓, node-3 ✗ (not enough CPU)
├── Score: node-2 (195) > node-1 (145)
└── Assign: Pod 1 → node-2
   (node-2 available: 2.5 CPU, 6.75Gi RAM)

Pod 2:
├── Filter: node-1 ✓, node-2 ✓, node-3 ✗
├── Score: node-2 (175) > node-1 (145)
└── Assign: Pod 2 → node-2
   (node-2 available: 2 CPU, 6.5Gi RAM)

Pod 3:
├── Filter: node-1 ✓, node-2 ✓, node-3 ✗
├── Score: node-2 (155) > node-1 (145)
└── Assign: Pod 3 → node-2
   (node-2 available: 1.5 CPU, 6.25Gi RAM)

Final distribution:
├── node-1: 0 Pods (still available)
├── node-2: 3 Pods (all Pods fit!)
└── node-3: 0 Pods (insufficient resources)

Note: Scheduler prefers node-2 vì có nhiều resources nhất
```

---

## 4️⃣ Controller Manager (kube-controller-manager)

### Vai Trò: Đảm Bảo Desired State

**Controller Manager = Operations Manager giám sát mọi thứ**

```
Controller Pattern:
1. Watch actual state
2. Compare với desired state
3. If different → Take action
4. Repeat (forever!)

→ Self-healing mechanism!
```

### Controllers Chính

**1. Node Controller**
```
Nhiệm vụ:
├── Monitor Node health
├── Mark unhealthy Nodes as NotReady
├── Evict Pods từ dead Nodes (sau 40s default)
└── Update Node conditions

Watch loop (every 5s):
for each Node:
  if no heartbeat for 40s:
    Mark Node as NotReady
    Wait 5 minutes
    if still NotReady:
      Delete Pods on that Node
```

**2. Replication Controller / ReplicaSet Controller**
```
Nhiệm vụ: Đảm bảo số lượng Pods

Watch loop (every 30s):
Desired replicas: 3
Actual replicas: 2  ← Một Pod crashed!

Action:
└─> Create 1 new Pod

Example:
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: webapp-rs
spec:
  replicas: 3  ← Controller maintains this!
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx
```

**3. Endpoints Controller**
```
Nhiệm vụ: Update Service Endpoints

Watch:
├── Services created/updated
└── Pods created/deleted/changed

Action:
Service "webapp" selects app=webapp Pods
  → Find all matching Pods
  → Get their IPs
  → Update Endpoints object

Endpoints object:
apiVersion: v1
kind: Endpoints
metadata:
  name: webapp
subsets:
- addresses:
  - ip: 10.244.1.5  # Pod 1
  - ip: 10.244.2.8  # Pod 2
  - ip: 10.244.1.9  # Pod 3
  ports:
  - port: 80
```

**4. Service Account Controller**
```
Nhiệm vụ: Manage ServiceAccounts

Watch:
└── Namespaces created

Action:
When new Namespace created:
  → Create "default" ServiceAccount
  → Create token Secret for ServiceAccount

Every Pod gets a ServiceAccount (default if not specified)
```

**5. Deployment Controller**
```
Nhiệm vụ: Manage Deployments (ReplicaSets + Rolling Updates)

Watch:
└── Deployment changes

Actions:
├── New Deployment → Create ReplicaSet
├── Update image → Create new ReplicaSet, scale up/down
├── Rollback → Switch to old ReplicaSet
└── Pause/Resume → Control rollout

Example rolling update:
Deployment: nginx:1.19 → nginx:1.20
1. Create new ReplicaSet (nginx:1.20) with 0 replicas
2. Scale up new (1), scale down old (2)
3. Scale up new (2), scale down old (1)
4. Scale up new (3), scale down old (0)
→ Done! Zero downtime!
```

**6. Job Controller**
```
Nhiệm vụ: Run Jobs to completion

Watch:
└── Jobs

Actions:
├── Create Pods for Job
├── Monitor completion
├── Restart Pods nếu fail (based on restartPolicy)
└── Mark Job as Complete/Failed

Example:
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
spec:
  completions: 1  # Run once
  template:
    spec:
      containers:
      - name: backup
        image: backup:latest
      restartPolicy: Never
```

### Reconciliation Loop Pattern

**Core pattern của tất cả controllers:**

```go
// Pseudo-code
for {
  // 1. Watch for changes
  actualState = getActualState()
  desiredState = getDesiredState()
  
  // 2. Compare
  if actualState != desiredState {
    // 3. Take action
    reconcile(actualState, desiredState)
  }
  
  // 4. Wait và repeat
  sleep(30 * time.Second)
}
```

**Ví dụ ReplicaSet Controller:**

```
Loop iteration 1:
Desired: 3 Pods
Actual: 3 Pods
→ No action (all good!)

--- Pod 2 crashes ---

Loop iteration 2:
Desired: 3 Pods
Actual: 2 Pods (Pod 1, Pod 3)
→ Action: Create Pod 4!

Loop iteration 3:
Desired: 3 Pods
Actual: 3 Pods (Pod 1, Pod 3, Pod 4)
→ No action (recovered!)
```

---

## 5️⃣ Cloud Controller Manager (Optional)

### Vai Trò: Cloud Integration

**Cloud Controller Manager = Liaison với cloud provider**

```
Tách cloud-specific logic khỏi core K8s:
├── Node lifecycle (tạo/xóa VMs)
├── LoadBalancer Services (provision cloud LB)
├── Routes (configure VPC routing)
└── Storage (provision cloud volumes)
```

### Cloud-Specific Controllers

**1. Node Controller (Cloud)**
```
Nhiệm vụ:
├── Create cloud VMs for new Nodes
├── Delete cloud VMs when Nodes removed
├── Update Node labels với cloud metadata
└── Check Node existence in cloud

Example metadata:
labels:
  kubernetes.io/arch: amd64
  kubernetes.io/os: linux
  topology.kubernetes.io/region: us-central1
  topology.kubernetes.io/zone: us-central1-a
  node.kubernetes.io/instance-type: n1-standard-4
```

**2. Route Controller**
```
Nhiệm vụ:
Configure cloud VPC routes for Pod network

Example (GCP):
Node 1: Pod CIDR 10.244.1.0/24
Node 2: Pod CIDR 10.244.2.0/24

Cloud Controller creates routes:
├── 10.244.1.0/24 → Node 1 VM
└── 10.244.2.0/24 → Node 2 VM

→ Pods can communicate cross-node!
```

**3. Service Controller**
```
Nhiệm vụ:
Provision cloud LoadBalancers for Services

Example:
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: LoadBalancer  ← Cloud Controller acts!
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 8080

Cloud Controller:
1. Call cloud API: "Create LoadBalancer"
2. Get LoadBalancer IP
3. Update Service:
   status:
     loadBalancer:
       ingress:
       - ip: 35.xxx.xxx.xxx
```

---

## 🔄 Component Interaction Example

### Complete Flow: Create Deployment

```
USER: kubectl create deployment nginx --image=nginx --replicas=3
   ↓
┌─────────────────────────────────────────────┐
│ 1. API SERVER                               │
│    • Authenticate/Authorize                 │
│    • Validate Deployment spec               │
│    • Write to etcd                          │
└─────────────────┬───────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 2. DEPLOYMENT CONTROLLER                    │
│    • Watch API Server                       │
│    • Detect new Deployment                  │
│    • Create ReplicaSet                      │
│    • → API Server: "Create ReplicaSet"     │
└─────────────────┬───────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 3. REPLICASET CONTROLLER                    │
│    • Watch API Server                       │
│    • Detect new ReplicaSet (replicas: 3)    │
│    • Create 3 Pods                          │
│    • → API Server: "Create 3 Pods"         │
└─────────────────┬───────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 4. API SERVER                               │
│    • Write Pods to etcd                     │
│    • Pods status: Pending (no Node)         │
└─────────────────┬───────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 5. SCHEDULER                                │
│    • Watch for Pending Pods                 │
│    • Evaluate Nodes                         │
│    • Assign: Pod1→Node1, Pod2→Node2, Pod3→Node1 │
│    • → API Server: Update Pod.spec.nodeName│
└─────────────────┬───────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 6. API SERVER                               │
│    • Update Pods in etcd                    │
│    • Pods now have Node assignments         │
└─────────────────────────────────────────────┘
                  ↓
      [Now kubelet takes over - see Worker Nodes section]
```

---

## 🎓 Kiểm Tra Hiểu Biết

### Câu Hỏi Tự Kiểm Tra

**1. Vì sao API Server là component duy nhất giao tiếp với etcd?**
<details>
<summary>Xem đáp án</summary>

**Lý do:**
1. **Security**: Centralized access control
2. **Consistency**: Single source of truth
3. **Validation**: API Server validates tất cả writes
4. **Auditing**: Log mọi changes
5. **Simplicity**: Other components không cần biết etcd details

Nếu mọi component đều access etcd:
- ❌ Security nightmare
- ❌ Validation bị bypass
- ❌ Inconsistent data
- ❌ Hard to audit
</details>

**2. Scheduler scoring: Node nào được chọn và tại sao?**
```
Pod requests: CPU 1000m, RAM 2Gi
Nodes:
├── Node A: Available 2 CPU, 6Gi RAM (30% used)
├── Node B: Available 1.5 CPU, 3Gi RAM (50% used)
└── Node C: Available 0.5 CPU, 1Gi RAM (80% used)
```

<details>
<summary>Xem đáp án</summary>

**Filter phase:**
- Node A: ✓ (đủ resources)
- Node B: ✓ (đủ resources)
- Node C: ❌ (không đủ CPU)

**Scoring phase:**
- Node A: Score cao (nhiều available resources, balanced)
- Node B: Score trung bình (adequate resources)

**Winner: Node A**
Lý do: Nhiều resources available nhất, balanced CPU/RAM usage
</details>

**3. Controller nào responsible cho việc gì?**

Match controllers với nhiệm vụ:
1. Node Controller
2. ReplicaSet Controller
3. Endpoints Controller
4. Service Controller (Cloud)

Tasks:
a. Update Service Endpoints khi Pod IP changes
b. Maintain số lượng Pod replicas
c. Provision cloud LoadBalancer
d. Monitor Node health và evict Pods

<details>
<summary>Xem đáp án</summary>

1. Node Controller → d. Monitor Node health
2. ReplicaSet Controller → b. Maintain Pod replicas
3. Endpoints Controller → a. Update Service Endpoints
4. Service Controller (Cloud) → c. Provision LoadBalancer
</details>

---

## 💪 Bài Tập Thực Hành

### Bài 1: Trace Complete Flow

**Scenario:** Deploy webapp, 1 Pod crashes sau 5 phút

**Vẽ flow chi tiết từ lúc deploy đến lúc Pod recovered:**

<details>
<summary>Xem đáp án</summary>

```
1. kubectl create deployment webapp --replicas=2
   ↓
2. API Server: Validate, write to etcd
   ↓
3. Deployment Controller: Create ReplicaSet
   ↓
4. ReplicaSet Controller: Create 2 Pods
   ↓
5. Scheduler: Assign Pods to Nodes
   ↓
6. kubelet: Start containers
   ↓
7. Pods Running! ✅

--- 5 minutes later: Pod 1 crashes ---

8. kubelet detects crash
   ↓
9. kubelet → API Server: "Pod 1 failed"
   ↓
10. API Server → etcd: Update Pod 1 status
    ↓
11. ReplicaSet Controller watching:
    "Desired: 2, Actual: 1"
    ↓
12. ReplicaSet Controller → API Server:
    "Create new Pod"
    ↓
13. Scheduler assigns new Pod to Node
    ↓
14. kubelet starts new Pod
    ↓
15. Recovered! 2 Pods Running ✅
```
</details>

---

## 🎯 Key Takeaways

### Ghi Nhớ 5 Điều Quan Trọng

1. **API Server = Central Hub**
   - Mọi interaction qua API Server
   - Giao tiếp duy nhất với etcd
   - Authentication, Authorization, Validation

2. **etcd = Single Source of Truth**
   - Lưu tất cả cluster state
   - Distributed, consistent
   - BACKUP REGULARLY!

3. **Scheduler = Smart Assignment**
   - Filter → Score → Bind
   - Consider resources, constraints, affinity
   - Optimize cluster utilization

4. **Controllers = Reconciliation**
   - Watch → Compare → Act
   - Desired state = Actual state
   - Self-healing automatic

5. **Watch Pattern Everywhere**
   - Components watch API Server
   - React to changes
   - Distributed event-driven architecture

---

## 🚀 Tiếp Theo

Bạn đã hiểu chi tiết Control Plane!

**Next:** [2.3. Worker Nodes →](./03-worker-nodes.md)

Ở phần tiếp theo, chúng ta sẽ tìm hiểu Worker Nodes - nơi applications thực sự chạy.

---

[⬅️ 2.1. Overview](./01-overview.md) | [🏠 Mục Lục Chính](../README.md) | [📂 Phần 2: Architecture](./README.md) | [➡️ 2.3. Worker Nodes](./03-worker-nodes.md)
