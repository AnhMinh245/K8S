# Hướng Dẫn Học Kubernetes Từ Đầu

## 1. Kubernetes Là Gì? Giải Quyết Vấn Đề Gì?

### 🎯 Vấn đề thực tế

Tưởng tượng bạn là chủ một nhà hàng (ứng dụng web):

- **Giờ thấp điểm** (3h sáng): Chỉ cần 2 nhân viên
- **Giờ cao điểm** (12h trưa): Cần 20 nhân viên
- **Nếu 1 nhân viên ốm**: Cần người thay thế ngay
- **Nhiều chi nhánh**: Cần quản lý đồng bộ

**Kubernetes (K8s)** là "người quản lý tự động" giúp bạn:

- ✅ Tự động tăng/giảm số lượng container khi cần
- ✅ Thay thế container bị lỗi tự động
- ✅ Phân phối traffic đều giữa các container
- ✅ Cập nhật ứng dụng không downtime
- ✅ Quản lý hàng trăm/ngàn container trên nhiều máy chủ

### 💡 Vì sao cần Kubernetes?

Khi chạy container trong production, bạn cần:

1. **Orchestration** (điều phối): Chạy container ở đâu? Bao nhiêu bản?
2. **Self-healing**: Container chết → tự động khởi động lại
3. **Scaling**: Tự động scale khi traffic tăng
4. **Load balancing**: Phân phối request đều
5. **Rolling updates**: Cập nhật ứng dụng an toàn
6. **Service discovery**: Các container tìm nhau như thế nào?

---

## 2. Kubernetes vs Docker Standalone

| Tiêu chí | Docker Standalone | Kubernetes |
|----------|-------------------|------------|
| **Phạm vi** | Chạy container trên 1 máy | Quản lý container trên nhiều máy (cluster) |
| **Tự động hóa** | Thủ công restart khi container chết | Tự động restart, thay thế |
| **Scale** | `docker run` thêm container thủ công | Tự động scale theo CPU/memory/traffic |
| **Load balancing** | Cần công cụ bên ngoài (nginx, HAProxy) | Built-in (Service) |
| **Update** | Stop → Start thủ công | Rolling update tự động |
| **Phù hợp** | Dev/Test, ứng dụng nhỏ | Production, ứng dụng lớn, multi-server |

**Ví dụ thực tế:**

- **Docker**: Bạn có 1 cửa hàng nhỏ, tự quản lý được
- **Kubernetes**: Bạn có chuỗi 100 cửa hàng, cần hệ thống quản lý tập trung

---

## 3. Kiến Trúc Kubernetes

Kubernetes = **Cluster** (cụm máy chủ) gồm 2 loại node:

