# 📖 Hướng Dẫn Sử Dụng Tài Liệu Kubernetes

> Cách học hiệu quả dựa trên mục tiêu và trình độ của bạn

---

## ✅ Tổng Quan Tài Liệu

### 📊 Cấu Trúc Hoàn Chỉnh

```
📚 46 files markdown
📂 10 phần chính
🇻🇳 100% tiếng Việt (giữ thuật ngữ tiếng Anh)
💻 Có examples và thực hành
```

| Phần | Chủ Đề | Files | Độ Quan Trọng |
|------|---------|-------|---------------|
| **1. Introduction** | Tổng quan K8s | 4 | ⭐⭐⭐ |
| **2. Architecture** | Cách K8s hoạt động | 4 | ⭐⭐⭐⭐⭐ |
| **3. Core Concepts** | Pods, Namespaces, Labels | 5 | ⭐⭐⭐⭐⭐ |
| **4. Workloads** | Deployments, StatefulSets | 7 | ⭐⭐⭐⭐⭐ |
| **5. Networking** | Services, Ingress | 4 | ⭐⭐⭐⭐ |
| **6. Configuration** | ConfigMaps, Secrets | 3 | ⭐⭐⭐⭐ |
| **7. Storage** | Volumes, PV/PVC | 4 | ⭐⭐⭐ |
| **8. High Availability** | Health checks, Scaling | 4 | ⭐⭐⭐⭐⭐ |
| **9. Next Steps** | Lộ trình nâng cao | 2 | ⭐⭐⭐ |
| **10. Observability** | Monitoring, Logging | 7 | ⭐⭐⭐⭐ |

---

## 🎯 Chọn Cách Học Phù Hợp

### 1️⃣ Đánh Giá Trình Độ Hiện Tại

**Trả lời các câu hỏi để tự đánh giá:**

**A. Về Kinh Nghiệm:**
- [ ] Chưa biết gì về containers
- [ ] Biết Docker cơ bản
- [ ] Biết Docker tốt, chưa dùng K8s
- [ ] Đã dùng K8s, muốn hiểu sâu hơn

**B. Về Mục Tiêu:**
- [ ] Hiểu K8s để làm việc/interview
- [ ] Deploy applications lên K8s
- [ ] Quản lý K8s cluster (DevOps/SRE)
- [ ] Setup monitoring/observability
- [ ] Thi certification (CKA/CKAD)

**C. Về Thời Gian:**
- [ ] Có nhiều thời gian, học kỹ từng phần
- [ ] Thời gian có hạn, cần học nhanh
- [ ] Học cuối tuần/lúc rảnh
- [ ] Intensive, muốn nhanh nhất có thể

---

### 2️⃣ Lộ Trình Theo Trình Độ

## 🌱 A. Hoàn Toàn Mới Với Containers/K8s

**Nên học gì và theo thứ tự nào:**

### Bước 1: Hiểu "Tại Sao" Cần Kubernetes

**Đọc:**
- `README.md` - Overview toàn bộ
- `QUICK-START.md` - Hướng dẫn setup
- **Phần 1: Introduction** (đọc hết 4 files)
  - `01-what-is-kubernetes.md` ⭐ Hiểu K8s giải quyết vấn đề gì
  - `02-k8s-vs-docker.md` ⭐ Phân biệt Docker và K8s
  - `03-when-to-use-k8s.md` - Biết khi nào nên/không nên dùng

**Checkpoint:** Có thể giải thích K8s là gì cho người khác?

---

### Bước 2: Hiểu Cách K8s Hoạt Động

**Đọc:**
- **Phần 2: Architecture** (đọc hết 4 files)
  - `01-overview.md` ⭐ Big picture
  - `02-control-plane.md` 🔥 **CỰC QUAN TRỌNG** - Đọc kỹ!
  - `03-worker-nodes.md` ⭐ Nơi chạy containers

