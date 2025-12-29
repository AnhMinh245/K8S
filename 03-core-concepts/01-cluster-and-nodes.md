# 3.1. Cluster & Node

> Hiểu Cluster và Node - nền tảng của K8s

---

## 🎯 Cluster

**Cluster** = Tập hợp các máy chủ (Nodes) làm việc cùng nhau

```
Cluster "production"
├── Master Nodes (Control Plane)
│   ├── master-1
│   ├── master-2
│   └── master-3
└── Worker Nodes
    ├── worker-1
    ├── worker-2
    ├── worker-3
    └── ...
```

**Ví dụ:** Cluster = Tập đoàn công ty với nhiều văn phòng

---

## 🖥️ Node

**Node** = 1 máy chủ (physical server hoặc VM) trong cluster

### Node Types

**1. Master Node (Control Plane)**
- Chạy các control plane components
- Ra quyết định
- Không chạy application workloads (by default)

**2. Worker Node**
- Chạy application Pods
- Nhận lệnh từ Control Plane
- Báo cáo status

### Node Information

```bash
# List nodes
kubectl get nodes

# Output:
NAME       STATUS   ROLES           AGE   VERSION
master-1   Ready    control-plane   10d   v1.28.0
worker-1   Ready    <none>          10d   v1.28.0
worker-2   Ready    <none>          10d   v1.28.0

# Detailed node info
kubectl describe node worker-1
```

### Node Capacity

```yaml
Capacity:
  cpu:                4      # 4 CPU cores
  memory:             16Gi   # 16 GB RAM
  pods:               110    # Max 110 Pods
  
Allocatable:          # Available for Pods
  cpu:                3800m  # ~3.8 cores (200m for system)
  memory:             15Gi   # ~15 GB
  pods:               110
```

### Node Conditions

```
Ready:              True   # Node healthy
MemoryPressure:     False  # Enough memory
DiskPressure:       False  # Enough disk
PIDPressure:        False  # Enough process IDs
NetworkUnavailable: False  # Network OK
```

---

## 🎓 Key Takeaways

1. **Cluster:** Group of Nodes working together
2. **Master Node:** Control Plane, decision making
3. **Worker Node:** Run application workloads
4. **Node capacity:** CPU, memory, max Pods
5. **Node conditions:** Health status indicators

---

[⬆️ Phần 3](./README.md) | [➡️ 3.2. Pods](./02-pods.md)