```
┌─────────────────────────────────────────────────┐
│               KUBERNETES CLUSTER                │
│                                                 │
│  ┌───────────────────────────────────────┐     │
│  │        CONTROL PLANE (Master)         │     │
│  │  ┌──────────┐  ┌──────────────────┐  │     │
│  │  │   API    │  │    Scheduler     │  │     │
│  │  │  Server  │  │  (đặt lịch Pod)  │  │     │
│  │  └──────────┘  └──────────────────┘  │     │
│  │  ┌──────────┐  ┌──────────────────┐  │     │
│  │  │  etcd    │  │   Controller     │  │     │
│  │  │ (lưu trữ)│  │    Manager       │  │     │
│  │  └──────────┘  └──────────────────┘  │     │
│  └───────────────────────────────────────┘     │
│              ↓  ↓  ↓                            │
│  ┌─────────────────────────────────────────┐   │
│  │         WORKER NODES (Minions)          │   │
│  │  ┌────────────┐  ┌────────────┐         │   │
│  │  │  Node 1    │  │  Node 2    │  ...    │   │
│  │  │  ┌──────┐  │  │  ┌──────┐  │         │   │
│  │  │  │ Pod  │  │  │  │ Pod  │  │         │   │
│  │  │  │ Pod  │  │  │  │ Pod  │  │         │   │
│  │  │  └──────┘  │  │  └──────┘  │         │   │
│  │  │  kubelet   │  │  kubelet   │         │   │
│  │  │  kube-proxy│  │  kube-proxy│         │   │
│  │  └────────────┘  └────────────┘         │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### 3.1. Control Plane (Bộ não của cluster)

**Vai trò:** Ra quyết định, quản lý toàn bộ cluster

#### 🔵 API Server

- **Là gì:** Cổng giao tiếp duy nhất của K8s
- **Vai trò:** Nhận lệnh (từ kubectl, dashboard), xử lý, lưu vào etcd
- **Ví dụ:** Như "tổng đài" của công ty, mọi yêu cầu đều qua đây

#### 🔵 etcd

- **Là gì:** Database phân tán, lưu trữ toàn bộ trạng thái cluster
- **Vai trò:** "Bộ nhớ" của K8s - lưu cấu hình, trạng thái của mọi thứ
- **Ví dụ:** Như "sổ sách" ghi chép mọi thông tin

#### 🔵 Scheduler

- **Là gì:** Bộ phận lập lịch
- **Vai trò:** Quyết định Pod sẽ chạy trên Worker Node nào
- **Tiêu chí:** CPU/RAM còn trống, các ràng buộc (constraints)
- **Ví dụ:** Như "quản lý nhân sự" phân công nhân viên vào ca làm việc

#### 🔵 Controller Manager

- **Là gì:** Bộ điều khiển liên tục giám sát và duy trì trạng thái mong muốn
- **Vai trò:** "Giám sát viên" - liên tục kiểm tra và sửa sai
- **Ví dụ:**
  - Bạn muốn 3 Pod → Controller đảm bảo luôn có đúng 3 Pod
  - 1 Pod chết → Controller tạo Pod mới ngay lập tức

**Các loại Controller quan trọng:**

- **Node Controller:** Giám sát Node, phát hiện Node chết
- **Replication Controller:** Đảm bảo số lượng Pod mong muốn
- **Endpoints Controller:** Kết nối Service với Pod
- **Service Account Controller:** Quản lý quyền truy cập

### 3.2. Worker Node (Người lao động thực tế)

**Vai trò:** Nơi các container/Pod thực sự chạy

#### 🟢 kubelet

- **Là gì:** "Agent" chạy trên mỗi Worker Node
- **Vai trò:**
  - Nhận lệnh từ API Server
  - Đảm bảo các Pod chạy đúng như mô tả
  - Báo cáo trạng thái về Control Plane
- **Ví dụ:** Như "trưởng ca" tại mỗi cửa hàng

#### 🟢 kube-proxy

- **Là gì:** Network proxy
- **Vai trò:** Quản lý network rules để Pod có thể giao tiếp
- **Ví dụ:** Như "tổng đài viên nội bộ" điều hướng cuộc gọi

#### 🟢 Container Runtime

- **Là gì:** Phần mềm chạy container (Docker, containerd, CRI-O)
- **Vai trò:** Thực thi việc chạy/dừng container
- **Ví dụ:** Như "động cơ" thực sự làm việc

---

## 4. Các Khái Niệm Cốt Lõi

### 4.1. Cluster

**Là gì:** Toàn bộ hệ thống K8s (Control Plane + Worker Nodes)

**Ví dụ:** Cả tập đoàn với văn phòng trung tâm + các chi nhánh

### 4.2. Node

**Là gì:** 1 máy chủ (vật lý hoặc VM) trong cluster

**Loại:**

- **Master Node:** Chạy Control Plane
- **Worker Node:** Chạy ứng dụng

**Ví dụ:** Mỗi Node = 1 tòa nhà văn phòng

### 4.3. Pod ⭐ (Quan trọng nhất)

**Là gì:** Đơn vị nhỏ nhất trong K8s, chứa 1 hoặc nhiều container

**Đặc điểm:**

- 1 Pod = 1 IP address duy nhất
- Các container trong cùng Pod share network & storage
- Pod là "ephemeral" (tạm thời) - có thể bị xóa/tạo lại bất cứ lúc nào

**Vì sao không chạy container trực tiếp?**

- Pod cho phép gom nhiều container liên quan (sidecar pattern)
- Abstraction layer giúp K8s quản lý dễ hơn

**Ví dụ:**

```
Pod = 1 Chiếc xe buýt
- Container chính = Tài xế (ứng dụng chính)
- Sidecar container = Phụ xe (logging, monitoring)
- Chúng đi cùng nhau, chia sẻ không gian
```

**Khi nào dùng 1 Pod nhiều container?**

- Container phụ hỗ trợ container chính
- Ví dụ: Web app + Log collector trong cùng Pod

### 4.4. Namespace

**Là gì:** "Phòng ban" ảo trong cluster, phân chia resources

**Vì sao cần:**

- Tách biệt môi trường: `dev`, `staging`, `production`
- Phân quyền: Team A không thấy resources của Team B
- Quản lý quota: Giới hạn CPU/RAM cho mỗi namespace

**Namespace mặc định:**

- `default`: Namespace mặc định
- `kube-system`: Các thành phần hệ thống K8s
- `kube-public`: Public, ai cũng truy cập được
- `kube-node-lease`: Heartbeat của Node

**Ví dụ:**

```
Công ty (Cluster)
├── Phòng Dev (namespace: dev)
├── Phòng Test (namespace: test)
└── Phòng Prod (namespace: prod)
```

### 4.5. Label & Selector

#### Label

**Là gì:** Key-value pairs gắn vào object (Pod, Service, Node...)

**Mục đích:** Tổ chức và query objects

**Ví dụ:**

```yaml
labels:
  app: web
  environment: production
  version: v1.2
  tier: frontend
