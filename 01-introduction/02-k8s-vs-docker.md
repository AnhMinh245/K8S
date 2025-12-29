# 1.2. So Sánh Kubernetes vs Docker

> Hiểu rõ sự khác biệt, biết khi nào dùng K8s, khi nào dùng Docker

---

## 🎯 Mục Tiêu

- Hiểu Docker giải quyết vấn đề gì
- Phân biệt rõ K8s và Docker
- Biết khi nào nên dùng cái gì

---

## 🐳 Docker Giải Quyết Vấn Đề Gì?

### Vấn Đề: "Works on My Machine"

**Trước Docker:**
```
Developer A (MacOS):
  Python 3.8, MySQL 5.7, Redis 4.0
  → Code chạy OK ✅

Developer B (Windows):
  Python 3.9, MySQL 8.0, Redis 5.0
  → Code báo lỗi ❌

Production Server (Linux):
  Python 3.7, MySQL 5.6, Redis 3.2
  → Toàn bộ lỗi ❌❌❌
```

**Giải pháp Docker:**
```
Dockerfile định nghĩa môi trường:
  - Python 3.8
  - MySQL 5.7
  - Redis 4.0
  - Tất cả dependencies

→ Docker image chứa TOÀN BỘ môi trường
→ Chạy giống hệt nhau trên MacOS, Windows, Linux ✅
```

### Docker Là Gì?

**Docker** là nền tảng để:
1. **Package application** vào containers (đóng gói)
2. **Run containers** trên một máy chủ (chạy)
3. **Isolate** containers với nhau (cách ly)

**Ví dụ thực tế:**
```bash
# Build image
docker build -t my-app:1.0 .

# Run container
docker run -p 8080:80 my-app:1.0

# Stop container
docker stop my-app
```

---

## ⚖️ Kubernetes vs Docker: Khác Biệt Cơ Bản

### Nhầm Lẫn Phổ Biến

❌ **"Kubernetes vs Docker" không phải so sánh đúng!**

✅ **Đúng ra:**
- **Docker** = Container runtime + image format
- **Kubernetes** = Container orchestration platform

**Chúng bổ trợ cho nhau, không thay thế!**

```
┌─────────────────────────────────┐
│        Kubernetes               │  ← Orchestration layer
│  (Quản lý, deploy, scale...)    │
└─────────────────────────────────┘
            ↓ uses
┌─────────────────────────────────┐
│   Container Runtime             │  ← Container layer
│   (Docker, containerd, CRI-O)   │
└─────────────────────────────────┘
```

### So Sánh Chi Tiết

| Tiêu chí | Docker (Standalone) | Kubernetes |
|----------|---------------------|------------|
| **Mục đích chính** | Chạy containers trên 1 máy | Quản lý containers trên nhiều máy |
| **Scope** | Single host | Multi-host cluster |
| **Scaling** | Thủ công: `docker run` thêm container | Tự động: HPA scale theo CPU/memory |
| **Load Balancing** | Cần tool bên ngoài (nginx, HAProxy) | Built-in Service |
| **Self-Healing** | Không có (cần Docker Swarm hoặc script) | Tự động restart, replace containers |
| **Service Discovery** | Thủ công config network | Tự động DNS, service discovery |
| **Rolling Updates** | Thủ công: stop → start từng container | Tự động rolling updates |
| **Configuration** | Env vars, volumes | ConfigMap, Secret |
| **Storage** | Volumes (local hoặc plugins) | PersistentVolumes (nhiều loại storage) |
| **Networking** | Bridge, host, overlay (đơn giản) | CNI plugins (phức tạp, mạnh mẽ) |
| **Monitoring** | Cần tool bên ngoài | Metrics API, integration sẵn |
| **Learning Curve** | Dễ học | Khó học hơn nhiều |
| **Use Case** | Dev/Test, ứng dụng đơn giản | Production, multi-service, enterprise |

---

## 🔄 Docker Swarm vs Kubernetes

**Docker Swarm** = Orchestration tool của Docker (competitor của K8s)

### So Sánh

| Feature | Docker Swarm | Kubernetes |
|---------|--------------|------------|
| **Setup** | Rất đơn giản | Phức tạp |
| **Learning Curve** | Dễ (nếu đã biết Docker) | Khó |
| **Features** | Cơ bản, đủ dùng | Đầy đủ, mạnh mẽ |
| **Community** | Nhỏ | Rất lớn |
| **Ecosystem** | Hạn chế | Phong phú (Helm, Operators...) |
| **Production Usage** | Ít công ty dùng | Industry standard |
| **Auto-scaling** | Cơ bản | Mạnh mẽ (HPA, VPA, Cluster Autoscaler) |
| **Cloud Support** | Hạn chế | Hỗ trợ mọi cloud provider |

