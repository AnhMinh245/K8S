# 1.1. Kubernetes Là Gì?

> Hiểu Kubernetes từ vấn đề thực tế nó giải quyết

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **TẠI SAO** cần Kubernetes
- ✅ Biết Kubernetes **GIẢI QUYẾT VẤN ĐỀ GÌ**
- ✅ Nắm được các tính năng cốt lõi
- ✅ So sánh với cách deploy truyền thống

---

## 🏢 Vấn Đề Trong Thực Tế

### Câu Chuyện: Startup Phát Triển Nhanh

**Giai đoạn 1: Khởi đầu (10 users)**
```
Bạn có 1 web app đơn giản:
├── Frontend (React)
├── Backend (Node.js)
└── Database (PostgreSQL)

Deploy: Chạy trên 1 server (VPS)
Cost: $20/tháng
Problem: Không có!
```

**Giai đoạn 2: Tăng trưởng (1,000 users)**
```
Server quá tải!
├── CPU: 90%
├── Memory: 85%
├── Response time: 5 giây
└── Đêm nào cũng bị crash

Solution: Thêm server!
```

**Giai đoạn 3: Scale lên (10,000 users)**
```
Bạn mua thêm 5 servers:
├── Server 1: Frontend
├── Server 2: Backend  
├── Server 3: Backend (replica)
├── Server 4: Database
└── Server 5: Redis cache

Problems bắt đầu xuất hiện:
❌ Server 2 chết → 50% requests fail
❌ Deploy code mới → phải SSH vào từng server
❌ Traffic spike → không đủ server
❌ 3AM server chết → phải thức dậy restart
❌ Scaling thủ công quá chậm
❌ Monitoring từng server riêng → mệt mỏi
```

**Giai đoạn 4: Hỗn loạn (100,000 users)**
```
20 servers, 50 containers:
❌ Deploy mất 2 giờ
❌ Không biết container nào chạy ở đâu
❌ 1 server chết → ảnh hưởng nhiều services
❌ Không biết server nào đang overload
❌ Rollback code → phải làm thủ công
❌ Team mất ngủ mỗi đêm
```

### Vấn Đề Cốt Lõi

Khi hệ thống lớn lên, bạn cần:

1. **Tự động hóa** thay vì thủ công
2. **Self-healing** khi có lỗi xảy ra
3. **Scaling** tự động theo traffic
4. **Deploy** nhanh và an toàn
5. **Monitoring** tập trung
6. **Resource management** hiệu quả

→ **ĐÂY LÀ LÚC CẦN KUBERNETES!**

---

## 🚀 Kubernetes Là Gì?

### Định Nghĩa

**Kubernetes (K8s)** là một nền tảng mã nguồn mở để **tự động hóa việc triển khai, mở rộng và quản lý các ứng dụng container**.

### Giải Thích Bằng Ví Dụ Thực Tế

**Kubernetes giống như một "người quản lý thông minh" cho data center của bạn:**

```
🏭 Data Center = Nhà máy sản xuất
📦 Container = Công nhân
🎯 Kubernetes = Giám đốc điều hành

Kubernetes làm gì:
✓ Phân công việc cho workers (scheduling)
✓ Đảm bảo đủ workers đang làm việc (replication)
✓ Thay thế worker bị ốm (self-healing)
✓ Tăng/giảm workers theo khối lượng công việc (scaling)
✓ Phân phối công việc đều (load balancing)
✓ Cập nhật quy trình không gián đoạn (rolling updates)
```

---

## 💡 Kubernetes Giải Quyết Như Thế Nào?

### Trước Kubernetes vs Với Kubernetes

**TRƯỚC (Manual):**
```bash
# Deploy code mới (phải làm trên 20 servers!)
ssh server1 "docker pull myapp:v2 && docker restart myapp"
ssh server2 "docker pull myapp:v2 && docker restart myapp"
ssh server3 "docker pull myapp:v2 && docker restart myapp"
# ... 17 lần nữa 😫

# Server chết lúc 3 AM
→ Điện thoại reo
→ Thức dậy
→ SSH vào
→ Restart thủ công
→ Mất ngủ
```

**VỚI KUBERNETES:**
```bash
# Deploy code mới (1 command duy nhất!)
kubectl set image deployment/myapp myapp=myapp:v2

# Server/Container chết
→ Kubernetes tự động phát hiện
→ Tự động start container mới
→ Bạn ngủ ngon 😴
```

### So Sánh Cụ Thể

