# 📘 Phần 1: Introduction - Giới Thiệu Kubernetes

> Hiểu Kubernetes từ vấn đề thực tế và biết khi nào nên sử dụng

---

## 🎯 Mục Tiêu Phần Này

Sau khi hoàn thành Phần 1, bạn sẽ:

✅ **Hiểu Kubernetes là gì** và giải quyết vấn đề gì  
✅ **Phân biệt** Docker và Kubernetes  
✅ **Biết khi nào** nên và không nên dùng K8s  
✅ **Có foundation** để học các phần tiếp theo  

---

## 📚 Nội Dung

### [1.1. Kubernetes Là Gì?](./01-what-is-kubernetes.md) ⭐⭐⭐⭐⭐

**Thời gian:** 30-45 phút

**Nội dung:**
- Vấn đề thực tế (Startup phát triển từ 10 → 100,000 users)
- Kubernetes giải quyết như thế nào
- Các tính năng cốt lõi:
  - Container Orchestration
  - Self-Healing
  - Auto-Scaling
  - Service Discovery & Load Balancing
  - Rolling Updates & Rollbacks
  - Configuration Management
- Kiến trúc tổng quan
- So sánh: Trước vs Sau K8s

**Key Points:**
```
✓ Kubernetes = Tự động hóa quản lý containers
✓ Self-healing = Ngủ ngon hơn
✓ Scaling = Tiết kiệm tiền + Performance
✓ Zero downtime deployments
✓ Declarative configuration
```

**Bài Tập:**
- Quiz về workflow self-healing
- So sánh scenarios manual vs K8s
- Identify các tính năng cốt lõi

---

### [1.2. So Sánh Kubernetes vs Docker](./02-k8s-vs-docker.md) ⭐⭐⭐⭐

**Thời gian:** 30-40 phút

**Nội dung:**
- Docker là gì (Container runtime)
- Kubernetes là gì (Container orchestration)
- Mối quan hệ: K8s DÙNG Docker
- So sánh chi tiết:
  - Mục đích
  - Scope (1 máy vs cluster)
  - Commands
  - Use cases
- Docker Swarm vs Kubernetes
- Migration path: Docker → K8s

**Key Points:**
```
✓ Docker ≠ Kubernetes (khác nhau hoàn toàn)
✓ Không phải "này hay kia" mà là "cùng nhau"
✓ Docker: Chạy containers
✓ Kubernetes: Quản lý containers
✓ Start với Docker, sau đó học K8s
```

**Bài Tập:**
- So sánh commands Docker vs kubectl
- Convert Docker Compose → K8s manifests
- Decision making exercises

---

### [1.3. Khi Nào Nên Dùng Kubernetes](./03-when-to-use-k8s.md) ⭐⭐⭐⭐⭐

**Thời gian:** 40-50 phút

**Nội dung:**
- Khi NÊN dùng K8s:
  - Microservices architecture (20+ services)
  - High availability requirements (99.9%+)
  - Dynamic scaling needs
  - Multi-environment management
  - Growing team (10+ developers)
- Khi KHÔNG NÊN dùng K8s:
  - Simple applications
  - Small team without K8s expertise
  - Cost-sensitive projects (< $100/month)
  - MVP / Prototypes
- Decision framework (checklist)
- Migration path (Docker → K8s)
- Alternatives (PaaS, Docker Swarm, Serverless)

**Key Points:**
```
✓ K8s không phải cho everyone
✓ Start simple, scale later
✓ Consider costs (K8s expensive cho small projects)
✓ Team expertise matters
✓ Alternatives exist
```

**Bài Tập:**
- Đánh giá 3 scenarios: nên dùng K8s hay không
- Calculate cost comparison
- Create migration plan

---

## 🎓 Learning Approach

### Cách Học Hiệu Quả

**1. Đọc Tuần Tự**
```
1.1 → 1.2 → 1.3

Mỗi phần xây dựng trên phần trước!
```

