# 📘 Kubernetes Learning Guide - Hướng Dẫn Học Kubernetes Từ Đầu

> Tài liệu học Kubernetes có hệ thống, dễ hiểu cho người mới bắt đầu
> 
> **Đặc điểm:** Giải thích bằng ví dụ thực tế • Trình bày có hệ thống • Giải thích "vì sao" chứ không chỉ định nghĩa

---

## 🎯 Mục Tiêu Của Tài Liệu

Sau khi học xong tài liệu này, bạn sẽ:
- ✅ Hiểu Kubernetes là gì và giải quyết vấn đề gì
- ✅ Nắm vững kiến trúc và các thành phần của K8s
- ✅ Biết cách deploy và quản lý ứng dụng trên K8s
- ✅ Hiểu networking, storage, configuration trong K8s
- ✅ Áp dụng được các best practices về HA và scaling

---

## 📚 Mục Lục Chi Tiết

### [**Phần 1: Introduction - Giới Thiệu**](./01-introduction/README.md)
Làm quen với Kubernetes, hiểu vấn đề và giải pháp

- [1.1. Kubernetes Là Gì?](./01-introduction/01-what-is-kubernetes.md)
  - Vấn đề trong thực tế
  - Kubernetes giải quyết như thế nào
  - Các tính năng cốt lõi
  
- [1.2. So Sánh Kubernetes vs Docker](./01-introduction/02-k8s-vs-docker.md)
  - Sự khác biệt cơ bản
  - Docker Swarm vs Kubernetes
  - Khi nào dùng cái gì
  
- [1.3. Khi Nào Nên Dùng Kubernetes](./01-introduction/03-when-to-use-k8s.md)
  - Use cases phù hợp
  - Khi nào KHÔNG nên dùng K8s
  - Các giải pháp thay thế

---

### [**Phần 2: Architecture - Kiến Trúc**](./02-architecture/README.md)
Hiểu sâu về kiến trúc và cách K8s hoạt động

- [2.1. Tổng Quan Kiến Trúc Kubernetes](./02-architecture/01-overview.md)
  - Cluster là gì
  - Master-Worker model
  - Communication flow
  
- [2.2. Control Plane - Bộ Não Của Cluster](./02-architecture/02-control-plane.md)
  - API Server
  - etcd - Database của cluster
  - Scheduler - Bộ lập lịch
  - Controller Manager
  - Cloud Controller Manager
  
- [2.3. Worker Node - Nơi Chạy Workload](./02-architecture/03-worker-nodes.md)
  - kubelet
  - kube-proxy
  - Container Runtime
  - Addons

---

### [**Phần 3: Core Concepts - Khái Niệm Cốt Lõi**](./03-core-concepts/README.md)
Các khái niệm nền tảng cần nắm vững

- [3.1. Cluster & Node](./03-core-concepts/01-cluster-and-nodes.md)
  - Cluster là gì
  - Node types và roles
  - Node management
  
- [3.2. Pod - Đơn Vị Cơ Bản Nhất](./03-core-concepts/02-pods.md)
  - Pod là gì và vì sao cần Pod
  - Single vs Multi-container Pod
  - Pod lifecycle
  - Init containers và Sidecar pattern
  
- [3.3. Namespace - Phân Vùng Resources](./03-core-concepts/03-namespaces.md)
  - Namespace là gì
  - Use cases thực tế
  - Namespaces mặc định
  - Best practices
  
- [3.4. Labels & Selectors](./03-core-concepts/04-labels-selectors.md)
  - Labels để tổ chức resources
  - Selectors để query
  - Annotations vs Labels
  - Ví dụ thực tế

---

### [**Phần 4: Workloads - Quản Lý Ứng Dụng**](./04-workloads/README.md)
Các cách deploy và quản lý ứng dụng

- [4.1. ReplicaSet](./04-workloads/01-replicaset.md)
  - Đảm bảo số lượng Pod
  - Self-healing cơ bản
  
- [4.2. Deployment - Workload Phổ Biến Nhất](./04-workloads/02-deployment.md)
  - Quản lý stateless apps
  - Rolling updates
  - Rollback
  - Update strategies
  - Use cases thực tế
  
- [4.3. StatefulSet - Cho Ứng Dụng Stateful](./04-workloads/03-statefulset.md)
  - Khi nào cần StatefulSet
  - Stable network identity
  - Ordered deployment/scaling
  - Persistent storage
  - Deploy database cluster
  
- [4.4. DaemonSet - Chạy Trên Mọi Node](./04-workloads/04-daemonset.md)
  - Use cases: monitoring, logging, networking
  - Node selection
  - Update strategies
  
