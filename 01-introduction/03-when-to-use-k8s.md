# 1.3. Khi Nào Nên Dùng Kubernetes

> Decision framework để quyết định có nên dùng K8s hay không

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Biết khi nào **NÊN** dùng Kubernetes  
- ✅ Biết khi nào **KHÔNG NÊN** dùng Kubernetes
- ✅ Đánh giá dự án có cần K8s không
- ✅ Hiểu các giải pháp thay thế

---

## ⚠️ Quan Trọng: Kubernetes KHÔNG Phải Cho Mọi Người!

### Anti-Pattern: "Mọi người dùng nên mình cũng dùng"

**Câu chuyện thực tế:**

```
Startup A (3 developers, 500 users):
├── Nghe Kubernetes hot → quyết định dùng
├── 2 tháng setup
├── 1 tháng học
├── Cost tăng 3x (so với VPS đơn giản)
├── Team overwhelmed
└── Cuối cùng: Quay lại Docker Compose

Kết quả: 
❌ Lãng phí 3 tháng
❌ Lãng phí tiền
❌ Team frustrated
❌ Product development delay
```

**Bài học:** Kubernetes mạnh mẽ nhưng phức tạp. Chỉ dùng KHI THỰC SỰ CẦN!

---

## ✅ Khi Nào NÊN Dùng Kubernetes

### 1. Microservices Architecture

**SCENARIO:**
```
Bạn có:
├── 20+ microservices
├── Mỗi service 3-5 replicas
├── Total: 60-100 containers
└── Chạy trên 10+ servers
```

**TẠI SAO CẦN K8S:**
```
Không có K8s:
❌ Deploy 100 containers thủ công = Nightmare
❌ Service discovery giữa services = Phức tạp
❌ Load balancing = Phải setup riêng
❌ Health checks = Viết scripts riêng
❌ Scaling = Thủ công, chậm
❌ Updates = Rủi ro downtime cao

Với K8s:
✓ Deploy 100 containers = 1 command
✓ Service discovery = Built-in
✓ Load balancing = Tự động
✓ Health checks = Built-in
✓ Auto-scaling = Configure và quên đi
✓ Rolling updates = Zero downtime
```

**VÍ DỤ THỰC TẾ:**
```
E-commerce Platform:
├── frontend-service (5 replicas)
├── auth-service (3 replicas)
├── product-service (5 replicas)
├── cart-service (3 replicas)
├── order-service (5 replicas)
├── payment-service (3 replicas)
├── notification-service (2 replicas)
├── search-service (3 replicas)
└── recommendation-service (3 replicas)

Total: 9 services × average 3.5 replicas = 32 containers
Plus: databases, caches, queues = 50+ containers

→ KUBERNETES LÀ MUST-HAVE!
```

---

### 2. High Availability Requirements

**SCENARIO:**
```
Requirements:
├── 99.9% uptime (43 minutes downtime/month)
├── Zero downtime deployments
├── Auto-recovery from failures
└── Multi-zone/region deployment
```

**TẠI SAO CẦN K8S:**

Kubernetes provides out-of-the-box:

```yaml
# High Availability Setup
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-service
spec:
  replicas: 5  # Multiple replicas
  strategy:
    type: RollingUpdate  # Zero downtime updates
    rollingUpdate:
      maxUnavailable: 1  # Always 4 running
      maxSurge: 2
  
  template:
    spec:
      # Multi-zone deployment
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: critical-service
              topologyKey: topology.kubernetes.io/zone
      
      containers:
      - name: app
        image: critical-service:v1
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          periodSeconds: 5

---
# Auto-scaling based on load
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: critical-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: critical-service
  minReplicas: 5
  maxReplicas: 20
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
✓ Container crash → K8s tự động restart
✓ Node failure → Pods recreated trên node khác
✓ Deploy code mới → Rolling update, zero downtime
✓ Traffic spike → Auto-scale lên 20 replicas
✓ Multi-zone → Survive zone failures
```

---

### 3. Dynamic Scaling Needs

**SCENARIO:**
```
Traffic patterns:
├── 8 AM - 12 PM: 1000 req/s
├── 12 PM - 5 PM: 5000 req/s  
├── 5 PM - 10 PM: 10000 req/s
├── 10 PM - 8 AM: 100 req/s
└── Black Friday: 50000 req/s
```