**2. Active Reading**
```
✓ Làm quiz sau mỗi section
✓ Vẽ diagrams để hiểu
✓ So sánh với experience của bạn
✓ Note down câu hỏi
```

**3. Thực Hành Ngay**
```
✓ Không cần setup K8s ngay
✓ Hiểu concepts trước
✓ Setup ở Phần 2
```

---

## 💡 Prerequisites

### Trước Khi Bắt Đầu

**Bạn NÊN biết:**
```
✓ Cơ bản về Linux/command line
✓ Khái niệm về containers (Docker basics)
✓ Networking cơ bản (IP, ports)
✓ Web applications architecture
```

**Bạn KHÔNG CẦN biết:**
```
❌ Kubernetes (chúng ta sẽ học!)
❌ Advanced Docker
❌ DevOps tools
❌ Cloud platforms
```

---

## 🗺️ Navigation

### Structure

```
01-introduction/
├── README.md (file này)
├── 01-what-is-kubernetes.md
├── 02-k8s-vs-docker.md
└── 03-when-to-use-k8s.md

Estimated time: 2-3 hours total
```

### Reading Order

```
1. Start: README.md (file này)
   ↓
2. 1.1. Kubernetes Là Gì?
   ↓
3. 1.2. K8s vs Docker
   ↓
4. 1.3. Khi Nào Dùng K8s?
   ↓
5. Next: Phần 2 - Architecture
```

---

## 📊 Self-Assessment

### Checkpoint: Bạn Đã Sẵn Sàng Chưa?

**Trả lời các câu hỏi sau:**

**1. Kubernetes giải quyết vấn đề gì?**
<details>
<summary>Check answer</summary>

- Tự động hóa deploy, scaling, management containers
- Self-healing khi có lỗi
- Load balancing
- Rolling updates zero-downtime
- Service discovery
- Resource management
</details>

**2. Khác nhau Docker vs Kubernetes?**
<details>
<summary>Check answer</summary>

- Docker: Chạy containers trên 1 máy
- Kubernetes: Quản lý containers trên nhiều máy
- Kubernetes DÙNG Docker (hoặc runtime khác)
- Không phải "này hay kia", mà "cùng nhau"
</details>

**3. Khi nào NÊN dùng Kubernetes?**
<details>
<summary>Check answer</summary>

- Microservices (10+ services)
- High availability needed
- Auto-scaling requirements
- Multiple servers
- Growing team
- Budget > $500/month
</details>

**4. Khi nào KHÔNG NÊN dùng Kubernetes?**
<details>
<summary>Check answer</summary>

- Simple apps (1-2 services)
- Small team, no K8s expertise
- Budget < $100/month
- MVP/Prototypes
- < 10 containers
→ Use Docker Compose, PaaS instead
</details>

**5. Self-healing hoạt động như thế nào?**
<details>
<summary>Check answer</summary>

```
1. K8s continuously checks container health
2. Detects unhealthy/crashed container
3. Automatically starts new container
4. Service continues without interruption
5. User doesn't notice any issue
```
</details>

---

## 🎯 Key Takeaways - Phần 1

### 5 Điều Quan Trọng Nhất

**1. Kubernetes = Container Orchestration Platform**
```
Tự động hóa:
├── Deployment
├── Scaling
├── Management
└── của containers trên nhiều servers
```

**2. Docker và Kubernetes KHÁC NHAU**
```
Docker: Chạy containers
Kubernetes: Quản lý containers
→ Cần cả hai!
```

**3. Self-Healing = Game Changer**
```
Container crash → K8s tự restart
Node failure → K8s recreate pods
→ Ngủ ngon, không lo 3 AM phone call!
```

**4. K8s Không Phải Cho Everyone**
```
Small app → Docker Compose
Medium app → PaaS hoặc Docker Swarm  
Large/Enterprise → Kubernetes
→ Choose based on needs!
```