```

**Ví dụ thực tế:** Như "nhãn dán" trên hồ sơ

- Hồ sơ A: `[Khẩn cấp] [Phòng Kế toán] [2024]`
- Hồ sơ B: `[Thường] [Phòng IT] [2024]`

#### Selector

**Là gì:** Cách tìm/chọn objects dựa trên Label

**Ví dụ:**

- "Tìm tất cả Pod có label `app=web` và `environment=production`"
- Service dùng Selector để biết gửi traffic đến Pod nào

**Tại sao quan trọng:**

- Service → tìm Pod bằng Selector
- Deployment → quản lý Pod bằng Label
- Monitoring → group metrics theo Label

---

## 5. Workload Cơ Bản

### 5.1. ReplicaSet

**Là gì:** Đảm bảo số lượng Pod mong muốn luôn chạy

**Vai trò:**

- Bạn muốn 3 bản copy của ứng dụng → ReplicaSet đảm bảo luôn có 3 Pod
- 1 Pod chết → Tự động tạo Pod mới

**Ví dụ:**

```
Desired State: 3 Pod
Current State: 2 Pod (1 Pod bị lỗi)
→ ReplicaSet tự động tạo thêm 1 Pod mới
```

**Lưu ý:** Hiếm khi tạo ReplicaSet trực tiếp, thường dùng Deployment

### 5.2. Deployment ⭐ (Dùng nhiều nhất)

**Là gì:** Quản lý ReplicaSet, hỗ trợ rolling update và rollback

**Vì sao cần:**

- **Update ứng dụng không downtime:**
  - Tạo Pod mới version mới
  - Chờ Pod mới ready
  - Xóa Pod cũ từ từ
- **Rollback nhanh:** Lùi về version cũ 1 lệnh
- **Scale dễ dàng:** Tăng/giảm số Pod

**Ví dụ thực tế:**

```
Version 1.0: 3 Pod đang chạy
Deploy Version 1.1:
  Step 1: Tạo 1 Pod v1.1 (total: 3 v1.0 + 1 v1.1)
  Step 2: Xóa 1 Pod v1.0 (total: 2 v1.0 + 1 v1.1)
  Step 3: Tạo 1 Pod v1.1 (total: 2 v1.0 + 2 v1.1)
  ...
  Step N: 3 Pod v1.1
```

**Khi nào dùng:** Hầu hết ứng dụng stateless (web app, API, microservices)

### 5.3. StatefulSet

**Là gì:** Giống Deployment nhưng cho ứng dụng **stateful** (có trạng thái)

**Đặc điểm:**

- Mỗi Pod có **identity cố định** (tên không đổi)
- Pod tạo/xóa **theo thứ tự** (0 → 1 → 2...)
- Mỗi Pod có **storage riêng**, không mất khi Pod restart

**Vì sao cần:**

- Database (MySQL, PostgreSQL, MongoDB)
- Distributed systems (Kafka, Elasticsearch, Redis Cluster)

**Ví dụ Database Cluster:**

```
mysql-0: Master (read/write)
mysql-1: Slave 1 (read-only)
mysql-2: Slave 2 (read-only)

Nếu mysql-1 chết → Pod mới vẫn tên mysql-1, mount đúng volume cũ
```

**So sánh:**

- **Deployment:** Pod tên random (web-abc123, web-xyz789) → Xóa/tạo tên mới
- **StatefulSet:** Pod tên cố định (db-0, db-1, db-2) → Xóa/tạo giữ nguyên tên

**Khi nào dùng:**

- ✅ Database
- ✅ Distributed storage
- ✅ App cần network identity ổn định
- ❌ Web app thông thường (dùng Deployment)

### 5.4. DaemonSet

**Là gì:** Chạy **1 Pod trên mỗi Node** (hoặc một nhóm Node cụ thể)

**Vì sao cần:**

- Monitoring agent: Cần thu thập metrics từ mọi Node
- Log collector: Lấy log từ mọi Node
- Network plugin: Mỗi Node cần network driver

**Ví dụ:**

```
Cluster: 5 Nodes
DaemonSet (log-collector) → Tự động tạo 5 Pod (1 Pod/Node)