**TẠI SAO CẦN K8S:**

**Manual Scaling (Không có K8s):**
```
08:00 - Traffic thấp
     → 10 servers idle → Lãng phí tiền

12:00 - Traffic tăng
     → Manually add servers (30 min)
     → Users experience slowness

18:00 - Peak traffic
     → Frantically adding more servers
     → Some requests timeout
     → Users complain

22:00 - Traffic giảm
     → Servers still running → Lãng phí tiền
     → Quên tắt servers → Bill shock cuối tháng
```

**Với Kubernetes HPA:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 3    # 100 req/s
  maxReplicas: 100  # 50000 req/s
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50  # Scale up 50% at a time
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scale down
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
```

**KẾT QUẢ:**
```
08:00 → 3 pods (min)     → Cost: $30/hour
12:00 → Auto scale to 15 → Cost: $150/hour  
18:00 → Auto scale to 30 → Cost: $300/hour
22:00 → Auto scale to 5  → Cost: $50/hour

Black Friday:
00:00 → 100 pods → Survive traffic spike!
02:00 → Back to 10 pods

Savings: 40-60% vs always-on max capacity
Performance: Always optimal
Effort: Zero (fully automated)
```

---

### 4. Multi-Environment Management

**SCENARIO:**
```
Environments:
├── Development (10 services)
├── Staging (10 services)
├── QA (10 services)
├── UAT (10 services)
└── Production (10 services)

Total: 50 service deployments
```

**TẠI SAO CẦN K8S:**

**Kubernetes Namespaces:**
```yaml
# Organize by environment
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace qa
kubectl create namespace uat
kubectl create namespace production

# Deploy to specific environment
kubectl apply -f app.yaml -n dev
kubectl apply -f app.yaml -n staging
kubectl apply -f app.yaml -n production

# Resource quotas per environment
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    pods: "50"

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    pods: "500"
```

**LỢI ÍCH:**
```
✓ Isolation giữa environments
✓ Consistent deployment process
✓ Resource quotas per environment
✓ Easy promotion: dev → staging → prod
✓ Config management với ConfigMaps/Secrets
```

---

### 5. Team Size & Growth

**SCENARIO:**
```
Team growth:
├── Now: 5 developers
├── 6 months: 15 developers
├── 12 months: 30 developers
└── Multiple teams working on different services
```

**TẠI SAO CẦN K8S:**

```
Scaling Team = Scaling Infrastructure

Kubernetes enables:
✓ Self-service deployments (devs deploy riêng)
✓ Standardized platform
✓ Clear ownership (namespace per team)
✓ Resource isolation
✓ GitOps workflows
```

**VÍ DỤ:**
```
Team Structure:
├── Team Frontend (namespace: frontend)
│   ├── webapp-service
│   └── mobile-api-service
│
├── Team Backend (namespace: backend)
│   ├── user-service
│   ├── product-service
│   └── order-service
│
└── Team Data (namespace: data)
    ├── analytics-service
    └── reporting-service

Mỗi team:
✓ Có namespace riêng
✓ Resource quotas
✓ RBAC permissions
✓ Deploy independent
✓ Monitor services riêng
```

---

## ❌ Khi Nào KHÔNG NÊN Dùng Kubernetes

### 1. Simple Applications

**SCENARIO:**
```
├── 1 web server
├── 1 database
├── < 1000 users
└── 1 server
```

**TẠI SAO KHÔNG CẦN K8S:**
```
Kubernetes overkill:
❌ Setup complexity >>> benefit
❌ Cost cao hơn (vs simple VPS)
❌ Learning curve steep
❌ Maintenance overhead

Giải pháp tốt hơn: Docker Compose
```

**VÍ DỤ:**
```yaml
# docker-compose.yml - ĐƠN GIẢN và ĐỦ!
version: '3.8'
services:
  web:
    image: myapp:latest
    ports:
      - "80:3000"
    environment:
      - DB_HOST=db
    restart: unless-stopped
  
  db:
    image: postgres:14
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  pgdata:

# Deploy:
$ docker-compose up -d

# Update:
$ docker-compose pull && docker-compose up -d