| Tình huống | Manual | Kubernetes |
|------------|--------|------------|
| **Deploy 50 containers** | 50 lần SSH + commands | 1 command |
| **Container crash** | Thức dậy 3 AM restart | Tự động restart |
| **Traffic tăng 10x** | Mua server, setup, config (2 ngày) | Tự động scale (2 phút) |
| **Rollback version cũ** | Revert từng server (30 phút) | 1 command (10 giây) |
| **Load balancing** | Setup nginx/HAProxy manual | Built-in |
| **Health checks** | Viết scripts riêng | Built-in |

---

## 🎯 Các Tính Năng Cốt Lõi

### 1. Container Orchestration (Điều Phối Container)

**TẠI SAO CẦN:**
Khi bạn có 100 containers, không thể quản lý thủ công được!

**KUBERNETES LÀM GÌ:**
```
Bạn nói: "Tôi cần 10 containers chạy app này"
K8s làm:
├── Tìm servers phù hợp
├── Deploy containers
├── Theo dõi health
├── Restart nếu chết
└── Đảm bảo luôn có đủ 10 containers
```

**VÍ DỤ THỰC TẾ:**
```yaml
# Bạn chỉ cần nói:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 10  # Tôi muốn 10 containers

# Kubernetes lo phần còn lại!
```

---

### 2. Self-Healing (Tự Phục Hồi)

**TẠI SAO CẦN:**
Server/Container chết là chuyện thường xuyên. Không thể trực 24/7 được!

**KUBERNETES LÀM GÌ:**
```
Container chết
    ↓
K8s phát hiện (health check)
    ↓
K8s tự động start container mới
    ↓
Service tiếp tục hoạt động
    ↓
User không hề biết có sự cố!
```

**VÍ DỤ THỰC TẾ:**
```
22:00: Container bị lỗi và crash
22:00:05: Kubernetes phát hiện
22:00:10: Container mới đã chạy
22:00:15: Service hoạt động bình thường

Bạn: Đang ngủ ngon 😴
User: Không hề biết có sự cố
```

---

### 3. Scaling (Mở Rộng Tự Động)

**TẠI SAO CẦN:**
Traffic không đều - sáng ít, tối nhiều. Black Friday tăng 100x!

**KUBERNETES LÀM GÌ:**

**Horizontal Scaling** (tăng số lượng containers):
```
Traffic thấp (8 AM):
  2 containers [□□]

Traffic cao (8 PM):
  10 containers [□□□□□□□□□□]

Black Friday:
  50 containers [□□□□□□□□□□...□□]
```

**VÍ DỤ THỰC TẾ:**
```yaml
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2    # Ít nhất 2 containers
  maxReplicas: 50   # Nhiều nhất 50 containers
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale khi CPU > 70%
```

**Kết quả:**
- CPU thấp → K8s giảm xuống 2 containers (tiết kiệm tiền)
- CPU cao → K8s tăng lên 50 containers (đảm bảo performance)
- **Tự động, không cần can thiệp!**

---

### 4. Service Discovery & Load Balancing

**TẠI SAO CẦN:**
10 containers backend, frontend cần biết gọi container nào?

**KUBERNETES LÀM GÌ:**
```
Frontend muốn gọi Backend API
    ↓
Gọi: http://backend-service
    ↓
Kubernetes tự động:
├── Tìm backend containers đang healthy
├── Phân phối request đều
└── Load balance tự động

Frontend không cần biết:
❌ Backend chạy ở server nào
❌ Backend có bao nhiêu containers
❌ IP addresses là gì
```

**VÍ DỤ THỰC TẾ:**
```yaml
# Service - Kubernetes tự động load balance
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080

# Frontend chỉ cần:
curl http://backend-service/api/users
# Kubernetes lo việc routing!
```

---

### 5. Rolling Updates & Rollbacks

**TẠI SAO CẦN:**
Deploy code mới không được downtime!

**KUBERNETES LÀM GÌ:**

**Rolling Update** (Update lần lượt):
```
Version cũ: v1 (10 containers)
[v1][v1][v1][v1][v1][v1][v1][v1][v1][v1]

Step 1: Tạo 1 container v2, xóa 1 container v1
[v1][v1][v1][v1][v1][v1][v1][v1][v1][v2]

Step 2: Tạo 1 container v2, xóa 1 container v1  
[v1][v1][v1][v1][v1][v1][v1][v1][v2][v2]

...tiếp tục cho đến khi...

Step 10: Tất cả đã là v2
[v2][v2][v2][v2][v2][v2][v2][v2][v2][v2]

✓ Zero downtime
✓ Luôn có containers serving traffic
```

**Rollback nếu có lỗi:**
```bash
# Deploy version mới
kubectl set image deployment/webapp webapp=myapp:v2

# Oh no! Version v2 có bug!
# Rollback về version cũ (10 giây)
kubectl rollout undo deployment/webapp

# Done! Về lại version v1 stable
```

