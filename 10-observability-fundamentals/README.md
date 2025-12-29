# Phần 10: Observability Fundamentals trong K8s

> Kiến thức K8s cần thiết để triển khai giải pháp Observability (Datadog, Dynatrace, Prometheus...)

---

## 🎯 Mục Tiêu

Hiểu các K8s concepts liên quan đến Observability:
- ✅ Metrics collection architecture
- ✅ Logging patterns trong K8s
- ✅ Labels & Annotations cho filtering
- ✅ Service discovery cho monitoring
- ✅ RBAC permissions cho monitoring tools
- ✅ Deploy monitoring agents (DaemonSet pattern)

---

## 📚 Nội Dung

- [10.1. Metrics Architecture](./01-metrics-architecture.md) - Metrics Server, cAdvisor, Custom metrics
- [10.2. Logging Architecture](./02-logging-architecture.md) - Stdout/stderr, log aggregation
- [10.3. Labels & Annotations for Observability](./03-labels-annotations-observability.md) - Tagging strategy
- [10.4. Service Discovery & Monitoring](./04-service-discovery-monitoring.md) - Endpoints, DNS
- [10.5. Deploying Monitoring Agents](./05-deploying-monitoring-agents.md) - DaemonSet, RBAC, ServiceAccounts
- [10.6. Events & Audit Logs](./06-events-audit-logs.md) - K8s events, audit trail

---

## 🎓 Tại Sao Cần Học Phần Này?

### Khi triển khai Datadog/Dynatrace, bạn cần hiểu:

**1. Metrics Collection**
```
Datadog Agent cần:
  → Biết Pods nào đang chạy (API Server)
  → Lấy metrics từ kubelet (cAdvisor)
  → Đọc custom metrics (Metrics API)
  → Access resource usage (requests/limits)
```

**2. Log Aggregation**
```
Log collector cần:
  → Đọc container logs (stdout/stderr)
  → Access log files trên Node (hostPath)
  → Parse và enrich với K8s metadata
  → Forward đến backend
```

**3. Auto-discovery**
```
Monitoring tool cần:
  → Discover Services (API Server watch)
  → Detect new Pods (Events)
  → Tag với labels (metadata)
  → Update targets dynamically
```

**4. Permissions**
```
Agent cần RBAC để:
  → List/Watch Pods, Services, Nodes
  → Read metrics từ Metrics API
  → Access logs
  → Create Events (optional)
```

---

## 🔗 Liên Quan Đến Các Phần Khác

- **Phần 2 (Architecture):** API Server, kubelet, kube-proxy
- **Phần 3 (Core Concepts):** Labels, Annotations
- **Phần 4 (Workloads):** DaemonSet (deploy agents)
- **Phần 5 (Networking):** Service discovery
- **Phần 8 (HA):** Metrics-based autoscaling (HPA)

---

## ⏱️ Thời Gian Học

**Ước tính:** 4-5 giờ

Quan trọng cho:
- DevOps Engineers
- SRE (Site Reliability Engineers)
- Platform Engineers
- Ai triển khai monitoring/observability

---

## 🚀 Bắt Đầu

👉 [10.1. Metrics Architecture trong K8s](./01-metrics-architecture.md)

---

[⬅️ Phần 9: Next Steps](../09-next-steps/README.md) | [🏠 Mục Lục Chính](../README.md)

