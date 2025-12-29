# 2.2. Control Plane - Bộ Não Của Cluster

> Hiểu sâu về từng component trong Control Plane

---

## 🎯 Mục Tiêu

- Hiểu vai trò chi tiết của từng component
- Biết cách các components tương tác
- Nắm được workflow của operations
- Troubleshoot được vấn đề Control Plane

---

## 🏛️ Control Plane Components

```
┌──────────────────────────────────────────────┐
│          CONTROL PLANE (Master Node)         │
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │          kube-apiserver                │ │ ← Entry point
│  │      (REST API, Authentication)        │ │
│  └─────────────┬──────────────────────────┘ │
│                │                              │
│    ┌───────────┼───────────┬────────────┐   │
│    │           │           │            │   │
│    ▼           ▼           ▼            ▼   │
│  ┌──────┐  ┌────────┐  ┌───────┐  ┌───────┐│
│  │ etcd │  │Scheduler│  │Control│  │Cloud │││
│  │      │  │        │  │ ler   │  │Ctrl  │││
│  │      │  │        │  │Manager│  │Mgr   │││
│  └──────┘  └────────┘  └───────┘  └───────┘│
│  Storage    Placement   Reconcile  Cloud   ││
│                                             ││
└──────────────────────────────────────────────┘
```

---

## 1️⃣ API Server (kube-apiserver)

### 🎯 Vai Trò

**API Server = Gateway duy nhất của Kubernetes**

- ✅ **Frontend cho Control Plane**
- ✅ **Nhận mọi requests** (kubectl, controllers, kubelet...)
- ✅ **Authentication & Authorization**
- ✅ **Validation**
- ✅ **Interface với etcd**

### 🏢 Ví Dụ Thực Tế

**API Server = Receptionist (lễ tân) của công ty**

```
Khách đến (kubectl):
  1. Security check (authentication)
  2. Kiểm tra quyền hạn (authorization)
  3. Kiểm tra yêu cầu hợp lệ (validation)
  4. Chuyển đến bộ phận phù hợp (routing)
  5. Ghi nhận vào sổ sách (persist to etcd)
```

### 🔄 Workflow: Create Deployment

```
1. kubectl create deployment
   ↓
2. API Server receives HTTP POST request
   ↓
3. Authentication: "Who are you?"
   - Check certificate, token, or credentials
   ↓
4. Authorization (RBAC): "Can you do this?"
   - Check permissions
   ↓
5. Admission Control: "Is this allowed?"
   - Mutating webhooks (modify request)
   - Validating webhooks (accept/reject)
   ↓
6. Validation: "Is YAML correct?"
   - Schema validation
   - Required fields present
   ↓
7. Persist to etcd
   - Write Deployment object to database
   ↓
8. Return response to kubectl
   "Deployment created"
   ↓
9. Watch mechanism triggers
   - Controller Manager notified
```

### 🔧 Chức Năng Chi Tiết

#### Authentication (Xác thực)
**Câu hỏi:** "Bạn là ai?"

**Methods:**
- **Client certificates:** X.509 certs
- **Bearer tokens:** Static tokens, service account tokens
- **Basic auth:** Username/password (deprecated, không nên dùng)
- **OpenID Connect:** SSO integration

**Example:**
```bash
# kubectl with certificate
kubectl --client-certificate=user.crt \
        --client-key=user.key \
        --server=https://k8s-api:6443 \
        get pods
```

#### Authorization (Phân quyền)
**Câu hỏi:** "Bạn có quyền làm việc này không?"

**Modes:**
- **RBAC (Role-Based Access Control):** Most common ⭐
- **ABAC (Attribute-Based):** Complex, less used
- **Webhook:** External authorization service
- **Node:** Special authorization for kubelets

**RBAC Example:**
```yaml
# User "john" can only GET pods in namespace "dev"
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
- kind: User
  name: john
roleRef:
  kind: Role
  name: pod-reader
```

#### Admission Control
**Câu hỏi:** "Request này có compliant với policies không?"

**Two phases:**
1. **Mutating Admission:** Modify request
   - Add default values
   - Inject sidecars
   - Set resource limits

2. **Validating Admission:** Accept or reject
   - Enforce naming conventions
   - Require labels
   - Block privileged containers