---

### 6. Configuration Management

**TẠI SAO CẦN:**
Mỗi môi trường (dev, staging, prod) có config khác nhau.

**KUBERNETES LÀM GÌ:**
```
ConfigMaps: Non-sensitive config
├── API URLs
├── Feature flags
├── Settings
└── Environment variables

Secrets: Sensitive data (encrypted)
├── Database passwords
├── API keys
├── Certificates
└── Tokens
```

**VÍ DỤ THỰC TẾ:**
```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  API_URL: "https://api.production.com"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"

# Secret (encrypted at rest)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=  # base64 encoded

# Sử dụng trong Pod
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:v1
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secrets
```

---

## 🏗️ Kiến Trúc Tổng Quan

### Kubernetes Cluster

```
┌─────────────────────────────────────────────────────────┐
│                   KUBERNETES CLUSTER                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌────────────────────────────────────────┐            │
│  │     CONTROL PLANE (Bộ não)             │            │
│  │  ┌──────────────────────────────────┐  │            │
│  │  │  API Server                       │  │            │
│  │  │  (Điểm vào duy nhất)             │  │            │
│  │  └──────────────────────────────────┘  │            │
│  │  ┌──────────────────────────────────┐  │            │
│  │  │  etcd                            │  │            │
│  │  │  (Database - lưu trạng thái)     │  │            │
│  │  └──────────────────────────────────┘  │            │
│  │  ┌──────────────────────────────────┐  │            │
│  │  │  Scheduler                       │  │            │
│  │  │  (Quyết định Pod chạy ở đâu)     │  │            │
│  │  └──────────────────────────────────┘  │            │
│  │  ┌──────────────────────────────────┐  │            │
│  │  │  Controller Manager              │  │            │
│  │  │  (Đảm bảo desired state)         │  │            │
│  │  └──────────────────────────────────┘  │            │
│  └────────────────────────────────────────┘            │
│                                                         │
│  ┌────────────────────────────────────────┐            │
│  │     WORKER NODES (Thợ làm việc)       │            │
│  │                                        │            │
│  │  Node 1:                               │            │
│  │  ├─ Pod 1 [Container A, Container B]  │            │
│  │  ├─ Pod 2 [Container C]                │            │
│  │  └─ Pod 3 [Container D, Container E]  │            │
│  │                                        │            │
│  │  Node 2:                               │            │
│  │  ├─ Pod 4 [Container F]                │            │
│  │  └─ Pod 5 [Container G]                │            │
│  │                                        │            │
│  │  Node 3:                               │            │
│  │  ├─ Pod 6 [Container H]                │            │
│  │  └─ Pod 7 [Container I, Container J]  │            │
│  └────────────────────────────────────────┘            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Workflow đơn giản:**
```
1. Bạn: "kubectl create deployment webapp --image=myapp:v1"
       ↓
2. API Server: Nhận request
       ↓
3. etcd: Lưu trạng thái mong muốn
       ↓
4. Scheduler: "Đặt Pod này vào Node 2"
       ↓
5. Controller: "Đảm bảo Pod running"
       ↓
6. Node 2: Download image, start container
       ↓
7. Pod running! ✅
```

---

## 📊 So Sánh: Trước vs Sau Kubernetes

### Scenario: Deploy Microservices

**TRƯỚC KUBERNETES:**
```
❌ 5 services × 3 environments × 4 servers = 60 manual deployments
❌ Mỗi deploy: 15 phút
❌ Total: 15 hours (gần 2 ngày làm việc!)
❌ Lỗi 1 server → toàn bộ process phải làm lại
❌ Scaling: Phải provision server mới (vài giờ)
❌ Monitoring: 60 nơi khác nhau
```

**VỚI KUBERNETES:**
```
✅ 1 command: kubectl apply -f deployments/
✅ Mỗi deploy: 5 phút
✅ Total: 5 phút (nhanh hơn 180 lần!)
✅ Lỗi → tự động rollback
✅ Scaling: Tự động trong 30 giây
✅ Monitoring: Centralized dashboard
```

---

## 🎓 Kiểm Tra Hiểu Biết

### Câu Hỏi Tự Kiểm Tra

**1. Kubernetes giải quyết vấn đề gì?**
<details>
<summary>Xem đáp án</summary>

- Tự động hóa deploy, scaling, management containers
- Self-healing khi có lỗi
- Load balancing tự động
- Rolling updates zero-downtime
- Resource management hiệu quả
</details>

**2. Self-healing hoạt động như thế nào?**
<details>
<summary>Xem đáp án</summary>

1. Kubernetes liên tục check health của containers
2. Phát hiện container không healthy/crashed
3. Tự động start container mới thay thế
4. Service tiếp tục hoạt động không gián đoạn
</details>

**3. Horizontal scaling là gì? Cho ví dụ.**
<details>
<summary>Xem đáp án</summary>

Tăng/giảm số lượng containers dựa trên load:
- Traffic thấp: 2 containers
- Traffic cao: 10 containers
- Black Friday: 50 containers

Ví dụ: Web shop bình thường 5 containers, Black Friday tự động scale lên 50 containers.
</details>

**4. Tại sao cần Service trong Kubernetes?**
<details>
<summary>Xem đáp án</summary>

- Containers có IP động, thay đổi liên tục
- Service cung cấp stable endpoint (DNS name)
- Tự động load balance giữa multiple containers
- Service discovery cho các services khác
</details>

---

## 💪 Bài Tập Thực Hành

### Bài 1: Hiểu Workflow

**Tình huống:** Bạn có webapp với 3 replicas. 1 Pod crash.

**Câu hỏi:** Vẽ flow Kubernetes tự động phục hồi.

<details>
<summary>Xem đáp án</summary>

```
1. Pod 2 crash
   [Pod 1] [Pod 2 ❌] [Pod 3]
   