Thêm Node thứ 6 → DaemonSet tự động tạo Pod thứ 6
```

**Use cases thực tế:**

- **Monitoring:** Prometheus Node Exporter trên mỗi Node
- **Logging:** Fluentd/Filebeat thu log từ mọi Node
- **Security:** Antivirus agent trên mỗi Node
- **Storage:** Ceph, GlusterFS storage daemon

**Khi nào dùng:**

- ✅ Cần chạy service trên MỌI Node
- ❌ Application logic (dùng Deployment)

### 5.5. Job

**Là gì:** Chạy Pod đến khi **hoàn thành task** rồi dừng

**Đặc điểm:**

- Pod chạy 1 lần xong → Trạng thái "Completed"
- Nếu Pod fail → Job tự động retry

**Ví dụ use case:**

- Data migration: Chuyển dữ liệu từ DB cũ sang DB mới
- Batch processing: Xử lý hàng nghìn file ảnh
- Database backup: Chạy backup lúc 2h sáng
- ETL jobs: Extract, Transform, Load data

**Ví dụ thực tế:**

```
Job: Gửi email cho 10,000 user
→ K8s tạo Pod
→ Pod gửi hết email
→ Pod "Completed" (không restart)
```

**Parallel Jobs:**

- Có thể chạy nhiều Pod song song
- Ví dụ: 10 Pod cùng xử lý 10,000 file

### 5.6. CronJob

**Là gì:** Job chạy theo **lịch định kỳ** (giống crontab Linux)

**Ví dụ:**

```
CronJob: Backup database mỗi đêm 2h sáng
Schedule: "0 2 * * *"
→ K8s tự động tạo Job lúc 2h
→ Job tạo Pod → Backup → Completed
```

**Use cases:**

- ✅ Backup định kỳ
- ✅ Gửi báo cáo hàng ngày
- ✅ Dọn dẹp log cũ
- ✅ Đồng bộ dữ liệu
- ✅ Health check định kỳ

**Cron syntax:**

```
┌───────────── phút (0 - 59)
│ ┌───────────── giờ (0 - 23)
│ │ ┌───────────── ngày trong tháng (1 - 31)
│ │ │ ┌───────────── tháng (1 - 12)
│ │ │ │ ┌───────────── thứ trong tuần (0 - 6)
│ │ │ │ │
* * * * *

Ví dụ:
"0 2 * * *"       → 2h sáng mỗi ngày
"*/15 * * * *"    → Mỗi 15 phút
"0 0 * * 0"       → 00h Chủ nhật hàng tuần
```

---

## 6. Networking

### 6.1. Pod Giao Tiếp Như Thế Nào?

**Nguyên tắc Network trong K8s:**

1. **Mỗi Pod có 1 IP duy nhất**
   - Pod A (IP: 10.1.1.5)
   - Pod B (IP: 10.1.1.6)

2. **Pod-to-Pod communication không cần NAT**
   - Pod A có thể gọi trực tiếp `http://10.1.1.6:8080`
   - Dù 2 Pod ở 2 Node khác nhau

3. **Container trong cùng Pod share network**
   - localhost giữa các container
   - Container 1 nghe port 8080 → Container 2 gọi `localhost:8080`

**Vấn đề:** IP của Pod thay đổi khi Pod restart!
→ **Giải pháp:** Service (xem phần tiếp)

### 6.2. Service ⭐

**Vấn đề cần giải quyết:**

- Pod IP thay đổi khi Pod chết/tạo lại
- Có nhiều Pod replicas → Gọi Pod nào?
- Cần load balancing giữa các Pod

**Service là gì:**

- **Stable endpoint** (IP/DNS cố định) trỏ đến một nhóm Pod
- Load balancing tự động
- Service discovery

**Cách hoạt động:**

```
Client → Service (IP: 10.0.0.100) 
         → Load balance → Pod 1 (10.1.1.5)
                       → Pod 2 (10.1.1.6)
                       → Pod 3 (10.1.1.7)
```

Service dùng **Selector** để tìm Pod:

```yaml
Service selector: app=web
→ Tìm tất cả Pod có label app=web
→ Gửi traffic đến các Pod đó
```

### 6.3. Các Loại Service

#### 🔵 ClusterIP (Default)

**Đặc điểm:**

- Service chỉ truy cập **nội bộ cluster**
- Có IP internal (VD: 10.0.0.100)

