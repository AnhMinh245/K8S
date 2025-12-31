# 1.2. So Sánh Kubernetes vs Docker

> Hiểu rõ sự khác biệt và khi nào nên dùng cái gì

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Phân biệt Docker và Kubernetes
- ✅ Hiểu Docker làm gì, K8s làm gì
- ✅ Biết khi nào dùng Docker, khi nào dùng K8s
- ✅ Hiểu mối quan hệ Docker - Kubernetes

---

## 🤔 Câu Hỏi Thường Gặp

**"Kubernetes thay thế Docker à?"**  
→ **KHÔNG!** Docker và Kubernetes làm việc khác nhau.

**"Dùng Kubernetes thì không cần Docker?"**  
→ **SAI!** Kubernetes dùng Docker (hoặc container runtime khác).

**"Vậy khác nhau chỗ nào?"**  
→ Đọc tiếp! 👇

---

## 📦 Docker Là Gì?

### Định Nghĩa

**Docker** là nền tảng để **đóng gói và chạy ứng dụng trong containers**.

### Giải Thích Bằng Ví Dụ

**Docker giống như một "thùng container vận chuyển":**

```
🏭 Application = Hàng hóa
📦 Docker Container = Thùng container
🚢 Docker Engine = Cần cẩu (để chạy container)

Docker làm gì:
✓ Đóng gói app + dependencies vào container
✓ Chạy container trên máy tính
✓ Đảm bảo app chạy giống nhau mọi nơi
```

### Docker Giải Quyết Vấn Đề Gì?

**Vấn đề: "It works on my machine!" (Trên máy tôi chạy được mà!)**

```
Developer: "App chạy ngon lành trên máy tôi!"
   ↓
Deploy lên server
   ↓
Server: "Lỗi! Thiếu dependencies!"

Nguyên nhân:
❌ Python version khác nhau
❌ OS khác nhau (Windows vs Linux)
❌ Dependencies thiếu/khác version
❌ Environment variables khác
```

**Giải pháp: Docker Container**

```
Docker Container bao gồm:
├── Application code
├── Runtime (Node.js, Python, etc.)
├── System libraries
├── Dependencies
└── Configuration

→ Chạy giống nhau ở EVERYWHERE!
   ✓ Laptop developer
   ✓ Testing server
   ✓ Production server
   ✓ Cloud (AWS, GCP, Azure)
```

---

## ☸️ Kubernetes Là Gì?

### Định Nghĩa

**Kubernetes** là nền tảng để **quản lý và điều phối nhiều containers**.

### Giải Thích Bằng Ví Dụ

**Kubernetes giống như "công ty vận tải logistics":**

```
📦 Docker Container = Thùng container (1 cái)
🏭 Kubernetes = Công ty logistics quản lý 1000 containers

Kubernetes làm gì:
✓ Quyết định container nào đi tàu nào (scheduling)
✓ Theo dõi containers (monitoring)
✓ Thay thế containers hỏng (self-healing)
✓ Tăng/giảm containers theo nhu cầu (scaling)
✓ Điều phối giữa nhiều servers (orchestration)
```

### Kubernetes Giải Quyết Vấn Đề Gì?

**Vấn đề: Quản lý 100 containers thủ công = Địa ngục!**

```
Bạn có:
├── 20 servers
├── 100 containers
└── Manual management

Vấn đề:
❌ Container 37 chết → không ai biết cho đến khi user complain
❌ Server 5 chết → 10 containers biến mất
❌ Traffic tăng → phải start containers thủ công
❌ Deploy version mới → phải update từng container
❌ Không biết container nào đang chạy ở đâu
```

**Giải pháp: Kubernetes tự động hóa tất cả!**

---

## 🔄 Mối Quan Hệ: Docker vs Kubernetes

### Không Phải "Này hay Kia", Mà Là "Cùng Nhau"!