**5. Start Simple, Scale Later**
```
Phase 1: Docker
Phase 2: Evaluate needs
Phase 3: Kubernetes (if needed)
→ Đừng over-engineer!
```

---

## 📚 Thuật Ngữ Đã Học

| Thuật Ngữ | Tiếng Việt | Ý Nghĩa |
|-----------|------------|---------|
| **Kubernetes** | Kubernetes | Nền tảng điều phối containers |
| **Container** | Container | Đóng gói app + dependencies |
| **Orchestration** | Điều phối | Quản lý tự động nhiều containers |
| **Self-Healing** | Tự phục hồi | Tự động thay thế resources lỗi |
| **Scaling** | Mở rộng | Tăng/giảm resources theo nhu cầu |
| **Cluster** | Cluster | Tập hợp servers chạy K8s |
| **Node** | Node | Server/VM trong cluster |
| **Pod** | Pod | Đơn vị nhỏ nhất trong K8s |

---

## ❓ FAQs - Câu Hỏi Thường Gặp

**Q1: Tôi cần biết Docker trước không?**
```
A: NÊN biết Docker basics:
✓ Container là gì
✓ Images, containers
✓ Dockerfile
✓ Basic commands

Nhưng không cần expert!
```

**Q2: Kubernetes có khó học không?**
```
A: Trung bình → Khó
├── Concepts nhiều
├── Learning curve steep
└── Cần thời gian practice

Tips:
✓ Học từ từ, từng phần
✓ Hands-on practice nhiều
✓ Follow tài liệu này tuần tự
✓ 1-2 tháng để proficient
```

**Q3: Chi phí chạy Kubernetes?**
```
A: Depends on deployment:

Managed K8s (Recommended):
├── GKE: $75/month (control plane) + nodes
├── EKS: $75/month + nodes
├── AKS: Free control plane + nodes
└── Nodes: $30-100/node/month

Self-hosted:
├── Control plane: Free
├── Nodes: Server costs only
└── But: High maintenance effort

Minimum: ~$150-200/month
```

**Q4: Dùng K8s cho side project có được không?**
```
A: Tùy mục đích:

Learning: ✅ Yes! (Use Minikube/kind locally)
Production: ❌ No! (Overkill, expensive)

Side project → Use:
✓ Heroku
✓ Railway
✓ Docker Compose
✓ DigitalOcean App Platform
```

**Q5: Kubernetes vs Docker Swarm?**
```
A: 
Docker Swarm:
✓ Simpler
✓ Easier to learn
✗ Fewer features
✗ Smaller ecosystem
✗ Less popular

Kubernetes:
✓ Feature-rich
✓ Huge ecosystem
✓ Industry standard
✗ More complex
✗ Harder to learn

Recommend: Kubernetes (worth investment)
```

---

## 🚀 Tiếp Theo

### Bạn Đã Sẵn Sàng!

Bạn đã hiểu:
- ✅ Kubernetes là gì
- ✅ Tại sao cần K8s
- ✅ Khác nhau Docker vs K8s
- ✅ Khi nào nên/không nên dùng

**Next Step:** [Phần 2: Architecture →](../02-architecture/README.md)

Ở phần tiếp theo, chúng ta sẽ deep dive vào kiến trúc Kubernetes:
- Control Plane components
- Worker Node components
- Communication flow
- Hands-on với Minikube

---

## 📖 Resources Bổ Sung

### Documentation
- [Kubernetes Official Docs](https://kubernetes.io/docs/)
- [Docker Documentation](https://docs.docker.com/)

### Tutorials
- [Kubernetes Tutorial for Beginners](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
- [Interactive Kubernetes Playground](https://www.katacoda.com/courses/kubernetes)

### Books
- "Kubernetes Up & Running" - Kelsey Hightower
- "The Kubernetes Book" - Nigel Poulton

---

[🏠 Mục Lục Chính](../README.md) | [➡️ Phần 2: Architecture](../02-architecture/README.md)