**Khi nào dùng:**

- Communication giữa các service trong cluster
- Backend API không cần expose ra ngoài

**Ví dụ:**

```
Frontend Pod → ClusterIP Service (backend-service)
              → Backend Pod 1
              → Backend Pod 2
```

#### 🟢 NodePort

**Đặc điểm:**

- Expose service ra **ngoài cluster**
- Mở 1 port (30000-32767) trên **MỌI Node**
- Truy cập: `http://<NodeIP>:<NodePort>`

**Cách hoạt động:**

```
Internet → Node 1:30080 ─┐
       → Node 2:30080 ─┼→ Service → Pod 1
       → Node 3:30080 ─┘           → Pod 2
```

**Khi nào dùng:**

- Development/Testing
- Cluster nhỏ
- Không có LoadBalancer

**Nhược điểm:**

- Phải nhớ Node IP và port
- Không có SSL termination

#### 🟡 LoadBalancer

**Đặc điểm:**

- Tự động tạo **External Load Balancer** (cloud provider)
- Có IP public
- Dành cho **production**

**Hoạt động:**

```
Internet → Cloud Load Balancer (IP: 203.0.113.5)
          → NodePort Service
          → Pod 1, Pod 2, Pod 3
```

**Khi nào dùng:**

- Production trên cloud (AWS, GCP, Azure)
- Cần expose 1 service ra internet

**Nhược điểm:**

- Mỗi Service = 1 Load Balancer = Chi phí
- Nếu có 10 service → 10 Load Balancer khác nhau

#### 🔴 ExternalName

**Đặc điểm:**

- Map service trong cluster đến **tên miền bên ngoài**
- Không có load balancing

**Ví dụ:**

```yaml
Service: database-service → external-db.example.com

Pod gọi: database-service
→ K8s resolve thành: external-db.example.com
```

**Khi nào dùng:**

- Database external
- API bên ngoài cluster

### 6.4. Ingress ⭐

**Vấn đề:**

- 10 services → LoadBalancer → 10 IP public, 10 Load Balancer (tốn kém!)
- Cần routing thông minh (path-based, host-based)
- Cần SSL/TLS termination

**Ingress là gì:**

- **HTTP/HTTPS load balancer** cho cluster
- **1 IP public** phục vụ nhiều service
- Hỗ trợ routing, SSL, authentication...

**Cách hoạt động:**

```
Internet → Ingress Controller (1 IP: 203.0.113.5)
          │
          ├→ example.com/api    → API Service → API Pods
          ├→ example.com/web    → Web Service → Web Pods
          └→ shop.example.com   → Shop Service → Shop Pods
```

**Ví dụ routing:**

```yaml
Rules:
- Host: example.com
  Paths:
    - /api → backend-service
    - /    → frontend-service

- Host: blog.example.com
  Path: / → blog-service
```

**Khi nào dùng Ingress:**

- ✅ Nhiều service cần expose
- ✅ Cần routing phức tạp
- ✅ Cần SSL/TLS
- ✅ Production environment

**Khi nào dùng LoadBalancer:**

- ✅ 1 service đơn giản
- ✅ Non-HTTP protocol (TCP/UDP)

**Ingress Controller:**

- Ingress chỉ là "rules" (cấu hình)
- Cần **Ingress Controller** để thực thi
- Popular: NGINX Ingress, Traefik, HAProxy, Istio

---

## 7. Configuration & Security

### 7.1. ConfigMap

**Vấn đề:**

- Hard-code config trong code → Phải rebuild image khi đổi config
- Môi trường khác nhau (dev/prod) → Cần config khác nhau

**ConfigMap là gì:**

- Lưu trữ **configuration data** dạng key-value
- Tách biệt config khỏi container image
- Không mã hóa (dùng cho data không nhạy cảm)

**Ví dụ:**

```yaml
ConfigMap: app-config
Data:
  database_url: "mysql://db.example.com:3306"
  log_level: "info"
  max_connections: "100"
```

**Cách dùng:**

1. **Environment variables:**

   ```
   Pod → Đọc ConfigMap → Set env var DATABASE_URL
   ```

2. **Mount as files:**

   ```
   ConfigMap → Mount vào /etc/config/app.conf
   Pod đọc file config
   ```

**Khi nào dùng:**

- ✅ Database URLs
- ✅ Feature flags
- ✅ Log levels
- ✅ Application settings
- ❌ Passwords, API keys (dùng Secret)

**Ví dụ thực tế:**