**Thực hành:**
```bash
# Setup local cluster
minikube start

# Xem components
kubectl get nodes
kubectl get pods -n kube-system
kubectl describe node minikube
```

**Checkpoint:** Vẽ được diagram K8s architecture? Giải thích được Control Plane và Worker Node?

---

### Bước 3: Làm Chủ Building Blocks

**Đọc:**
- **Phần 3: Core Concepts** (đọc hết 5 files theo thứ tự)
  - `01-cluster-and-nodes.md` - Hiểu Node
  - `02-pods.md` 🔥 **CỰC QUAN TRỌNG** - Đơn vị cơ bản nhất
  - `03-namespaces.md` ⭐ Tổ chức resources
  - `04-labels-selectors.md` ⭐ Query và group objects

**Thực hành:**
```bash
# Tạo Pod đầu tiên
kubectl run nginx --image=nginx
kubectl get pods
kubectl describe pod nginx
kubectl logs nginx
kubectl delete pod nginx

# Tạo namespace
kubectl create namespace dev
kubectl create namespace prod

# Query với labels
kubectl get pods -l app=web
kubectl get pods --show-labels
```

**Checkpoint:** Tạo được Pod, biết dùng Namespace và Labels?

---

### Bước 4: Deploy và Quản Lý Applications

**Đọc theo thứ tự:**
- **Phần 4: Workloads**
  - `README.md` - Overview các workloads
  - `01-replicaset.md` - Hiểu nền tảng
  - `02-deployment.md` 🔥🔥 **QUAN TRỌNG NHẤT** - Đọc cực kỹ!
  - `05-jobs-cronjobs.md` - Batch tasks (dễ)
  - `04-daemonset.md` - Node services (đọc khi cần)
  - `03-statefulset.md` - Databases (advanced, học sau)

**Thực hành:**
```bash
# Deploy application
kubectl create deployment web --image=nginx --replicas=3
kubectl get deployments
kubectl get pods

# Scale
kubectl scale deployment web --replicas=5

# Rolling update
kubectl set image deployment/web nginx=nginx:1.22
kubectl rollout status deployment/web

# Rollback nếu có vấn đề
kubectl rollout undo deployment/web

# Health checks
kubectl edit deployment web  # Thêm liveness/readiness probes
```

**Checkpoint:** Deploy được app với Deployment, biết scale và update?

---

### Bước 5: Networking Cơ Bản

**Đọc:**
- **Phần 5: Networking**
  - `01-pod-networking.md` - Hiểu Pod communication
  - `02-services.md` 🔥 **CỰC QUAN TRỌNG** - Expose apps
  - `03-ingress.md` ⭐ HTTP/HTTPS routing

**Thực hành:**
```bash
# Expose với Service
kubectl expose deployment web --port=80 --type=ClusterIP
kubectl get service web

# Test từ trong cluster
kubectl run test --image=busybox -it --rm -- wget -O- http://web

# NodePort (access từ ngoài)
kubectl expose deployment web --port=80 --type=NodePort --name=web-nodeport
minikube service web-nodeport  # Mở browser
```

**Checkpoint:** Expose được app, hiểu các loại Services?

---

### Bước 6: Configuration Management

**Đọc:**
- **Phần 6: Configuration**
  - `01-configmap.md` ⭐ Non-sensitive config
  - `02-secrets.md` ⭐ Sensitive data

**Thực hành:**
```bash
# ConfigMap
kubectl create configmap app-config --from-literal=LOG_LEVEL=debug
kubectl get configmap app-config -o yaml

# Secret
kubectl create secret generic db-password --from-literal=password=secret123
kubectl get secret db-password -o yaml

# Dùng trong Pod
# Tạo Pod với env from ConfigMap/Secret
```

**Checkpoint:** Biết dùng ConfigMap và Secret?

---

### Bước 7: Production Readiness

**Đọc:**
- **Phần 8: High Availability**
  - `01-self-healing.md` - Tự động recovery
  - `02-health-checks.md` 🔥 **CỰC QUAN TRỌNG** - Must know!
  - `03-scaling.md` ⭐ HPA, VPA