```
┌─────────────────────────────────────────┐
│         KUBERNETES CLUSTER              │
├─────────────────────────────────────────┤
│                                         │
│  Node 1:                                │
│  ├─ Docker Engine                       │
│  │  ├─ Container 1 (webapp)             │
│  │  ├─ Container 2 (api)                │
│  │  └─ Container 3 (worker)             │
│                                         │
│  Node 2:                                │
│  ├─ Docker Engine                       │
│  │  ├─ Container 4 (webapp)             │
│  │  └─ Container 5 (api)                │
│                                         │
│  Node 3:                                │
│  ├─ Docker Engine                       │
│  │  └─ Container 6 (database)           │
│                                         │
└─────────────────────────────────────────┘

Docker: Chạy containers
Kubernetes: Quản lý containers trên nhiều servers
```

### Ví Dụ Thực Tế

**Restaurant Analogy:**

```
DOCKER = Đầu bếp (Chef)
├── Biết nấu món ăn
├── Có công cụ nấu nướng
└── Làm được 1 món tại 1 thời điểm

KUBERNETES = Quản lý nhà hàng (Restaurant Manager)
├── Quản lý nhiều đầu bếp
├── Phân công việc cho đầu bếp
├── Thuê thêm đầu bếp khi đông khách
├── Thay đầu bếp bị ốm
└── Đảm bảo nhà hàng hoạt động trơn tru

→ Cần CẢ HAI để nhà hàng thành công!
```

---

## 📊 So Sánh Chi Tiết

### Docker vs Kubernetes

| Đặc Điểm | Docker | Kubernetes |
|----------|--------|------------|
| **Mục đích** | Chạy containers | Quản lý containers |
| **Scope** | 1 máy | Nhiều máy (cluster) |
| **Commands** | `docker run`, `docker stop` | `kubectl create`, `kubectl scale` |
| **Phù hợp** | Dev, testing, small apps | Production, large scale |
| **Learning curve** | Dễ | Khó hơn |
| **Setup** | Đơn giản | Phức tạp hơn |

### Ví Dụ Commands

**DOCKER (Chạy 1 container):**
```bash
# Chạy webapp container
docker run -d \
  --name webapp \
  -p 80:8080 \
  --env DB_HOST=localhost \
  myapp:v1

# Chạy thêm 2 containers nữa (phải làm thủ công)
docker run -d --name webapp2 -p 81:8080 myapp:v1
docker run -d --name webapp3 -p 82:8080 myapp:v1

# Container chết? Phải restart thủ công
docker restart webapp
```

**KUBERNETES (Quản lý nhiều containers):**
```bash
# Chạy 3 containers (tự động!)
kubectl create deployment webapp --image=myapp:v1 --replicas=3

# Kubernetes tự động:
# ✓ Chạy 3 containers
# ✓ Phân phối lên 3 nodes
# ✓ Load balance
# ✓ Monitor health
# ✓ Restart nếu chết

# Scale lên 10 containers (1 command!)
kubectl scale deployment webapp --replicas=10

# Container chết? Kubernetes tự restart!
```

---

## 🎭 Kịch Bản Thực Tế

### Scenario 1: Blog Cá Nhân (100 users/day)

**YÊU CẦU:**
```
├── 1 web server
├── 1 database
├── Ít traffic
└── Budget thấp
```

**GIẢI PHÁP: Docker là đủ!**
```bash
# docker-compose.yml
version: '3.8'
services:
  web:
    image: myblog:latest
    ports:
      - "80:3000"
  
  db:
    image: postgres:14
    volumes:
      - db-data:/var/lib/postgresql/data

# Chạy
docker-compose up -d

✓ Đơn giản
✓ Dễ quản lý
✓ Đủ cho nhu cầu
```

**TẠI SAO KHÔNG CẦN KUBERNETES:**
- Chỉ 1 server
- Traffic ít
- Không cần scaling
- Docker đủ đơn giản và rẻ

---

### Scenario 2: E-commerce (10,000 users, Black Friday)

**YÊU CẦU:**
```
├── 20+ servers
├── 100+ containers
├── Traffic không đều (Black Friday spike)
├── Zero downtime requirements
├── Auto scaling
└── High availability
```

**GIẢI PHÁP: CẦN Kubernetes!**
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 10  # Bình thường
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: ecommerce:v2
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi

