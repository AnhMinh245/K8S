# 3.2. Pod - Đơn Vị Cơ Bản Nhất

> Pod là building block cơ bản nhất trong Kubernetes

---

## 🎯 Pod Là Gì?

**Pod** = Smallest deployable unit, chứa 1 hoặc nhiều containers

```
┌─────────────────────────────┐
│          Pod                │
│  ┌──────────────────────┐   │
│  │   Main Container     │   │ ← App chính
│  │   (nginx)            │   │
│  └──────────────────────┘   │
│                             │
│  ┌──────────────────────┐   │
│  │ Sidecar Container    │   │ ← Helper (optional)
│  │ (log collector)      │   │
│  └──────────────────────┘   │
│                             │
│  Shared:                    │
│   • Network (same IP)       │
│   • Storage volumes         │
│   • IPC namespace           │
└─────────────────────────────┘
```

---

## 🏢 Ví Dụ Thực Tế

**Pod = Chiếc xe buýt**

```
Xe buýt (Pod):
  ├─ Tài xế (Main container - App chính)
  ├─ Phụ xe (Sidecar container - Logging/Monitoring)
  └─ Shared space (Network, volumes)

Đặc điểm:
  • Cùng di chuyển (deployed together)
  • Cùng lúc hoạt động (lifecycle linked)
  • Chia sẻ không gian (same IP, volumes)
```

---

## 🎨 Single vs Multi-Container Pod

### Single Container Pod (Phổ biến nhất)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
```

**Use case:** 99% cases - 1 Pod chạy 1 app

### Multi-Container Pod (Sidecar Pattern)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logging
spec:
  containers:
  - name: web-app
    image: nginx:1.21
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  
  - name: log-collector
    image: fluent/fluentd
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  
  volumes:
  - name: logs
    emptyDir: {}
```

**Use case:** Container phụ hỗ trợ container chính

**Patterns:**
- **Sidecar:** Helper container (logging, monitoring)
- **Ambassador:** Proxy to external services
- **Adapter:** Transform output

---

## 🔄 Pod Lifecycle

```
┌─────────┐
│ Pending │ ← Pod created, waiting to be scheduled
└────┬────┘
     │
     ▼
┌─────────┐
│ Running │ ← Pod scheduled, containers running
└────┬────┘
     │
     ├─────────────┐
     │             │
     ▼             ▼
┌──────────┐  ┌─────────┐
│Succeeded │  │ Failed  │ ← Job/CronJob completed
└──────────┘  └─────────┘
     │             │
     ▼             ▼
┌──────────────────────┐
│    Terminated        │
└──────────────────────┘
```

### Pod Phases

| Phase | Mô Tả |
|-------|-------|
| **Pending** | Pod created, not scheduled yet or pulling images |
| **Running** | At least 1 container running |
| **Succeeded** | All containers terminated successfully |
| **Failed** | Containers terminated, at least 1 failed |
| **Unknown** | Can't determine state (e.g., Node lost) |

### Container States

```yaml
State:
  Running:        # Container is running
    startedAt: 2024-01-01T10:00:00Z
    
  Waiting:        # Container not running yet
    reason: ImagePullBackOff
    
  Terminated:     # Container finished
    exitCode: 0
    reason: Completed
```

---

## 🔧 Init Containers

**Init containers** chạy TRƯỚC main containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z database:3306; do sleep 2; done']
  
  - name: setup-config
    image: busybox
    command: ['sh', '-c', 'cp /config/* /app/config/']
    volumeMounts:
    - name: config
      mountPath: /app/config
  
  containers:
  - name: app
    image: myapp:1.0
```

**Use cases:**
- Wait for services (database, cache)
- Initialize configuration
- Clone git repo
- Database migrations

---

## 📊 Pod Networking

```
Pod: IP = 10.1.1.5

Container 1 in Pod:
  - localhost:8080 (nginx)

Container 2 in Pod:
  - localhost:9090 (metrics exporter)

Communication:
  - Within Pod: localhost
  - Between Pods: Pod IP directly
  - To Services: Service DNS/IP
```

**Ví dụ:**
```bash
# Container 1 can call Container 2
curl localhost:9090/metrics

# Other Pods can call this Pod
curl 10.1.1.5:8080
```

---

## 💡 Pod Best Practices

### ✅ DO

1. **Stateless:** Pod có thể bị xóa/tạo lại bất cứ lúc nào
2. **Single responsibility:** 1 Pod = 1 application concern
3. **Health checks:** Always define liveness/readiness probes
4. **Resource limits:** Set CPU/memory requests and limits
5. **Use controllers:** Don't create Pods directly, use Deployment/StatefulSet

### ❌ DON'T

1. **Multiple apps in 1 Pod:** Unless they're tightly coupled
2. **Store data in Pod:** Use volumes/PV instead
3. **SSH into Pod:** Use `kubectl exec` for debugging
4. **Long-lived Pods:** Treat Pods as cattle, not pets
5. **Manual scaling:** Use HPA instead

---

## 🎓 Key Takeaways

1. **Pod = Smallest unit:** Contains 1+ containers
2. **Shared resources:** Same IP, volumes, IPC
3. **Ephemeral:** Can be deleted/recreated anytime
4. **Lifecycle:** Pending → Running → Terminated
5. **Init containers:** Run before main containers
6. **Single responsibility:** Keep it simple
7. **Use controllers:** Deployment/StatefulSet, not bare Pods

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. Pod khác gì với Container?
2. Khi nào dùng multi-container Pod?
3. Init containers dùng để làm gì?
4. Tại sao Pod là "ephemeral"?
5. Containers trong cùng Pod communicate như thế nào?

---

[⬅️ 3.1. Cluster](./01-cluster-and-nodes.md) | [⬆️ Phần 3](./README.md) | [➡️ 3.3. Namespace](./03-namespaces.md)