**Thực hành:**
```bash
# Add health checks to Deployment
# Edit deployment, add:
# livenessProbe, readinessProbe

# Setup HPA
kubectl autoscale deployment web --cpu-percent=70 --min=2 --max=10
kubectl get hpa
```

**Checkpoint:** App có health checks và có thể auto-scale?

---

### Bước 8: Storage (Khi Cần)

**Đọc khi cần deploy databases/stateful apps:**
- **Phần 7: Storage**
  - `01-volumes.md` - Ephemeral storage
  - `02-persistent-volumes.md` ⭐ PV/PVC
  - `03-storage-classes.md` - Dynamic provisioning

**Thực hành:**
```bash
# Tạo PVC
# Deploy database với PVC
# Verify data persistence
```

---

## 🚀 B. Biết Docker, Mới Với K8s

**Khác gì với Beginner:**
- ⏭️ Skip Phần 1 (đã hiểu containers)
- 📖 Phần 2: Đọc nhanh, focus `02-control-plane.md`
- 🎯 Focus: Phần 3-6 và Phần 8

**Thứ tự học:**

1. **Phần 2** (Architecture) - Đọc nhanh để hiểu
2. **Phần 3** (Core Concepts) - Đọc kỹ
3. **Phần 4** (Workloads) - Focus `02-deployment.md` 🔥
4. **Phần 5** (Networking) - Services + Ingress
5. **Phần 6** (Configuration) - ConfigMap + Secret
6. **Phần 8** (HA) - Health checks + HPA
7. **Phần 7** (Storage) - Khi cần database

**Project-based learning:**
- Deploy multi-tier app (frontend + backend + database)
- Setup Ingress cho routing
- Add monitoring
- Implement CI/CD

---

## 💼 C. Focus Vào Công Việc/Interview

**Mục tiêu:** Biết đủ để làm việc hoặc pass interview

### Must-Know Files (Đọc Trước Tiên)

**Critical - Không thể skip:**
1. `01-introduction/01-what-is-kubernetes.md` - Giải thích K8s là gì
2. `02-architecture/02-control-plane.md` 🔥 - Hiểu internals
3. `03-core-concepts/02-pods.md` 🔥 - Building block
4. `04-workloads/02-deployment.md` 🔥 - Deploy apps
5. `05-networking/02-services.md` 🔥 - Expose apps
6. `06-configuration/01-configmap.md` - Config
7. `06-configuration/02-secrets.md` - Secrets
8. `08-high-availability/02-health-checks.md` 🔥 - Production

### Should-Know Files (Đọc Tiếp)

9. `03-core-concepts/03-namespaces.md`
10. `03-core-concepts/04-labels-selectors.md`
11. `04-workloads/03-statefulset.md` (nếu làm databases)
12. `05-networking/03-ingress.md`
13. `08-high-availability/03-scaling.md` (HPA)
14. `07-storage/02-persistent-volumes.md` (PV/PVC)

### Nice-to-Have (Học Sau)

- Phần còn lại: Học khi cần hoặc có thời gian

**Thực hành:**
- Deploy 1 complete application với tất cả components
- Practice kubectl commands cho fluency
- Troubleshoot common issues

---

## 🎓 D. Chuẩn Bị Thi Certification (CKA/CKAD)

**Coverage:** Cần đọc toàn bộ Phần 1-8

**Thứ tự:**
1. **Đọc tuần tự** Phần 1 → Phần 8
2. **Focus đặc biệt:**
   - CKA: Cluster management, troubleshooting, RBAC
   - CKAD: Application deployment, Services, ConfigMaps/Secrets
3. **Skip:** Phần 9 (Next Steps) - học sau khi thi
4. **Phần 10** (Observability):
   - CKA: Đọc
   - CKAD: Optional

**Practice:**
- Hands-on practice CỰC KỲ QUAN TRỌNG
- Làm practice exams (Killer.sh)
- Time management (finish trong thời gian)

