# 1.1. Kubernetes Là Gì?

> Hiểu bản chất của Kubernetes và vấn đề nó giải quyết

---

## 🎯 Mục Tiêu

- Hiểu Kubernetes là gì bằng ví dụ thực tế
- Nắm được các vấn đề K8s giải quyết
- Biết các tính năng cốt lõi của K8s

---

## 📖 Định Nghĩa Đơn Giản

**Kubernetes (K8s)** là một **hệ thống orchestration** (điều phối) cho container, giúp bạn:
- Tự động deploy, scale và quản lý container
- Chạy trên nhiều máy chủ (cluster)
- Đảm bảo ứng dụng luôn sẵn sàng (high availability)

**Orchestration** nghĩa là gì? → Tự động hóa việc quản lý vòng đời của containers.

---

## 🏢 Vấn Đề Thực Tế

### Câu Chuyện 1: Cửa Hàng Coffee

Tưởng tượng bạn mở một chuỗi cửa hàng coffee:

#### 🕐 Giai đoạn 1: Cửa hàng nhỏ (1 địa điểm)
- **Nhân viên:** 2 người
- **Quản lý:** Bạn tự quản lý trực tiếp
- **Vấn đề:** Nhân viên ốm? Bạn tự thay thế
- **Tương đương:** Chạy Docker trên 1 máy chủ

✅ **Dễ quản lý, không cần hệ thống phức tạp**

#### 🏬 Giai đoạn 2: Chuỗi lớn (100 cửa hàng)
- **Nhân viên:** 500 người
- **Phân bố:** 100 địa điểm khác nhau
- **Thách thức:**
  - Làm sao biết cửa hàng nào cần thêm nhân viên?
  - Nhân viên ốm ở chi nhánh A → Ai điều thêm người?
  - Giờ cao điểm ở chi nhánh B → Cần thêm người nhanh!
  - Update quy trình mới → 100 cửa hàng phải cập nhật đồng bộ
  - Làm sao đảm bảo mọi nơi đều hoạt động 24/7?

❌ **Quản lý thủ công = Không khả thi!**

✅ **Cần hệ thống tự động quản lý** → Đây chính là vai trò của Kubernetes!

---

### Câu Chuyện 2: Ứng Dụng Web Thực Tế

Bạn có một ứng dụng e-commerce chạy trên container:

#### 📊 Các vấn đề gặp phải:

**1. Scaling (Mở rộng)**
```
Thứ 2 - 3h sáng:     100 users online   → Cần 2 containers
Thứ 6 - 12h trưa:    10,000 users       → Cần 20 containers
Black Friday:        100,000 users      → Cần 200 containers?
```
❓ **Làm sao tự động tăng/giảm số container?**

**2. High Availability (Sẵn sàng cao)**
```
Container 1: Running
Container 2: CRASHED ❌
Container 3: Running
```
❓ **Ai phát hiện Container 2 chết và tự động restart?**

**3. Load Balancing (Cân bằng tải)**
```
Request 1 → Container nào?
Request 2 → Container nào?
Request 3 → Container nào?
```
❓ **Làm sao phân phối request đều giữa các container?**

**4. Service Discovery (Tìm kiếm dịch vụ)**
```
Frontend container cần gọi Backend container
Backend IP: 172.17.0.5

Nhưng Backend restart → IP mới: 172.17.0.12
```
❓ **Frontend làm sao biết IP mới của Backend?**

**5. Rolling Updates (Cập nhật không downtime)**
```
Hiện tại: Version 1.0 (5 containers)
Muốn deploy: Version 1.1

Nếu stop hết 5 containers cũ → Start 5 containers mới
→ Downtime 2-3 phút ❌
```
❓ **Làm sao update mà không bị gián đoạn service?**

**6. Multi-server Management (Quản lý nhiều server)**
```
Server 1: 8 GB RAM, 4 CPU cores
Server 2: 16 GB RAM, 8 CPU cores
Server 3: 32 GB RAM, 16 CPU cores
```
❓ **Container nên chạy trên server nào? Ai quyết định?**

**7. Configuration Management**
```
Dev environment:   DB = dev-db.internal
Staging:           DB = staging-db.internal
Production:        DB = prod-db.internal
```
❓ **Làm sao quản lý config khác nhau cho mỗi môi trường?**

---

## ✅ Kubernetes Giải Quyết Như Thế Nào?

