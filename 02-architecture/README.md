# Phần 2: Architecture - Kiến Trúc Kubernetes

> Hiểu sâu về kiến trúc và cách Kubernetes hoạt động

---

## 🎯 Mục Tiêu Học Phần Này

Sau khi hoàn thành phần này, bạn sẽ:
- ✅ Hiểu kiến trúc tổng thể của K8s cluster
- ✅ Nắm vai trò của từng component trong Control Plane
- ✅ Biết cách Worker Node hoạt động
- ✅ Hiểu communication flow giữa các components
- ✅ Có thể troubleshoot vấn đề architecture-level

---

## 📚 Nội Dung

### [2.1. Tổng Quan Kiến Trúc Kubernetes](./01-overview.md)
- High-level architecture
- Cluster là gì
- Master-Worker model
- Communication patterns
- Design principles

### [2.2. Control Plane - Bộ Não Của Cluster](./02-control-plane.md)
- API Server: Gateway của K8s
- etcd: Distributed database
- Scheduler: Resource allocation
- Controller Manager: State reconciliation
- Cloud Controller Manager
- Cách các components tương tác

### [2.3. Worker Node - Nơi Chạy Workload](./03-worker-nodes.md)
- kubelet: Node agent
- kube-proxy: Network proxy
- Container Runtime (Docker, containerd, CRI-O)
- Pod lifecycle
- Node management
- Addons và plugins

---

## 🗺️ Big Picture

```
┌─────────────────────────────────────────────────────────┐
│                  KUBERNETES CLUSTER                     │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │           CONTROL PLANE (Master)               │    │
│  │  ┌──────────┐  ┌────────────┐  ┌───────────┐  │    │
│  │  │   API    │  │  Scheduler │  │Controller │  │    │
│  │  │  Server  │◄─┤            │◄─┤  Manager  │  │    │
│  │  └────▲─────┘  └────────────┘  └───────────┘  │    │
│  │       │                                        │    │
│  │  ┌────▼─────┐                                  │    │
│  │  │   etcd   │  (State storage)                 │    │
│  │  └──────────┘                                  │    │
│  └────────────────────────────────────────────────┘    │
│              ▲                                          │
│              │ (API calls)                              │
│              ▼                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │             WORKER NODES                       │    │
│  │  ┌──────────────┐  ┌──────────────┐           │    │
│  │  │   Node 1     │  │   Node 2     │   ...     │    │
│  │  │ ┌──────────┐ │  │ ┌──────────┐ │           │    │
│  │  │ │ Pod      │ │  │ │ Pod      │ │           │    │
│  │  │ │ ┌──────┐ │ │  │ │ ┌──────┐ │ │           │    │
│  │  │ │ │App   │ │ │  │ │ │App   │ │ │           │    │
│  │  │ │ └──────┘ │ │  │ │ └──────┘ │ │           │    │
│  │  │ └──────────┘ │  │ └──────────┘ │           │    │
│  │  │              │  │              │           │    │
│  │  │  kubelet     │  │  kubelet     │           │    │
│  │  │  kube-proxy  │  │  kube-proxy  │           │    │
│  │  └──────────────┘  └──────────────┘           │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## ⏱️ Thời Gian Học

**Ước tính:** 4-5 giờ

- 2.1 Overview: 1 giờ
- 2.2 Control Plane: 2 giờ (quan trọng nhất)
- 2.3 Worker Nodes: 1-1.5 giờ

---

## 💡 Tips Học Phần Này

1. **Vẽ diagram:** Tự vẽ lại kiến trúc để nhớ lâu
2. **Tương tự thực tế:** Liên hệ với công ty, tổ chức
3. **Không cần nhớ hết:** Hiểu big picture, detail tra khi cần
4. **Hands-on sau:** Phần này là theory, practice sau
5. **Note câu hỏi:** Component nào chưa rõ để research thêm

---

## 🎓 Kiến Thức Trước Khi Bắt Đầu

**Nên biết:**
- Basic networking (IP, port, DNS)
- Client-server model
- REST API concepts
- Database cơ bản

**Không cần biết:**
- Kubernetes commands (sẽ học sau)
- YAML syntax (sẽ học sau)
- Container internals

---

## 🚀 Bắt Đầu

👉 [2.1. Tổng Quan Kiến Trúc](./01-overview.md)

---

[⬅️ Về Mục Lục Chính](../README.md) | [⬅️ Phần 1: Introduction](../01-introduction/README.md)