---

## 🔭 E. Focus Observability/SRE

**Profile:** SRE, Platform Engineer, Infrastructure

**Thứ tự học:**

### Foundation
1. **Phần 2** (Architecture) - Hiểu sâu internals
2. **Phần 3** (Core Concepts) - Đầy đủ
3. **Phần 4** (Workloads) - Focus StatefulSet
4. **Phần 5** (Networking) - Đầy đủ
5. **Phần 8** (HA & Scaling) - Critical

### Specialization
6. **Phần 10: Observability** 🔥 - DEEP DIVE
   - Đọc hết 7 files
   - Thực hành deploy monitoring stack
   - `01-metrics-architecture.md` - Chi tiết
   - `02-logging-architecture.md` - Chi tiết
   - `05-deploying-monitoring-agents.md` - Hands-on

**Thực hành:**
- Deploy Prometheus + Grafana
- Setup Datadog/Dynatrace Agent
- Configure alerts và dashboards
- Log aggregation (ELK/Loki)
- Distributed tracing

---

## 📚 Tips Học Hiệu Quả

### 1. Active Reading

**❌ Không nên:**
- Đọc lướt không ghi chú
- Đọc hết rồi mới thực hành
- Đọc nhiều quá không hiểu gì

**✅ Nên:**
- Ghi chú key concepts
- Vẽ diagrams để hiểu
- Tự hỏi "Tại sao?" và "Khi nào dùng?"
- Giải thích lại bằng lời của mình

### 2. Practice Immediately

**Đọc xong → Thực hành ngay!**

```
Ví dụ:
├─ Đọc về Pods → Tạo Pod ngay
├─ Đọc về Deployments → Deploy app ngay
├─ Đọc về Services → Expose app ngay
└─ KHÔNG đọc hết 10 phần rồi mới practice!

→ Knowledge retention cao hơn nhiều!
```

### 3. Build Real Projects

**Học qua projects:**
- Không chỉ chạy commands từ tài liệu
- Tạo projects của riêng bạn
- Deploy apps thực tế

**Project ideas:**
- WordPress + MySQL
- Microservices với 3+ services
- E-commerce platform
- CI/CD pipeline với K8s

### 4. Learn by Breaking Things

**Cách học tốt nhất: Break và fix!**

**Thử nghiệm:**
```bash
# Delete Pod → Watch tự động recreate
kubectl delete pod <name>

# Set resource limits thấp → See OOMKill
# Wrong image name → Debug ImagePullBackOff
# Wrong Service selector → Troubleshoot

→ Troubleshooting skills tăng nhanh!
```

### 5. Teach Others

**Best way to learn: Giải thích cho người khác**

Cách thực hiện:
- Viết blog
- Present cho team
- Answer questions trên forums
- Hoặc tự giải thích lại (rubber duck)

---

## 🗺️ Navigation Trong Tài Liệu

### Cách Di Chuyển

**1. Entry Points:**
```
README.md → Overview và table of contents
USAGE-GUIDE.md (file này) → Chọn cách học
QUICK-START.md → Setup môi trường
```

**2. Links Between Files:**
```
Mỗi file có navigation:
[⬅️ Previous] | [➡️ Next] | [🏠 Home]
```

**3. README Files:**
```
Mỗi section có README.md:
├─ Overview section đó
├─ List các topics
└─ Thời gian ước tính
```

### Khi Gặp Khó Khăn

**Concept khó hiểu?**
1. Đọc lại section đó chậm hơn
2. Xem diagrams và examples
3. Google thêm resources
4. Thực hành hands-on
5. Nếu vẫn khó → Skip, học tiếp, quay lại sau

**Không cần hiểu 100% ngay lần đầu!**

---

## 💡 Pro Tips

### Setup Môi Trường

**Cài ngay từ đầu:**
```bash
# Option 1: minikube (recommended)
brew install minikube  # macOS
minikube start

# Option 2: kind
kind create cluster

# Option 3: Docker Desktop
# Enable Kubernetes trong Settings
```