- [4.5. Job & CronJob - Batch Processing](./04-workloads/05-jobs-cronjobs.md)
  - Job cho task một lần
  - CronJob cho scheduled tasks
  - Parallel jobs
  - Use cases: backup, ETL, reporting

---

### [**Phần 5: Networking - Mạng Trong K8s**](./05-networking/README.md)
Cách Pod giao tiếp với nhau và bên ngoài

- [5.1. Pod Networking](./05-networking/01-pod-networking.md)
  - Mô hình mạng K8s
  - Pod-to-Pod communication
  - CNI plugins
  - DNS trong cluster
  
- [5.2. Service - Expose Ứng Dụng](./05-networking/02-services.md)
  - Service discovery
  - ClusterIP - Internal service
  - NodePort - Expose ra ngoài
  - LoadBalancer - Production
  - ExternalName
  - Headless Service
  
- [5.3. Ingress - HTTP/HTTPS Router](./05-networking/03-ingress.md)
  - Ingress là gì
  - Ingress Controller
  - Path-based và Host-based routing
  - SSL/TLS termination
  - Ví dụ với NGINX Ingress

---

### [**Phần 6: Configuration - Quản Lý Cấu Hình**](./06-configuration/README.md)
Tách biệt configuration khỏi code

- [6.1. ConfigMap](./06-configuration/01-configmap.md)
  - Lưu trữ configuration
  - Các cách sử dụng (env vars, files, command args)
  - Update ConfigMap
  - Use cases thực tế
  
- [6.2. Secret - Quản Lý Thông Tin Nhạy Cảm](./06-configuration/02-secrets.md)
  - Secret types
  - Encoding vs Encryption
  - Best practices
  - External secret management
  - Security considerations

---

### [**Phần 7: Storage - Lưu Trữ Dữ Liệu**](./07-storage/README.md)
Persistent data trong môi trường container

- [7.1. Volumes](./07-storage/01-volumes.md)
  - Vấn đề ephemeral storage
  - Các loại Volume (emptyDir, hostPath, cloud volumes)
  - Volume mounts
  
- [7.2. PersistentVolume & PersistentVolumeClaim](./07-storage/02-persistent-volumes.md)
  - PV và PVC là gì
  - Binding process
  - Access modes
  - Reclaim policies
  - Lifecycle
  
- [7.3. StorageClass - Dynamic Provisioning](./07-storage/03-storage-classes.md)
  - Dynamic vs Static provisioning
  - StorageClass parameters
  - Default StorageClass
  - Cloud provider examples

---

### [**Phần 8: High Availability & Scaling**](./08-high-availability/README.md)
Đảm bảo ứng dụng luôn sẵn sàng và tự động mở rộng

- [8.1. Self-Healing](./08-high-availability/01-self-healing.md)
  - Controller reconciliation loop
  - Automatic restart và rescheduling
  - Node failure handling
  
- [8.2. Health Checks](./08-high-availability/02-health-checks.md)
  - Liveness Probe
  - Readiness Probe
  - Startup Probe
  - Probe types và configuration
  
- [8.3. Scaling - Tự Động Mở Rộng](./08-high-availability/03-scaling.md)
  - Manual scaling
  - Horizontal Pod Autoscaler (HPA)
  - Vertical Pod Autoscaler (VPA)
  - Cluster Autoscaler
  - HA architecture patterns

---

### [**Phần 9: Next Steps - Bước Tiếp Theo**](./09-next-steps/README.md)

- [9.1. Lộ Trình Học Tiếp](./09-next-steps/01-learning-path.md)
  - Hands-on practice
  - Advanced topics
  - Production skills
  - Certification paths

---

## 🗺️ Lộ Trình Học Đề Xuất

### 🟢 Beginner (Người Mới Bắt Đầu)
**Mục tiêu:** Hiểu cơ bản, chạy được ứng dụng đơn giản

1. Đọc Phần 1: Introduction (hiểu tổng quan)
2. Đọc Phần 2: Architecture (biết K8s hoạt động như thế nào)
3. Đọc Phần 3: Core Concepts (nắm các khái niệm cơ bản)
4. Thực hành: Cài Minikube, chạy Pod đầu tiên
5. Đọc Phần 4.2: Deployment (workload quan trọng nhất)
6. Đọc Phần 5.2: Service (expose ứng dụng)
7. Thực hành: Deploy một web app đơn giản

**Thời gian:** 1-2 tuần (2-3 giờ/ngày)

### 🟡 Intermediate (Trung Cấp)
**Mục tiêu:** Deploy production-ready applications

8. Đọc hết Phần 4: Workloads (các loại workload)
9. Đọc hết Phần 5: Networking (Service, Ingress)
10. Đọc Phần 6: Configuration (ConfigMap, Secret)
11. Đọc Phần 7: Storage (PV, PVC)
12. Thực hành: Deploy microservices với database
13. Đọc Phần 8: High Availability (HA, scaling)
14. Thực hành: Setup monitoring, autoscaling