**Built-in Admission Controllers:**
- `NamespaceLifecycle`: Prevent operations on terminating namespaces
- `LimitRanger`: Enforce resource limits
- `ServiceAccount`: Auto-inject service account tokens
- `ResourceQuota`: Enforce quotas
- `PodSecurityPolicy`: Security policies (deprecated in v1.25)

**Custom Admission Webhooks:**
```yaml
# Example: Require all Deployments have label "team"
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: require-team-label
webhooks:
- name: validate.example.com
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
  clientConfig:
    service:
      name: validation-service
      namespace: default
```

### 📡 API Server Features

#### RESTful API
```bash
# All kubectl commands = HTTP requests

# kubectl get pods
GET /api/v1/namespaces/default/pods

# kubectl create -f deployment.yaml
POST /apis/apps/v1/namespaces/default/deployments

# kubectl delete pod nginx
DELETE /api/v1/namespaces/default/pods/nginx
```

#### Watch Mechanism
**Controllers watch for changes:**
```
Controller:
  watch /api/v1/pods
  
API Server:
  Keeps connection open
  Sends events when changes occur:
    ADDED: new pod created
    MODIFIED: pod updated
    DELETED: pod removed
```

**Ví dụ:**
```bash
# Watch pods (like controllers do)
kubectl get pods --watch

# Output:
# NAME    STATUS    AGE
# nginx   Pending   0s     ← ADDED event
# nginx   Running   5s     ← MODIFIED event
# nginx   Terminating 30s  ← MODIFIED event
#                          ← DELETED event
```

### 🔒 Security Best Practices

1. **Enable TLS:** Always use HTTPS
2. **Strong authentication:** Use certificates, not passwords
3. **RBAC enabled:** Principle of least privilege
4. **Audit logging:** Log all API calls
5. **Network policies:** Restrict access to API server
6. **Update regularly:** Patch security vulnerabilities

---

## 2️⃣ etcd

### 🎯 Vai Trò

**etcd = Database của Kubernetes**

- ✅ **Distributed key-value store**
- ✅ **Lưu entire cluster state**
- ✅ **Consistent và highly-available**
- ✅ **Source of truth**

### 🏢 Ví Dụ Thực Tế

**etcd = Kho lưu trữ hồ sơ của công ty**

```
Mọi thông tin quan trọng:
- Danh sách nhân viên (Pods)
- Cơ cấu tổ chức (Deployments, Services)
- Lịch sử thay đổi (Revisions)
- Cấu hình (ConfigMaps, Secrets)

Đặc điểm:
- Luôn cập nhật (consistent)
- Sao lưu nhiều bản (distributed)
- Mọi quyết định dựa trên dữ liệu này
```

### 💾 etcd Stores

**Everything in Kubernetes:**
- Pods, Deployments, Services
- ConfigMaps, Secrets
- Namespaces
- Resource quotas
- RBAC roles
- Network policies
- Custom resources

**Data structure:**
```
/registry/pods/default/nginx
/registry/deployments/production/web-app
/registry/services/default/api-service
/registry/secrets/default/db-credentials
...
```

### 🔍 etcd Internals

#### Raft Consensus Algorithm
**Problem:** Nhiều etcd nodes, làm sao đảm bảo consistent?

**Solution:** Raft consensus

```
etcd cluster (3 members):
  Node 1 (Leader)    ← Nhận writes
  Node 2 (Follower)  ← Replicate
  Node 3 (Follower)  ← Replicate

Write flow:
  1. Client sends write to Leader
  2. Leader replicates to Followers
  3. Wait for majority (2/3) to confirm
  4. Commit write
  5. Return success
```

**Quorum (Majority):**
- 1 node: No HA (single point of failure)
- 2 nodes: ❌ Can't form quorum if 1 fails
- 3 nodes: ✅ Tolerates 1 failure (recommended minimum)
- 5 nodes: ✅ Tolerates 2 failures (production)
- 7 nodes: ✅ Tolerates 3 failures (large clusters)

**Rule:** `quorum = (n/2) + 1`

#### Watch & Notification
**Efficient change detection:**
```
API Server watches etcd:
  watch /registry/pods/
  
etcd:
  When pod changes → Notify API Server immediately
  No polling needed → Efficient
```