### Kubectl Productivity

**Aliases để save time:**
```bash
# Add to ~/.bashrc or ~/.zshrc
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kdel='kubectl delete'
alias kl='kubectl logs'
alias ka='kubectl apply -f'
alias kx='kubectl exec -it'

# Auto-completion
source <(kubectl completion bash)  # bash
source <(kubectl completion zsh)   # zsh
```

### Personal Cheat Sheet

**Tạo file notes riêng:**
```bash
# commands hay dùng
kubectl get pods -o wide
kubectl describe pod <name>
kubectl logs -f <pod>
kubectl exec -it <pod> -- bash
kubectl port-forward <pod> 8080:80

# debugging
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top nodes
kubectl top pods
```

### Join Communities

**Học cùng cộng đồng:**
- [Kubernetes Slack](https://kubernetes.slack.com/)
- [Reddit r/kubernetes](https://reddit.com/r/kubernetes)
- Local meetups
- Stack Overflow

---

## ✅ Self-Assessment Checkpoints

### Checkpoint 1: Basics
```
□ Giải thích được K8s là gì
□ Vẽ được architecture diagram
□ Hiểu Control Plane components
□ Setup được local cluster
□ Chạy được kubectl commands
```

### Checkpoint 2: Core Skills
```
□ Tạo và quản lý Pods
□ Deploy apps với Deployments
□ Rolling update và rollback
□ Dùng Namespaces và Labels
□ Understand ReplicaSet vs Deployment
```

### Checkpoint 3: Production Ready
```
□ Expose apps qua Services
□ Setup Ingress
□ Dùng ConfigMaps và Secrets
□ Configure health checks
□ Setup HPA
□ Deploy với PersistentVolumes
```

### Checkpoint 4: Advanced
```
□ Deploy complete microservices
□ Setup monitoring stack
□ Implement logging
□ Troubleshoot issues
□ Understand security best practices
```

---

## 🎯 Khi Nào "Đủ"?

**Bạn đã "đủ" khi:**

✅ Deploy được app production-ready  
✅ Troubleshoot được common issues  
✅ Hiểu networking và services  
✅ Biết configure monitoring  
✅ Có thể giải thích architecture cho người khác  

**Không cần:**
- ❌ Nhớ hết mọi commands
- ❌ Hiểu 100% mọi details
- ❌ Biết hết advanced topics ngay

**Remember:** Kubernetes rất rộng, học liên tục!

---

## 📖 Resources Bổ Sung

### Official Docs
- [Kubernetes Docs](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

### Practice
- [Killercoda K8s Playground](https://killercoda.com/kubernetes)
- [Play with Kubernetes](https://labs.play-with-k8s.com/)
- [KodeKloud](https://kodekloud.com/)

### Books
- "Kubernetes Up & Running" - Kelsey Hightower
- "The Kubernetes Book" - Nigel Poulton

---

## 🎉 Kết Luận

**Tài liệu này là starting point:**
- 46 files comprehensive coverage
- Tiếng Việt dễ hiểu
- Hands-on examples
- Flexible learning paths

**Bạn KHÔNG cần:**
- Follow strict order (trừ basics)
- Đọc hết mọi thứ
- Học theo timeline cụ thể
- Biết tất cả trước khi bắt đầu

**Bạn NÊN:**
- Chọn path phù hợp mục tiêu
- Practice while learning
- Build real projects
- Learn at YOUR own pace
- Ask questions, help others

---

**Sẵn sàng bắt đầu?**

👉 [README.md](./README.md) - Xem tổng quan  
👉 [Phần 1: Introduction](./01-introduction/README.md) - Beginners  
👉 [Phần 2: Architecture](./02-architecture/README.md) - Có kinh nghiệm Docker  
👉 [Phần 10: Observability](./10-observability-fundamentals/README.md) - SRE focus  

**Chúc bạn học tốt! 🚀**