```
ConfigMap: nginx-config
Data:
  nginx.conf: |
    server {
      listen 80;
      server_name example.com;
      ...
    }

Pod mount ConfigMap → /etc/nginx/nginx.conf
```

### 7.2. Secret

**Là gì:** Giống ConfigMap nhưng cho **sensitive data** (mật khẩu, token...)

**Đặc điểm:**

- Encode base64 (không phải mã hóa thực sự!)
- Được lưu encrypted trong etcd (nếu cấu hình)
- Access control chặt chẽ hơn

**Ví dụ:**

```yaml
Secret: db-credentials
Data:
  username: YWRtaW4=        # admin (base64)
  password: cGFzc3dvcmQxMjM= # password123 (base64)
```

**Cách dùng:**

1. **Environment variables:**

   ```
   Pod → DB_USERNAME, DB_PASSWORD từ Secret
   ```

2. **Mount as files:**

   ```
   Secret → Mount vào /etc/secrets/db-password
   App đọc file
   ```

**Types of Secret:**

- `Opaque`: Generic secret
- `kubernetes.io/dockerconfigjson`: Docker registry credentials
- `kubernetes.io/tls`: TLS certificate

**Best Practices:**

- ⚠️ Base64 không phải encryption, dễ decode
- ✅ Enable encryption at rest trong etcd
- ✅ Dùng external secret management (Vault, AWS Secrets Manager)
- ✅ Restrict RBAC access
- ✅ Không commit Secret vào Git!

**Ví dụ thực tế:**

```
Secret: tls-certificate
Data:
  tls.crt: <certificate>
  tls.key: <private key>

Ingress dùng Secret này cho HTTPS
```

---

## 8. Storage Cơ Bản

### 8.1. Vấn Đề Cần Giải Quyết

**Container/Pod mặc định:**

- Data trong container là **ephemeral** (tạm thời)
- Pod restart → Mất hết data
- Không chia sẻ data giữa các Pod

**Cần storage khi:**

- Database (data phải persistent)
- Upload files (ảnh, document)
- Logs cần giữ lâu dài
- Share data giữa nhiều Pod

### 8.2. Volume

**Là gì:** Mount storage vào Pod

**Đặc điểm:**

- Volume có lifecycle **gắn với Pod**
- Pod mất → Volume mất (tùy type)

**Các loại Volume:**

#### 1. emptyDir

- Tạo thư mục rỗng khi Pod start
- Pod mất → Volume mất
- **Dùng khi:** Temporary cache, scratch space

```yaml
Volume: cache-volume (emptyDir)
→ Container 1 ghi file vào /cache
→ Container 2 đọc từ /cache
→ Pod restart → Mất hết
```

#### 2. hostPath

- Mount thư mục trên **Node** vào Pod
- Pod mất → Data còn trên Node
- **Cảnh báo:** Pod schedule sang Node khác → Không thấy data

**Dùng khi:**

- DaemonSet cần access node filesystem
- Testing local

#### 3. Cloud storage (AWS EBS, GCE PD, Azure Disk)

- Mount cloud disk vào Pod
- Data persistent dù Pod mất

**Vấn đề của Volume:**

- Config quá chi tiết (path, size, type...)
- Developer phải biết infrastructure

→ **Giải pháp:** PersistentVolume & PVC

### 8.3. PersistentVolume (PV)

**Là gì:** "Kho lưu trữ" được admin chuẩn bị sẵn

**Đặc điểm:**

- Tài nguyên **cluster-level** (không thuộc namespace)
- Lifecycle **độc lập** với Pod
- Admin tạo trước

**Ví dụ:**

```yaml
PersistentVolume: pv-database
Type: AWS EBS
Size: 100GB
Access: ReadWriteOnce
```

**Access Modes:**

- **ReadWriteOnce (RWO):** 1 Node mount read/write
- **ReadOnlyMany (ROX):** Nhiều Node mount read-only
- **ReadWriteMany (RWX):** Nhiều Node mount read/write

**Reclaim Policy:**

- **Retain:** Giữ data khi PVC xóa (phải xóa manual)
- **Delete:** Xóa data khi PVC xóa
- **Recycle:** Xóa data, PV tái sử dụng (deprecated)

### 8.4. PersistentVolumeClaim (PVC)

**Là gì:** "Đơn xin cấp storage" từ developer

**Đặc điểm:**

- Developer tạo PVC: "Tôi cần 50GB storage"
- K8s tự động **bind** PVC với PV phù hợp
- Pod mount PVC (không mount PV trực tiếp)

**Quy trình:**