---
# hpa.yaml (Auto-scaling)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 10
  maxReplicas: 100  # Black Friday scale up!
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**KẾT QUẢ:**
```
Ngày thường:
├── 10 containers
├── Cost: $500/tháng
└── Performance tốt

Black Friday:
├── Tự động scale lên 100 containers
├── Cost: $500 + $200 (cho 1 ngày)
├── Zero downtime
├── Users happy
└── Sau Black Friday: Tự động scale xuống 10
```

**TẠI SAO CẦN KUBERNETES:**
- Nhiều servers
- Cần auto-scaling
- High availability
- Complex deployment requirements
- Docker + manual management không đủ

---

## 🔀 Docker Swarm vs Kubernetes

### Docker Swarm Là Gì?

**Docker Swarm** = Docker's orchestration tool (như Kubernetes)

```
Docker Swarm:
├── Do Docker phát triển
├── Tích hợp native với Docker
├── Đơn giản hơn Kubernetes
└── Ít features hơn Kubernetes
```

### So Sánh

| Feature | Docker Swarm | Kubernetes |
|---------|--------------|------------|
| **Độ phức tạp** | Đơn giản | Phức tạp hơn |
| **Learning curve** | Dễ | Khó |
| **Features** | Cơ bản | Đầy đủ, mạnh mẽ |
| **Community** | Nhỏ hơn | Rất lớn |
| **Ecosystem** | Hạn chế | Phong phú |
| **Production use** | Ít | Rất phổ biến |
| **Cloud support** | Limited | Excellent (GKE, EKS, AKS) |

### Khi Nào Dùng Docker Swarm?

```
✓ Team nhỏ, quen Docker
✓ Cần orchestration đơn giản
✓ Không cần features phức tạp
✓ Muốn setup nhanh
```

### Khi Nào Dùng Kubernetes?

```
✓ Production serious workloads
✓ Cần auto-scaling phức tạp
✓ Multi-cloud deployment
✓ Large team/organization
✓ Cần ecosystem phong phú
✓ Industry standard
```

---

## 💡 Quyết Định: Dùng Gì?

### Decision Tree

```
Bắt đầu Project
    ↓
    ├─→ Development/Testing?
    │   → Docker là đủ!
    │
    ├─→ Small app (< 5 containers)?
    │   → Docker Compose là đủ!
    │
    ├─→ Medium app, 1 server?
    │   → Docker + Docker Compose
    │
    ├─→ Multiple servers?
    │   ↓
    │   ├─→ Simple orchestration?
    │   │   → Docker Swarm
    │   │
    │   └─→ Production, need scaling?
    │       → Kubernetes ✓
    │
    └─→ Enterprise, serious production?
        → Kubernetes ✓ ✓ ✓
```

### Roadmap Học Tập

```
1. Học Docker trước (2-3 tuần)
   ├── Hiểu containers
   ├── Dockerfile
   ├── Docker Compose
   └── Networking basics

2. Sau đó học Kubernetes (1-2 tháng)
   ├── Có nền tảng Docker rồi
   ├── Hiểu why cần orchestration
   └── Apply kiến thức Docker vào K8s
```

---

## 🎓 Kiểm Tra Hiểu Biết

### Câu Hỏi Tự Kiểm Tra

**1. Docker và Kubernetes khác nhau như thế nào?**
<details>
<summary>Xem đáp án</summary>

- **Docker:** Chạy containers trên 1 máy
- **Kubernetes:** Quản lý containers trên nhiều máy
- **Mối quan hệ:** K8s dùng Docker (hoặc container runtime khác) để chạy containers
- **Analogy:** Docker = đầu bếp, Kubernetes = quản lý nhà hàng
</details>

**2. Khi nào Docker là đủ?**
<details>
<summary>Xem đáp án</summary>

- Development/testing
- Small applications
- 1 server
- Ít containers (< 10)
- Không cần auto-scaling
- Blog cá nhân, side projects
</details>

**3. Khi nào cần Kubernetes?**
<details>
<summary>Xem đáp án</summary>

- Production workloads
- Multiple servers
- Nhiều containers (50+)
- Cần auto-scaling
- High availability requirements
- E-commerce, SaaS platforms
</details>