✓ 5 phút setup
✓ $10-20/month
✓ Easy to understand
```

---

### 2. Small Team Without K8s Expertise

**SCENARIO:**
```
Team:
├── 2-3 developers
├── No DevOps engineer
├── No K8s experience
└── Limited time/budget
```

**TẠI SAO KHÔNG NÊN:**
```
Kubernetes Learning Curve:
├── 2-3 tháng học cơ bản
├── 6-12 tháng proficient
└── Liên tục phải học updates

Risks:
❌ Delayed product development
❌ Misconfiguration → security issues
❌ Downtime do lack of expertise
❌ Team frustration
```

**GIẢI PHÁP TỐT HƠN:**
```
Managed PaaS:
├── Heroku
├── Railway
├── Render
├── Digital Ocean App Platform
└── AWS Elastic Beanstalk

Benefits:
✓ Deploy trong 10 phút
✓ Zero K8s knowledge needed
✓ Auto-scaling included
✓ Focus on product
✓ Upgrade to K8s later when needed
```

---

### 3. Cost-Sensitive Projects

**SCENARIO:**
```
Budget: $50-100/month
Users: < 5000
Traffic: Low to medium
```

**TẠI SAO KHÔNG NÊN:**

**Cost Comparison:**
```
SIMPLE VPS:
├── DigitalOcean Droplet: $12/month
├── Database: $15/month
├── Backup: $5/month
└── Total: $32/month

KUBERNETES (Managed):
├── GKE Cluster: $75/month (control plane)
├── 3 Nodes (n1-standard-1): $75/month
├── Load Balancer: $20/month
├── Storage: $10/month
└── Total: $180/month

Difference: $150/month = 5.6x more expensive!
```

**KHI NÀO KUBERNETES WORTH IT:**
```
Khi scale lên:
├── Need 10+ servers
├── Auto-scaling saves costs
├── High availability required
└── → Kubernetes becomes cost-effective
```

---

### 4. MVP / Prototype / Proof of Concept

**SCENARIO:**
```
Goal: Validate idea nhanh nhất
Timeline: 2-4 tuần
Uncertainty: Có thể pivot hoặc kill project
```

**TẠI SAO KHÔNG NÊN:**
```
Kubernetes overhead:
❌ 1-2 tuần setup = 50% timeline
❌ Complexity distracts from product
❌ Nếu pivot → wasted effort

Principle: Start simple, scale later!
```

**GIẢI PHÁP MVP:**
```
Phase 1: MVP (Week 1-4)
├── Deploy: Heroku/Railway
├── Database: Managed (Heroku Postgres)
├── Time: 1 day setup
└── Focus: 100% on product

Phase 2: Validation (Month 2-3)
├── If product works → Keep monitoring
├── If scaling issues appear → Consider K8s
└── If need microservices → Plan migration

Phase 3: Scale (Month 4+)
├── Migrate to Kubernetes
├── Have clear requirements now
├── Have resources/team
└── Have proven product
```

---

## 🎯 Decision Framework

### Checklist: Có Cần Kubernetes?

**Trả lời Yes/No:**

```
□ Có > 10 services/microservices?
□ Có > 20 containers?
□ Chạy trên multiple servers?
□ Cần 99.9%+ uptime?
□ Traffic không đều, cần auto-scaling?
□ Team > 10 developers?
□ Budget > $500/month cho infrastructure?
□ Có DevOps engineer với K8s experience?
□ Need multi-environment (dev/staging/prod)?
□ Need container orchestration features?

Score:
├── 8-10 Yes: Kubernetes is MUST-HAVE ✅
├── 5-7 Yes: Kubernetes is RECOMMENDED ⚠️
├── 3-4 Yes: Consider alternatives first 🤔
└── 0-2 Yes: DON'T use Kubernetes ❌
```

---

### Migration Path

**Đừng All-in Ngay!**

```
Stage 1: Docker (1-3 months)
├── Containerize applications
├── Use Docker Compose locally
├── Deploy to VPS hoặc PaaS
└── Learn container concepts

Stage 2: Evaluate (Month 3-6)
├── Monitor scaling needs
├── Track cost vs value
├── Assess team readiness
└── Decide: Kubernetes or stay?

Stage 3: Kubernetes (Month 6+)
├── Start với managed K8s (GKE/EKS)
├── Migrate 1 service first (pilot)
├── Learn and iterate
├── Gradually migrate more services
└── Build expertise