```
1. Admin tạo PV: 100GB available

2. Developer tạo PVC: "Cần 50GB, RWO"

3. K8s bind: PVC → PV (nếu có PV phù hợp)

4. Pod mount PVC

5. Pod ghi data vào volume

6. Pod mất → Data vẫn còn trong PV
```

**Ví dụ thực tế:**

```yaml
# Admin tạo PV
PersistentVolume: pv-1
  Size: 100GB
  Type: AWS EBS

# Developer tạo PVC
PersistentVolumeClaim: mysql-pvc
  Request: 50GB, RWO

# K8s bind: mysql-pvc ←→ pv-1

# Pod mount PVC
Pod: mysql
  Volume: mysql-pvc → /var/lib/mysql
```

**Dynamic Provisioning:**

- Không cần admin tạo PV trước
- Developer tạo PVC → K8s tự động tạo PV
- Cần **StorageClass**

```yaml
StorageClass: fast-ssd
  Type: AWS EBS gp3

PVC request StorageClass: fast-ssd
→ K8s tự động tạo EBS volume
```

**Khi nào dùng:**

- ✅ Database (MySQL, PostgreSQL, MongoDB)
- ✅ File storage (upload files)
- ✅ StatefulSet
- ❌ Cache tạm (dùng emptyDir)

---

## 9. High Availability & Scaling

### 9.1. Self-Healing (Tự Phục Hồi)

**Là gì:** K8s tự động phát hiện và sửa lỗi

**Các tình huống:**

#### 1. Pod Crash

```
Desired State: 3 Pod running
Current State: 2 Pod (1 Pod crashed)
→ Controller Manager phát hiện
→ Tạo Pod mới
→ Quay về: 3 Pod running
```

#### 2. Node Down

```
Node 2 mất kết nối (hardware failure)
→ Node Controller đánh dấu Node "NotReady"
→ Sau timeout: Evict tất cả Pod trên Node 2
→ Scheduler đặt lịch Pod lên Node khác
```

#### 3. Health Check Fail

```
Pod đang chạy nhưng app bị deadlock
→ Liveness Probe fail
→ kubelet restart container
```

**Health Checks:**

#### Liveness Probe

- **Mục đích:** Kiểm tra container có "sống" không
- **Fail:** Restart container
- **Ví dụ:** HTTP GET /healthz mỗi 10s

#### Readiness Probe

- **Mục đích:** Container có sẵn sàng nhận traffic không
- **Fail:** Gỡ Pod khỏi Service (không nhận request)
- **Ví dụ:** App đang khởi động, load data → Chưa ready

#### Startup Probe

- **Mục đích:** App khởi động chậm (VD: 60s)
- **Fail:** Restart container
- **Ví dụ:** Java app startup lâu

**Ví dụ thực tế:**

```
Web app startup:
1. Container start
2. Startup Probe: Đợi app khởi động (60s timeout)
3. App ready → Startup Probe pass
4. Readiness Probe: Check app ready nhận traffic
5. Ready → Service gửi traffic đến Pod
6. Liveness Probe: Liên tục check app còn hoạt động
```

### 9.2. Scaling Trong Kubernetes

#### Manual Scaling

```
kubectl scale deployment web --replicas=5

Deployment: web
  Replicas: 3 → 5
  → Tạo thêm 2 Pod
```

#### Horizontal Pod Autoscaler (HPA) ⭐

**Là gì:** Tự động scale số lượng Pod dựa trên metrics

**Hoạt động:**

```
1. HPA giám sát metrics (CPU, Memory, custom metrics)
2. CPU > 80% → Scale up (thêm Pod)
3. CPU < 30% → Scale down (giảm Pod)
```

**Ví dụ cấu hình:**

```yaml
HPA: web-hpa
Target: Deployment web
Min replicas: 2
Max replicas: 10
Target CPU: 70%

Tình huống:
- CPU hiện tại: 85% (cao hơn target 70%)
→ HPA tăng từ 3 Pod lên 5 Pod
→ CPU giảm xuống 65%
→ Stable

- Traffic giảm, CPU: 20%
→ HPA giảm từ 5 Pod xuống 3 Pod
```

**Metrics có thể dùng:**

- CPU utilization (%)
- Memory utilization (%)
- Custom metrics: Requests per second, Queue length, Response time...

**Ví dụ thực tế:**

**E-commerce site:**

```
Giờ thấp điểm (2h sáng): 2 Pod (CPU 20%)
Giờ cao điểm (12h trưa): 20 Pod (CPU 70%)
Flash sale: 50 Pod (CPU 75%)

HPA tự động scale
```