### Ví Dụ So Sánh Config

**Docker Swarm:**
```yaml
version: '3'
services:
  web:
    image: nginx:1.20
    replicas: 3
    ports:
      - "80:80"
```

**Kubernetes:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
```

**Nhận xét:**
- Swarm đơn giản hơn nhiều
- K8s verbose hơn nhưng mạnh mẽ và linh hoạt hơn

---

## 📊 Khi Nào Dùng Cái Gì?

### ✅ Dùng Docker Standalone

**Phù hợp khi:**
1. **Development environment**
   - Dev trên laptop cá nhân
   - Test local trước khi deploy

2. **Single server application**
   - Blog cá nhân
   - Website nhỏ
   - Internal tools

3. **Learning/Prototyping**
   - Học containers
   - Proof of concept

4. **Simple workloads**
   - Chạy vài container
   - Không cần HA
   - Traffic ổn định

**Ví dụ thực tế:**
```bash
# Blog WordPress đơn giản
docker-compose up
  - WordPress container
  - MySQL container
  
→ Đủ cho blog cá nhân 1000 visits/day
```

### ✅ Dùng Docker Swarm

**Phù hợp khi:**
1. **Cần orchestration đơn giản**
   - Vài services
   - Scaling cơ bản
   
2. **Team nhỏ, chưa có DevOps**
   - Không đủ resource học K8s
   - Cần solution nhanh

3. **Legacy Docker users**
   - Đã quen Docker
   - Muốn migrate dễ dàng

**Ví dụ:** Startup với 3-5 microservices, 5-10 servers

### ✅ Dùng Kubernetes

**Phù hợp khi:**
1. **Production workloads lớn**
   - Nhiều microservices (10+)
   - High traffic
   - Cần 99.9%+ uptime

2. **Multi-environment**
   - Dev, Staging, Production
   - Multiple teams

3. **Dynamic scaling**
   - Traffic không đều
   - Cần auto-scale
   - Cost optimization

4. **Cloud-native applications**
   - Microservices architecture
   - CI/CD pipeline
   - Modern development practices

5. **Enterprise requirements**
   - Multi-tenancy
   - RBAC, security policies
   - Compliance

**Ví dụ thực tế:**
- E-commerce platform: 50 microservices, 1M requests/day
- SaaS application: Multi-tenant, auto-scaling
- Fintech: High security, compliance, audit logs

---

## 🏢 Case Studies Thực Tế

### Case 1: Startup Giai Đoạn Đầu

**Tình huống:**
- Team: 3 developers
- MVP: Monolith app + database
- Traffic: 100-500 users/day
- Budget: Hạn chế

**Giải pháp:**
```
✅ Docker Compose
  - docker-compose.yml
  - 1 VPS $20/month
  - Deploy = git pull + docker-compose up
  
❌ Không cần K8s (overkill)
```

### Case 2: Startup Scale-Up

**Tình huống:**
- Team: 10 developers
- Architecture: 5 microservices
- Traffic: 10,000 users/day, tăng nhanh
- Cần: Auto-scaling, zero downtime

**Giải pháp:**
```
✅ Kubernetes (managed: EKS, GKE, AKS)
  - Auto-scaling
  - Rolling updates
  - Multi-environment
  
hoặc

✅ Docker Swarm (nếu team chưa sẵn sàng K8s)
  - Đơn giản hơn
  - Đủ dùng cho giai đoạn này
```

### Case 3: Enterprise

**Tình huống:**
- Team: 50+ developers
- Architecture: 30+ microservices
- Traffic: Millions requests/day
- Requirements: HA, security, compliance

**Giải pháp:**
```
✅ Kubernetes (bắt buộc)
  - Multi-cluster setup
  - Service mesh (Istio)
  - GitOps (ArgoCD)
  - Monitoring stack (Prometheus, Grafana)
  
❌ Docker Swarm không đủ mạnh
```

### Case 4: Personal Project

**Tình huống:**
- Side project
- 1 developer
- Low traffic
- Learning purpose

**Giải pháp:**
```
✅ Docker Compose cho production
  - Đơn giản
  - Đủ dùng
  
✅ Minikube/Kind cho học K8s
  - Practice K8s concepts
  - Không deploy production trên Minikube
```

---

## 🔄 Migration Path: Docker → Kubernetes

### Lộ Trình Chuyển Đổi

**Phase 1: Docker Compose (Hiện tại)**
```
docker-compose.yml
  - web service
  - api service
  - database