**Thời gian:** 2-3 tuần

### 🔴 Advanced (Nâng Cao)
**Mục tiêu:** Production operations, security, advanced features

15. **Phần 10: Observability Fundamentals** (nếu làm DevOps/SRE)
16. RBAC và Security
17. Network Policies
18. Helm charts
19. CI/CD với K8s
20. Monitoring & Logging (Prometheus, Grafana, ELK)
21. Service Mesh (Istio)
22. GitOps (ArgoCD, Flux)

**Thời gian:** 1-2 tháng

---

## 🛠️ Công Cụ Cần Thiết

### Môi Trường Học
- **Minikube:** K8s local trên máy tính (đơn giản nhất)
- **Kind:** Kubernetes in Docker (nhẹ, nhanh)
- **Docker Desktop:** Built-in K8s (Windows/Mac)
- **K3s:** K8s lightweight cho resource hạn chế

### Command Line Tools
- **kubectl:** CLI chính để tương tác với K8s
- **k9s:** Terminal UI cho K8s (tiện lợi)
- **kubectx/kubens:** Switch context và namespace nhanh

### Learning Platforms
- **Katacoda:** Interactive K8s scenarios
- **Play with Kubernetes:** Online playground
- **Killer.sh:** CKA/CKAD practice

---

## 📖 Cách Sử Dụng Tài Liệu Này

### Học Tuần Tự
Đọc từ Phần 1 → Phần 9 theo thứ tự. Mỗi phần được thiết kế dựa trên kiến thức phần trước.

### Tham Khảo Nhanh
Dùng mục lục để jump đến phần cần tìm hiểu. Mỗi file độc lập, có đầy đủ context.

### Thực Hành Kết Hợp
Sau mỗi phần lý thuyết, thực hành ngay với Minikube hoặc playground.

### Ghi Chú Cá Nhân
Clone repo và thêm notes riêng của bạn vào mỗi file.

---

## 🎓 Tips Học Hiệu Quả

1. **Không học thuộc lòng:** Hiểu concept, biết cách tìm kiếm khi cần
2. **Thực hành nhiều:** 70% practice, 30% theory
3. **Dùng kubectl explain:** `kubectl explain pod.spec` → Built-in docs
4. **Đọc code thực tế:** Xem manifest của open source projects
5. **Join community:** K8s Slack, Reddit r/kubernetes
6. **Tạo lab environment:** Break things và fix lại
7. **Viết blog:** Giải thích lại những gì học được

---

## 📝 Quy Ước Trong Tài Liệu

### Biểu Tượng
- 🎯 Mục tiêu học
- 💡 Ý tưởng/Tips quan trọng
- ⚠️ Cảnh báo/Lưu ý
- ✅ Best practice
- ❌ Anti-pattern (không nên làm)
- 🔍 Deep dive (tìm hiểu sâu)
- 📊 Ví dụ/Demo
- 🏢 Use case thực tế

### Code Blocks
- **YAML examples:** Simplified cho dễ hiểu
- **Commands:** Thực tế, chạy được luôn
- **Output:** Kết quả mẫu để tham khảo

### Ví Dụ Thực Tế
Mỗi concept đều có ví dụ liên hệ với:
- Doanh nghiệp/công ty
- Ứng dụng web thực tế
- Vấn đề production phổ biến

---

## 🤝 Đóng Góp

Tài liệu này được xây dựng để phục vụ cộng đồng học Kubernetes.

Nếu bạn:
- Tìm thấy lỗi
- Có ý tưởng cải thiện
- Muốn thêm ví dụ thực tế
- Cần giải thích thêm về một phần nào đó

→ Feedback luôn được chào đón!

---

## 📚 Tài Liệu Tham Khảo

- **Official Docs:** https://kubernetes.io/docs/
- **Kubernetes The Hard Way:** Kelsey Hightower
- **Books:**
  - "Kubernetes in Action" - Marko Lukša
  - "Kubernetes Up & Running" - Kelsey Hightower
- **Courses:**
  - Kubernetes for Absolute Beginners - KodeKloud
  - CKA/CKAD Certification Courses

---

## 🚀 Bắt Đầu Ngay

**Ready to start?** 

👉 [Bắt đầu với Phần 1: Introduction](./01-introduction/README.md)

Hoặc nếu đã biết cơ bản:

👉 [Nhảy tới Phần 4: Workloads](./04-workloads/README.md)

---

**Good luck on your Kubernetes journey! 🎉**

> "Kubernetes is not rocket science, it's container orchestration science!" 🚢