### 1. **Auto-Scaling (Tự động mở rộng)**
```
Bạn: "Tôi muốn CPU < 70%, tự động scale từ 2-20 containers"
K8s: "OK! Tôi sẽ giám sát và tự động tăng/giảm"

Khi CPU = 85% → K8s tự động tạo thêm container
Khi CPU = 30% → K8s tự động xóa bớt container
```

### 2. **Self-Healing (Tự phục hồi)**
```
Container crashed ❌
→ K8s phát hiện sau 5 giây
→ K8s tự động restart container mới
→ Service tiếp tục hoạt động ✅

Toàn bộ server chết ❌
→ K8s di chuyển tất cả containers sang server khác
→ Không cần can thiệp thủ công ✅
```

### 3. **Load Balancing (Tự động cân bằng tải)**
```
5 Backend containers đang chạy
→ K8s tự động phân phối request đều
→ Built-in load balancer, không cần nginx/HAProxy
```

### 4. **Service Discovery (Tự động tìm dịch vụ)**
```
Backend container có IP thay đổi
→ Frontend không cần biết IP cụ thể
→ Gọi "backend-service" → K8s tự động route
```

### 5. **Rolling Updates & Rollback**
```
Deploy Version 1.1:
  K8s: Tạo 1 container v1.1
       Chờ container ready
       Xóa 1 container v1.0
       Lặp lại cho đến khi hết
  
  → Zero downtime ✅

Nếu Version 1.1 có bug:
  kubectl rollback
  → K8s tự động quay về v1.0 trong vài giây
```

### 6. **Intelligent Scheduling (Lập lịch thông minh)**
```
Container X cần 2GB RAM
→ K8s scan tất cả servers
→ Tìm server có đủ 2GB RAM trống
→ Tự động đặt container lên server đó
→ Cân bằng workload giữa các servers
```

### 7. **Configuration & Secret Management**
```
K8s quản lý:
- ConfigMap: Database URLs, settings, feature flags
- Secret: Passwords, API keys, certificates

Môi trường khác nhau? Chỉ cần thay ConfigMap
→ Không cần rebuild Docker image
```

---

## 🔧 Tính Năng Cốt Lõi Của Kubernetes

### 1. **Container Orchestration**
Quản lý vòng đời của containers: start, stop, restart, migrate

### 2. **Declarative Configuration**
```yaml
Bạn khai báo "Desired State" (trạng thái mong muốn):
  "Tôi muốn 3 replicas của container web"

K8s đảm bảo "Current State" = "Desired State":
  - Nếu có 2 containers → Tạo thêm 1
  - Nếu có 4 containers → Xóa 1
  - Liên tục giám sát và điều chỉnh
```

### 3. **Self-Healing**
Tự động phát hiện và sửa lỗi:
- Container crash → Restart
- Node down → Migrate containers
- Health check fail → Replace container

### 4. **Horizontal Scaling**
Tăng/giảm số lượng containers dễ dàng:
```bash
# Scale manual
kubectl scale deployment web --replicas=10

# Auto-scale
HorizontalPodAutoscaler: Min=2, Max=50, CPU=70%
```

### 5. **Service Discovery & Load Balancing**
- DNS tự động cho mọi service
- Load balancing built-in
- IP ổn định cho services

### 6. **Storage Orchestration**
- Tự động mount storage (local, cloud, network storage)
- Quản lý persistent data
- Volume lifecycle management

### 7. **Automated Rollouts & Rollbacks**
- Update ứng dụng không downtime
- Rollback nhanh khi có vấn đề
- Update strategies: Rolling, Blue-Green, Canary

### 8. **Secret & Configuration Management**
- Tách biệt config khỏi code
- Quản lý sensitive data (passwords, tokens)
- Update config không rebuild image

### 9. **Multi-Tenancy**
- Isolate workloads với Namespaces
- Resource quotas per team/project
- RBAC (Role-Based Access Control)

---

## 📊 So Sánh: Trước & Sau Kubernetes

| Tình huống | Không có K8s | Có K8s |
|------------|--------------|---------|
| **Container chết** | SSH vào server, restart thủ công | Tự động restart trong vài giây |
| **Traffic tăng đột biến** | Thêm container thủ công, cấu hình LB | Tự động scale, LB tự động |
| **Update ứng dụng** | Downtime 5-10 phút | Zero downtime |
| **Server chết** | Panic! Tạo server mới, deploy lại | Tự động migrate sang server khác |
| **Quản lý 10 services** | 10 bộ config, script deploy khác nhau | Cấu hình tập trung, declarative |
| **Multi-environment** | Maintain nhiều bộ script | Chỉ thay ConfigMap |