```

**Phase 2: Prepare for K8s**
```
1. Tách database ra ngoài (managed DB)
2. Externalize configuration (env vars)
3. Health checks
4. Logging tập trung
5. Stateless applications
```

**Phase 3: Kubernetes (Tương lai)**
```
1. Convert docker-compose → K8s manifests
   Tool: Kompose (tự động convert)
   
2. Deploy lên K8s cluster
   - Deployment cho mỗi service
   - Service cho networking
   - ConfigMap cho config
   
3. Incrementally add K8s features
   - Auto-scaling
   - Ingress
   - Monitoring
```

### Tools Hỗ Trợ Migration

**1. Kompose**
```bash
# Convert docker-compose.yml → K8s YAML
kompose convert -f docker-compose.yml

# Output:
# - deployment.yaml
# - service.yaml
# - pvc.yaml
```

**2. Helm Charts**
Tìm Helm chart có sẵn cho ứng dụng phổ biến:
```bash
# Thay vì tự config WordPress
helm install wordpress bitnami/wordpress
```

---

## 💡 Decision Tree: Chọn Công Cụ Nào?

```
Bắt đầu:
│
├─ Bạn đang học containers?
│  └─ YES → Docker (standalone)
│  └─ NO  → Tiếp tục
│
├─ Production application?
│  └─ NO  → Docker Compose
│  └─ YES → Tiếp tục
│
├─ Chỉ 1 server?
│  └─ YES → Docker Compose hoặc Swarm
│  └─ NO  → Tiếp tục
│
├─ < 5 microservices?
│  └─ YES → Docker Swarm (hoặc K8s nếu team có kinh nghiệm)
│  └─ NO  → Tiếp tục
│
├─ Enterprise, nhiều teams?
│  └─ YES → Kubernetes
│  
├─ Cần auto-scaling mạnh mẽ?
│  └─ YES → Kubernetes
│
├─ Multi-cloud hoặc hybrid cloud?
│  └─ YES → Kubernetes
│
└─ Default cho production hiện đại
   → Kubernetes
```

---

## 📈 Xu Hướng Hiện Nay

### Industry Trends

1. **K8s = Standard**
   - Đa số công ty lớn đã chuyển sang K8s
   - Job postings yêu cầu K8s experience
   
2. **Docker vẫn quan trọng**
   - Container format standard
   - K8s vẫn chạy Docker images
   
3. **Docker Swarm declining**
   - Docker Inc. focus vào Docker Desktop
   - Community chuyển sang K8s
   
4. **Managed Kubernetes phổ biến**
   - AWS EKS, GCP GKE, Azure AKS
   - Giảm complexity của K8s

### Cloud Provider Support

| Provider | Docker Support | K8s Support |
|----------|----------------|-------------|
| AWS | ECS (proprietary) | EKS (managed K8s) ⭐ |
| Google Cloud | Cloud Run | GKE (managed K8s) ⭐⭐ |
| Azure | Container Instances | AKS (managed K8s) ⭐ |
| DigitalOcean | Droplets | DOKS (managed K8s) |

**Nhận xét:** Mọi cloud provider đều invest mạnh vào K8s

---

## 🎓 Key Takeaways

1. **Docker ≠ K8s:** Docker là container runtime, K8s là orchestration
2. **Bổ trợ nhau:** K8s dùng Docker (hoặc containerd) để chạy containers
3. **Docker Compose:** Tốt cho dev, small apps
4. **Docker Swarm:** Đơn giản nhưng ít features
5. **Kubernetes:** Phức tạp nhưng mạnh mẽ, industry standard
6. **Choose based on needs:** Không phải lúc nào cũng cần K8s
7. **Learning path:** Docker → Docker Compose → Kubernetes

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. Docker và Kubernetes có thay thế nhau không?
2. Khi nào nên dùng Docker Compose thay vì Kubernetes?
3. So sánh Docker Swarm và Kubernetes?
4. "Works on my machine" là vấn đề gì và Docker giải quyết thế nào?
5. Managed Kubernetes (EKS, GKE) khác gì với self-hosted K8s?

---

## 🚀 Tiếp Theo

Bạn đã hiểu sự khác biệt giữa Docker và Kubernetes.

👉 Tiếp theo: [1.3. Khi Nào Nên Dùng Kubernetes](./03-when-to-use-k8s.md)

Chúng ta sẽ đi sâu vào các use cases cụ thể và decision framework.

---

[⬅️ 1.1. Kubernetes Là Gì?](./01-what-is-kubernetes.md) | [⬆️ Về Phần 1: Introduction](./README.md) | [🏠 Mục Lục Chính](../README.md)