### 🛡️ etcd Best Practices

1. **Odd number of members:** 3, 5, or 7
2. **Fast disks:** SSD required (etcd is I/O intensive)
3. **Separate etcd cluster:** Don't colocate with heavy workloads
4. **Regular backups:** Disaster recovery
5. **Monitor latency:** Should be < 10ms
6. **Secure:** TLS for client-server and peer-to-peer

### 💾 Backup & Restore

**Backup:**
```bash
# Backup etcd snapshot
etcdctl snapshot save /backup/etcd-snapshot.db

# Verify backup
etcdctl snapshot status /backup/etcd-snapshot.db
```

**Restore:**
```bash
# Restore from snapshot
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore
```

**Frequency:** Daily or more frequent (production)

---

## 3️⃣ Scheduler (kube-scheduler)

### 🎯 Vai Trò

**Scheduler = HR Manager**

- ✅ **Assign Pods to Nodes**
- ✅ **Resource optimization**
- ✅ **Constraint satisfaction**
- ✅ **Load balancing**

### 🏢 Ví Dụ Thực Tế

**Scheduler = Quản lý nhân sự phân công công việc**

```
Có 1 project mới (Pod):
  Yêu cầu:
    - 2 CPU cores
    - 4 GB RAM
    - GPU available
    - Location: Europe zone
  
Scheduler:
  1. Tìm danh sách nhân viên (Nodes) phù hợp
  2. Lọc: Loại bỏ không đủ điều kiện
  3. Chấm điểm: Node nào tốt nhất
  4. Phân công: Assign Pod to Node
```

### 🔄 Scheduling Process

```
┌─────────────────────────────────────┐
│  1. Watch for unscheduled Pods      │
│     (Pods with spec.nodeName="")    │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│  2. Filtering Phase                 │
│     Find Nodes that fit             │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│  3. Scoring Phase                   │
│     Rank Nodes by score             │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│  4. Binding                         │
│     Assign Pod to best Node         │
└─────────────────────────────────────┘
```

### 🔍 Filtering Phase (Predicates)

**Loại bỏ Nodes không phù hợp:**

| Filter | Mô Tả | Ví Dụ |
|--------|-------|-------|
| **NodeResourcesFit** | Đủ CPU/RAM? | Pod cần 2GB, Node còn 1GB → Fail |
| **NodeName** | Pod chỉ định Node cụ thể? | nodeName: node-1 → Chỉ node-1 |
| **NodeSelector** | Match labels? | nodeSelector: gpu=true → Chỉ Node có GPU |
| **NodeAffinity** | Advanced node selection | Prefer zone=us-west |
| **PodAffinity** | Co-locate với Pod khác? | Web pod gần Redis pod |
| **PodAntiAffinity** | Tách xa Pod khác? | 2 replicas khác Node |
| **Taints/Tolerations** | Node taint? | Node tainted "dedicated=gpu" |
| **Volume** | Volume có mount được? | AWS EBS chỉ trong 1 AZ |

**Example:**
```
Pod requires:
  CPU: 2 cores
  Memory: 4 GB
  GPU: yes
  Zone: us-west-1a

Cluster:
  Node 1: 8 cores, 16GB, GPU, us-west-1a ✅
  Node 2: 4 cores, 8GB, no GPU, us-west-1a ❌ (no GPU)
  Node 3: 8 cores, 16GB, GPU, us-east-1a ❌ (wrong zone)

Filtered nodes: [Node 1]
```

### 📊 Scoring Phase (Priorities)

**Chấm điểm Nodes:**

| Plugin | Mô Tả | Score |
|--------|-------|-------|
| **LeastResourceAllocation** | Prefer Node ít workload | More available resources = Higher score |
| **MostResourceAllocation** | Pack Pods densely | Less available resources = Higher score |
| **BalancedResourceAllocation** | Balance CPU/Memory usage | Even CPU and Memory % = Higher score |
| **ImageLocality** | Image đã có sẵn? | Image already pulled = Higher score |
| **InterPodAffinity** | Gần Pod affinity? | Match affinity = Higher score |