---

## 🔍 Kubernetes Không Phải Là Gì?

**❌ K8s KHÔNG phải:**
- **Container runtime:** K8s không run containers, nó dùng Docker/containerd
- **CI/CD tool:** K8s deploy apps, nhưng không build code
- **PaaS (Platform as a Service):** K8s là infrastructure layer, không phải app platform
- **Magic solution:** Vẫn cần hiểu networking, storage, security...

**✅ K8s LÀ:**
- **Orchestration platform:** Quản lý containers
- **Cluster management:** Quản lý nhiều servers như một hệ thống
- **Automation engine:** Tự động hóa operations

---

## 🏛️ Lịch Sử & Nguồn Gốc

### Nguồn Gốc
- **Năm 2014:** Google open-source Kubernetes
- **Dựa trên:** Borg và Omega (hệ thống nội bộ của Google)
- **Tên gọi:** "Kubernetes" (Hy Lạp) = "Người lái tàu" 🚢
- **K8s:** Viết tắt (K + 8 chữ cái + s = Kubernetes)

### Timeline
- **2014:** Google release K8s
- **2015:** v1.0, donate cho CNCF (Cloud Native Computing Foundation)
- **2017:** Docker Inc. thêm K8s support vào Docker
- **2018:** K8s thành industry standard
- **2020+:** Đa số cloud providers hỗ trợ K8s (AWS EKS, GCP GKE, Azure AKS)

### Tại Sao K8s Thành Công?
1. **Open source:** Không bị lock-in vendor
2. **Cloud-agnostic:** Chạy được mọi nơi (AWS, GCP, Azure, on-premise)
3. **Community lớn:** Hàng nghìn contributors
4. **Extensible:** Plugin architecture, dễ mở rộng
5. **Industry standard:** Đa số công ty lớn đang dùng

---

## 🎓 Khái Niệm Cần Nhớ

### Orchestration
Tự động hóa deployment, scaling, management của containers

### Cluster
Nhóm các servers (nodes) chạy chung, quản lý bởi K8s

### Declarative
Khai báo "muốn gì" thay vì "làm thế nào":
```yaml
# Declarative (K8s style)
"Tôi muốn 3 containers web chạy"
→ K8s tự xử lý

# Imperative (traditional style)
"Tạo container 1, đợi nó start, tạo container 2..."
```

### Desired State
Trạng thái mong muốn bạn khai báo → K8s đảm bảo trạng thái này luôn đúng

---

## 📈 Ai Đang Dùng Kubernetes?

### Tech Giants
- **Google:** Chạy mọi thứ trên K8s (Gmail, YouTube, Search...)
- **Microsoft:** Azure services
- **Amazon:** Internal services (dù có AWS ECS)
- **Netflix, Spotify, Airbnb, Uber:** Production workloads

### Industries
- **E-commerce:** Shopify, eBay
- **Finance:** ING Bank, Capital One
- **Gaming:** Pokémon GO
- **Media:** New York Times

### Startups
Hầu hết startups mới đều chọn K8s cho infrastructure

---

## 💡 Key Takeaways

1. **K8s = Orchestration tool** cho containers, giống "hệ điều hành cho cluster"
2. **Giải quyết vấn đề thực tế:** Scaling, HA, deployment, management
3. **Automation:** Giảm manual operations, tăng reliability
4. **Cloud-native standard:** Industry standard cho container workloads
5. **Không đơn giản:** Cần học và practice, nhưng đáng giá

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. Kubernetes giải quyết vấn đề gì so với chạy Docker đơn thuần?
2. Self-healing trong K8s hoạt động như thế nào?
3. Declarative configuration khác gì với imperative?
4. Tại sao K8s được gọi là "orchestration" tool?
5. K8s có thể thay thế Docker không?

---

## 🚀 Tiếp Theo

Bây giờ bạn đã hiểu K8s là gì và tại sao cần nó.

👉 Tiếp theo: [1.2. So Sánh Kubernetes vs Docker](./02-k8s-vs-docker.md)

Chúng ta sẽ đi sâu vào sự khác biệt giữa K8s và Docker, biết khi nào dùng cái gì.

---

[⬅️ Về Phần 1: Introduction](./README.md) | [⬆️ Về Mục Lục Chính](../README.md)