**4. Có thể dùng Kubernetes mà không dùng Docker không?**
<details>
<summary>Xem đáp án</summary>

**Có!** Kubernetes hỗ trợ nhiều container runtimes:
- Docker (phổ biến nhất)
- containerd
- CRI-O

Nhưng vẫn cần hiểu Docker concepts!
</details>

---

## 💪 Bài Tập Thực Hành

### Bài 1: Chọn công cụ phù hợp

**Cho các scenarios sau, chọn Docker hoặc Kubernetes:**

**a) Personal blog, 50 users/day, 1 server**
<details>
<summary>Xem đáp án</summary>

**Docker Compose** là đủ!

```yaml
version: '3.8'
services:
  blog:
    image: myblog:latest
    ports:
      - "80:3000"
  db:
    image: postgres:14
```

Lý do: Đơn giản, ít traffic, 1 server.
</details>

**b) SaaS platform, 10,000 users, need 99.9% uptime**
<details>
<summary>Xem đáp án</summary>

**Kubernetes!**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: saas-app
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
  template:
    spec:
      containers:
      - name: app
        image: saas:v1
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
```

Lý do: High availability, scaling, multiple servers needed.
</details>

**c) Microservices với 20 services, 5 servers**
<details>
<summary>Xem đáp án</summary>

**Kubernetes!**

20 services, 5 servers = Cần orchestration:
- Service discovery
- Load balancing
- Health checks
- Auto-scaling
- Rolling updates

Docker manual management sẽ là nightmare!
</details>

---

### Bài 2: Migrate từ Docker sang Kubernetes

**Bạn có Docker Compose:**
```yaml
version: '3.8'
services:
  web:
    image: webapp:v1
    ports:
      - "80:8080"
    environment:
      - DB_HOST=db
      - DB_PORT=5432
  
  db:
    image: postgres:14
    environment:
      - POSTGRES_PASSWORD=secret123
```

**Convert sang Kubernetes (simplified):**
<details>
<summary>Xem đáp án</summary>

```yaml
# web-deployment.yaml
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
      - name: webapp
        image: webapp:v1
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: postgres-service
        - name: DB_PORT
          value: "5432"

---
# web-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer

---
# db-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password

---
# db-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
```
</details>

---

## 🎯 Key Takeaways

### Ghi Nhớ 5 Điều Quan Trọng

1. **Docker ≠ Kubernetes**
   - Docker: Chạy containers
   - Kubernetes: Quản lý containers
   
2. **Không phải "này hay kia"**
   - Kubernetes DÙNG Docker
   - Cần cả hai!
   
3. **Start với Docker**
   - Học Docker trước
   - Sau đó học Kubernetes
   
4. **Choose based on needs**
   - Small app → Docker
   - Production, scaling → Kubernetes
   
5. **Kubernetes is the future**
   - Industry standard
   - Nhưng phải hiểu Docker trước!

---

## 📚 Thuật Ngữ Cần Nhớ

| Thuật Ngữ | Tiếng Việt | Ý Nghĩa |
|-----------|------------|---------|
| **Container Runtime** | Container Runtime | Phần mềm chạy containers (Docker, containerd) |
| **Orchestration** | Điều phối | Quản lý nhiều containers tự động |
| **Docker Compose** | Docker Compose | Tool để chạy multi-container apps |
| **Docker Swarm** | Docker Swarm | Orchestration tool của Docker |
| **Cluster** | Cluster | Nhóm servers chạy Kubernetes |

---

## 🚀 Tiếp Theo

Bạn đã phân biệt được Docker và Kubernetes!

**Next:** [1.3. Khi Nào Nên Dùng Kubernetes →](./03-when-to-use-k8s.md)

Ở phần tiếp theo, chúng ta sẽ tìm hiểu chi tiết khi nào NÊN và khi nào KHÔNG NÊN dùng Kubernetes.

---

[⬅️ 1.1. Kubernetes Là Gì](./01-what-is-kubernetes.md) | [🏠 Mục Lục Chính](../README.md) | [📂 Phần 1: Introduction](./README.md) | [➡️ 1.3. Khi Nào Dùng K8s](./03-when-to-use-k8s.md)