**Cooldown Period:**

- Scale up: Nhanh (default 3 phút)
- Scale down: Chậm (default 5 phút)
- Tránh "flapping" (scale lên xuống liên tục)

#### Vertical Pod Autoscaler (VPA)

**Là gì:** Tự động điều chỉnh CPU/Memory **request** của Pod

**Khác HPA:**

- HPA: Tăng **số lượng Pod**
- VPA: Tăng **size của Pod** (CPU/Memory)

**Khi nào dùng:**

- App không thể scale horizontal (VD: database)
- Không biết set CPU/Memory request bao nhiêu

**Lưu ý:** HPA và VPA không nên dùng chung trên cùng metric

#### Cluster Autoscaler

**Là gì:** Tự động tăng/giảm **số lượng Node** trong cluster

**Hoạt động:**

```
Tình huống 1: Scale up Node
- HPA tăng Pod: 10 → 30 Pod
- Node hiện tại: Không đủ CPU/Memory
→ Cluster Autoscaler thêm Node mới
→ Pod được schedule lên Node mới

Tình huống 2: Scale down Node
- Traffic giảm, HPA giảm Pod
- Node ít workload, lãng phí tài nguyên
→ Cluster Autoscaler xóa Node thừa
```

**Dành cho:** Cloud environment (AWS, GCP, Azure)

### 9.3. High Availability Architecture

**Nguyên tắc:**

1. **Multi-replica:** Luôn chạy >= 3 replicas
2. **Anti-affinity:** Các Pod phân tán nhiều Node
3. **Multi-zone:** Node phân tán nhiều Availability Zone
4. **Control Plane HA:** 3 hoặc 5 Master Node

**Ví dụ HA Setup:**

```
Production Cluster:
- 3 Master Node (3 AZ khác nhau)
- Worker Node:
  - Zone A: 3 Node
  - Zone B: 3 Node
  - Zone C: 3 Node

Deployment: web
- Replicas: 6
- Anti-affinity: Mỗi Zone có 2 Pod
- Nếu Zone A down → Zone B, C vẫn phục vụ
```

---

## 📝 Tóm Tắt Kiến Thức Cốt Lõi

### Kiến trúc

- **Control Plane:** API Server, etcd, Scheduler, Controller Manager
- **Worker Node:** kubelet, kube-proxy, Container Runtime

### Concepts

- **Pod:** Đơn vị nhỏ nhất, chứa container
- **Namespace:** Phân chia resources
- **Label & Selector:** Tổ chức và query objects

### Workloads

- **Deployment:** Stateless apps, rolling update
- **StatefulSet:** Database, distributed systems
- **DaemonSet:** 1 Pod/Node (monitoring, logging)
- **Job/CronJob:** Batch processing, scheduled tasks

### Networking

- **Service:** ClusterIP (internal), NodePort (expose), LoadBalancer (cloud)
- **Ingress:** HTTP/HTTPS router, 1 IP cho nhiều service

### Configuration

- **ConfigMap:** Non-sensitive config
- **Secret:** Passwords, tokens

### Storage

- **PV:** Admin tạo storage
- **PVC:** Developer yêu cầu storage
- **Dynamic Provisioning:** Tự động tạo storage

### High Availability

- **Self-healing:** Tự động restart, replace Pod
- **HPA:** Auto-scale theo metrics
- **Health checks:** Liveness, Readiness, Startup Probe

---

## 🎯 Lộ Trình Học Tiếp

1. **Hands-on Practice:**
   - Cài Minikube (K8s local)
   - Tạo Deployment, Service đơn giản
   - Thực hành kubectl commands

2. **YAML Configuration:**
   - Học cú pháp YAML
   - Viết manifest files
   - Apply/Update resources

3. **Advanced Topics:**
   - RBAC (Role-Based Access Control)
   - Network Policies
   - Resource Quotas & Limits
   - Helm (Package manager)

4. **Production Skills:**
   - Monitoring (Prometheus, Grafana)
   - Logging (ELK, Loki)
   - CI/CD với K8s
   - Security best practices

---

## 💡 Tips Học Kubernetes

1. **Thực hành nhiều:** Lý thuyết + Hands-on = Hiểu sâu
2. **Dùng kubectl explain:** `kubectl explain pod.spec` → Xem docs ngay CLI
3. **Đọc manifest của project thực tế:** Xem người ta config như thế nào
4. **Làm labs:** Katacoda, Play with Kubernetes
5. **Không học hết một lúc:** Kubernetes rộng, học dần từng phần

**Chúc bạn học tốt! 🚀**