**Example:**
```
Pod requires: 2 CPU, 4GB RAM

Node 1: 4/8 CPU used, 8/16GB used
  Score: 50 (balanced)

Node 2: 2/8 CPU used, 12/16GB used
  Score: 30 (unbalanced - high memory %)

Node 3: 1/8 CPU used, 2/16GB used
  Score: 80 (lots of free resources)

Winner: Node 3 (highest score)
```

### 🎨 Advanced Scheduling

#### Node Affinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - node-1
            - node-2
```

#### Pod Affinity (Co-location)
```yaml
# Place this Pod on same Node as Pods with label app=cache
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values:
        - cache
    topologyKey: kubernetes.io/hostname
```

#### Taints & Tolerations
```bash
# Taint Node (mark as special-purpose)
kubectl taint nodes node-1 dedicated=gpu:NoSchedule

# Pod must tolerate to schedule on node-1
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"
```

---

## 4️⃣ Controller Manager (kube-controller-manager)

### 🎯 Vai Trò

**Controller Manager = Giám sát viên**

- ✅ **Runs multiple controllers**
- ✅ **Reconcile desired vs actual state**
- ✅ **Self-healing**
- ✅ **Automation**

### 🏢 Ví Dụ Thực Tế

**Controller Manager = Tổ trưởng các giám sát viên**

```
Mỗi Controller = 1 Giám sát viên chuyên trách:

- Replication Controller:
    "Đảm bảo luôn có đủ 3 nhân viên ca sáng"
    Nếu 1 người ốm → Gọi người thay thế
  
- Node Controller:
    "Kiểm tra các chi nhánh còn hoạt động không"
    Chi nhánh mất liên lạc → Báo cáo và xử lý
  
- Endpoints Controller:
    "Cập nhật danh bạ điện thoại khi có thay đổi"
    Nhân viên mới vào → Thêm vào danh bạ
```

### 🔄 Control Loop Pattern

**Every controller runs this loop:**

```go
for {
  desired_state = get_from_etcd()
  actual_state = observe_reality()
  
  if desired_state != actual_state {
    take_action_to_reconcile()
  }
  
  sleep(resync_period)
}
```

### 📋 Built-in Controllers

#### 1. Replication Controller / ReplicaSet Controller
**Đảm bảo số lượng Pod:**
```
Desired: 3 replicas
Actual: 2 replicas (1 Pod crashed)

Action: Create 1 new Pod
```

#### 2. Deployment Controller
**Quản lý rollouts:**
```
User: kubectl set image deployment/web nginx=1.21

Controller:
  1. Create new ReplicaSet với image mới
  2. Scale up new ReplicaSet (1 → 2 → 3)
  3. Scale down old ReplicaSet (3 → 2 → 1 → 0)
```

#### 3. StatefulSet Controller
**Ordered, stable Pod identities:**
```
Desired: StatefulSet với 3 Pods

Controller:
  1. Create mysql-0 → Wait until Running
  2. Create mysql-1 → Wait until Running
  3. Create mysql-2 → Wait until Running
  
Scale down:
  1. Delete mysql-2
  2. Wait until terminated
  3. Delete mysql-1
  4. ...
```

#### 4. DaemonSet Controller
**1 Pod per Node:**
```
New Node added to cluster

Controller:
  Detect new Node
  → Create monitoring-agent Pod on that Node
```

#### 5. Job Controller
**Run-to-completion:**
```
Job: Run batch task

Controller:
  1. Create Pod
  2. Monitor Pod
  3. Pod exits with code 0 (success)
  4. Mark Job as Complete
  5. Don't restart Pod
```

#### 6. Node Controller
**Monitor Node health:**
```
Controller checks Node heartbeat:
  - Every 5s: kubelet sends heartbeat
  - No heartbeat for 40s: Mark Node "Unknown"
  - Unknown for 5 minutes: Evict Pods from Node
```

#### 7. Service Controller / Endpoints Controller
**Update Service endpoints:**
```
Service "web" selects Pods with label app=web

Controller watches:
  - Pod with app=web created → Add to endpoints
  - Pod deleted → Remove from endpoints
  - Pod not ready → Remove from endpoints
```

#### 8. Namespace Controller
**Cleanup on namespace deletion:**
```
kubectl delete namespace dev