2. kubelet phát hiện (health check fail)
   
3. kubelet báo API Server: "Pod 2 dead"
   
4. Controller Manager thấy: Desired=3, Actual=2
   
5. Controller tạo Pod mới
   [Pod 1] [Pod 4 🆕] [Pod 3]
   
6. Scheduler đặt Pod 4 vào Node phù hợp
   
7. Node start Pod 4
   [Pod 1] [Pod 4 ✅] [Pod 3]
```
</details>

### Bài 2: So Sánh Scenarios

**Tình huống:** Traffic tăng từ 100 req/s → 1000 req/s

**Manual approach:** Bạn sẽ làm gì?
**Kubernetes approach:** K8s sẽ làm gì?

<details>
<summary>Xem đáp án</summary>

**Manual:**
1. Phát hiện server overload (monitoring)
2. Provision server mới (30 phút - vài giờ)
3. Setup OS, dependencies
4. Deploy application
5. Configure load balancer
6. Test
Total: 2-4 giờ, rủi ro cao

**Kubernetes:**
1. HPA phát hiện CPU > 70%
2. Tự động tạo thêm Pods
3. Scheduler đặt vào Nodes available
4. Service tự động load balance
Total: 30 giây - 2 phút, tự động
</details>

---

## 🎯 Key Takeaways

### Ghi Nhớ 5 Điều Quan Trọng

1. **Kubernetes = Tự động hóa quản lý containers**
   - Không cần làm thủ công nữa
   
2. **Self-healing = Ngủ ngon hơn**
   - Container chết → K8s tự restart
   
3. **Scaling = Tiết kiệm tiền + Performance**
   - Tự động tăng/giảm theo load
   
4. **Rolling updates = Zero downtime**
   - Deploy không ảnh hưởng users
   
5. **Declarative = Nói "muốn gì", không phải "làm thế nào"**
   - Bạn: "Tôi muốn 10 Pods"
   - K8s: "OK, để tôi lo!"

---

## 📚 Thuật Ngữ Cần Nhớ

| Thuật Ngữ | Tiếng Việt | Ý Nghĩa |
|-----------|------------|---------|
| **Container** | Container | Đóng gói ứng dụng + dependencies |
| **Pod** | Pod | Đơn vị nhỏ nhất, chứa 1+ containers |
| **Node** | Node | Server/máy ảo chạy Pods |
| **Cluster** | Cluster | Tập hợp các Nodes |
| **Deployment** | Deployment | Quản lý Pods, cho phép rolling updates |
| **Service** | Service | Stable endpoint để access Pods |
| **Scaling** | Mở rộng | Tăng/giảm số lượng Pods/Nodes |
| **Self-healing** | Tự phục hồi | Tự động thay thế resources lỗi |

---

## 🚀 Tiếp Theo

Bạn đã hiểu Kubernetes là gì và giải quyết vấn đề gì!

**Next:** [1.2. So Sánh Kubernetes vs Docker →](./02-k8s-vs-docker.md)

Ở phần tiếp theo, chúng ta sẽ tìm hiểu sự khác biệt giữa Kubernetes và Docker, khi nào nên dùng cái nào.

---

[🏠 Mục Lục Chính](../README.md) | [📂 Phần 1: Introduction](./README.md) | [➡️ 1.2. K8s vs Docker](./02-k8s-vs-docker.md)