Don't skip stages!
```

---

## 🎯 Key Takeaways

### Ghi Nhớ 5 Điều Quan Trọng

1. **Kubernetes KHÔNG phải cho everyone**
   - Chỉ dùng khi thực sự cần
   
2. **Start simple, scale later**
   - MVP → Docker → Kubernetes
   
3. **Consider costs**
   - K8s expensive cho small projects
   - Cost-effective when scale
   
4. **Team expertise matters**
   - Cần training time
   - Có DevOps engineer tốt hơn
   
5. **Alternatives exist**
   - PaaS cho simple apps
   - Docker Swarm cho medium apps
   - Kubernetes cho enterprise

---

## 💪 Bài Tập Tự Đánh Giá

### Đánh Giá Dự Án Của Bạn

**Scenario 1:**
```
Personal Blog
├── WordPress
├── 100 visitors/day
├── 1 developer (you)
└── Budget: $20/month
```

**Nên dùng gì?**
<details>
<summary>Xem đáp án</summary>

**KHÔNG dùng Kubernetes!**

Giải pháp: Shared hosting hoặc Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  wordpress:
    image: wordpress:latest
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_PASSWORD: secret
  
  db:
    image: mysql:5.7
    volumes:
      - db_data:/var/lib/mysql

# Deploy on $10 VPS
```

Lý do: Quá simple, K8s overkill!
</details>

---

**Scenario 2:**
```
E-learning Platform
├── 50,000 students
├── 15 microservices
├── Peak traffic: 8-10 PM (10x normal)
├── Team: 20 developers
└── Budget: $5000/month
```

**Nên dùng gì?**
<details>
<summary>Xem đáp án</summary>

**Kubernetes là MUST-HAVE!**

Lý do:
✓ Multiple microservices
✓ Need auto-scaling (peak traffic)
✓ Large team
✓ Budget sufficient
✓ High availability needed

Setup:
```yaml
# Managed Kubernetes (GKE/EKS)
# Auto-scaling enabled
# Multi-zone deployment
# Monitoring & logging integrated
```
</details>

---

**Scenario 3:**
```
SaaS MVP
├── 2 developers
├── Validating idea
├── Timeline: 6 weeks
├── Budget: $200/month
└── No K8s experience
```

**Nên dùng gì?**
<details>
<summary>Xem đáp án</summary>

**Heroku/Railway/Render (PaaS)!**

Lý do:
✓ Fast deployment
✓ Zero K8s learning curve
✓ Focus on product
✓ Can migrate to K8s later if successful

```bash
# Deploy to Heroku in 5 minutes
$ heroku create myapp
$ git push heroku main
$ heroku ps:scale web=1

Done!
```
</details>

---

## 📚 Alternatives to Kubernetes

### When You Don't Need K8s

**1. Docker Compose**
```
Use when:
✓ Single server
✓ < 10 containers
✓ Development/small production

Example: docker-compose.yml
```

**2. PaaS (Platform as a Service)**
```
Options:
├── Heroku (easiest)
├── Railway (modern)
├── Render (affordable)
├── Fly.io (edge deployment)
└── DigitalOcean App Platform

Use when:
✓ Want simplicity
✓ Small/medium apps
✓ No DevOps resources
```

**3. Docker Swarm**
```
Use when:
✓ Need orchestration
✓ Simpler than K8s
✓ Team knows Docker well
✓ Medium scale (5-20 nodes)
```

**4. Serverless**
```
Options:
├── AWS Lambda
├── Google Cloud Functions
├── Azure Functions
└── Vercel/Netlify

Use when:
✓ Event-driven workloads
✓ Variable traffic
✓ No server management wanted
```

---

## 🚀 Tiếp Theo

Bạn đã biết khi nào nên/không nên dùng Kubernetes!

**Next:** [Phần 2: Kiến Trúc Kubernetes →](../02-architecture/README.md)

Ở phần tiếp theo, chúng ta sẽ deep dive vào kiến trúc của Kubernetes, hiểu cách nó hoạt động bên trong.

---

[⬅️ 1.2. K8s vs Docker](./02-k8s-vs-docker.md) | [🏠 Mục Lục Chính](../README.md) | [📂 Phần 1: Introduction](./README.md) | [➡️ Phần 2: Architecture](../02-architecture/README.md)