Controller:
  1. Set namespace status = Terminating
  2. Delete all objects in namespace (Pods, Services, etc)
  3. Wait until all objects deleted
  4. Delete namespace itself
```

### 🎯 Custom Controllers (Operators)

**Extend K8s with custom logic:**

**Example: MySQL Operator**
```yaml
# Custom Resource
apiVersion: mysql.example.com/v1
kind: MySQLCluster
metadata:
  name: my-db
spec:
  replicas: 3
  version: "8.0"
  
# Custom Controller watches MySQLCluster resources
# Creates: StatefulSet, Service, ConfigMap, PV/PVC
# Manages: Backups, failover, upgrades
```

---

## 5️⃣ Cloud Controller Manager

### 🎯 Vai Trò

**Integrate với cloud providers**

- ✅ **Node management** (cloud VMs)
- ✅ **LoadBalancer Service** (create cloud LB)
- ✅ **Route management** (VPC routes)
- ✅ **Volume management** (cloud disks)

### 🏢 Ví Dụ

**AWS:**
```
Service type: LoadBalancer

Cloud Controller Manager:
  1. Call AWS API
  2. Create ELB (Elastic Load Balancer)
  3. Configure target group
  4. Update Service with LB hostname
```

**Controllers:**
- **Node Controller:** Sync cloud VMs với K8s Nodes
- **Route Controller:** Setup network routes
- **Service Controller:** Create/delete cloud load balancers

---

## 🔗 Component Interactions

### Scenario: Create Deployment

```
┌─────────┐
│ kubectl │ kubectl create deployment web --image=nginx
└────┬────┘
     │
     ▼
┌────────────┐
│ API Server │ 1. Receive request
└────┬───────┘ 2. Authenticate, authorize
     │         3. Validate
     │         4. Write to etcd
     ▼
┌────────┐
│  etcd  │ Store Deployment object
└────────┘
     │
     │ (watch)
     ▼
┌──────────────────┐
│ Deployment       │ 5. Detect new Deployment
│ Controller       │ 6. Create ReplicaSet
└──────┬───────────┘
       │
       ▼
┌────────────┐
│ API Server │ 7. Write ReplicaSet to etcd
└────┬───────┘
     │
     │ (watch)
     ▼
┌──────────────────┐
│ ReplicaSet       │ 8. Detect new ReplicaSet
│ Controller       │ 9. Create 3 Pods
└──────┬───────────┘
       │
       ▼
┌────────────┐
│ API Server │ 10. Write Pods to etcd (unscheduled)
└────┬───────┘
     │
     │ (watch)
     ▼
┌──────────────────┐
│ Scheduler        │ 11. Detect unscheduled Pods
└──────┬───────────┘ 12. Find best Nodes
       │             13. Bind Pods to Nodes
       ▼
┌────────────┐
│ API Server │ 14. Update Pods with nodeName
└────┬───────┘
     │
     │ (kubelet watches on assigned Node)
     ▼
┌──────────────────┐
│ kubelet (Node)   │ 15. Detect Pod assigned to this Node
└──────────────────┘ 16. Pull image, start container
```

---

## 🎓 Key Takeaways

1. **API Server:** Gateway, authentication, validation
2. **etcd:** Database, source of truth, distributed
3. **Scheduler:** Assign Pods to Nodes intelligently
4. **Controller Manager:** Reconcile state, self-healing
5. **Cloud Controller:** Cloud integration
6. **Everything is async:** Components watch and react
7. **Declarative:** State stored, controllers ensure it matches reality

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. API Server làm gì với mỗi request?
2. etcd lưu trữ thông tin gì? Tại sao cần backup?
3. Scheduler quyết định Pod chạy trên Node nào bằng cách nào?
4. Controller Manager có nhiệm vụ gì?
5. Vẽ flow khi user chạy `kubectl create deployment`

---

## 🚀 Tiếp Theo

👉 [2.3. Worker Node - Nơi Chạy Workload](./03-worker-nodes.md)

---

[⬅️ 2.1. Overview](./01-overview.md) | [⬆️ Phần 2: Architecture](./README.md) | [🏠 Mục Lục Chính](../README.md)

