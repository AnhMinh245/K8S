# 10.1. Metrics Architecture trong Kubernetes

> Hiểu cách K8s expose metrics và monitoring tools collect chúng

---

## 🎯 K8s Metrics Ecosystem

```
┌─────────────────────────────────────────────────────┐
│                Application Pods                     │
│  ┌──────────────┐  ┌──────────────┐                │
│  │ App metrics  │  │ App metrics  │                │
│  │ :8080/metrics│  │ :9090/metrics│                │
│  └──────────────┘  └──────────────┘                │
└─────────────────────────────────────────────────────┘
              ↓ (scrape)                ↑ (expose)
┌─────────────────────────────────────────────────────┐
│          Monitoring Agents (DaemonSet)              │
│  • Datadog Agent    • Dynatrace OneAgent           │
│  • Prometheus Node Exporter                        │
└─────────────────────────────────────────────────────┘
              ↓ (collect)               ↑ (read)
┌─────────────────────────────────────────────────────┐
│              K8s Metrics Sources                    │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │  kubelet    │  │ Metrics      │  │  cAdvisor │ │
│  │  (API)      │  │  Server      │  │  (built-in)│ │
│  └─────────────┘  └──────────────┘  └───────────┘ │
└─────────────────────────────────────────────────────┘
              ↓                         ↑
┌─────────────────────────────────────────────────────┐
│              K8s API Server                         │
│  • Resource metrics (CPU, memory usage)            │
│  • Pod/Node metadata                               │
│  • Events                                          │
└─────────────────────────────────────────────────────┘
```

---

## 📊 1. Metrics Server

**Official K8s component for resource metrics**

### Cài Đặt

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Chức Năng

```
Metrics Server:
  → Collects resource metrics từ kubelet
  → CPU usage, Memory usage per Pod/Node
  → Exposes qua Metrics API
  → Dùng bởi kubectl top và HPA
```

### Kiểm Tra

```bash
# Pod metrics
kubectl top pods

# Output:
NAME         CPU(cores)   MEMORY(bytes)
web-abc123   10m          128Mi
web-xyz789   15m          256Mi

# Node metrics
kubectl top nodes

# Output:
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-1   500m         25%    4Gi             50%
node-2   800m         40%    6Gi             75%
```

### API Access

```bash
# Metrics API endpoint
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/default/pods/web-abc123
```

**Output structure:**
```json
{
  "metadata": {
    "name": "web-abc123",
    "namespace": "default"
  },
  "timestamp": "2024-01-15T10:30:00Z",
  "window": "30s",
  "containers": [
    {
      "name": "nginx",
      "usage": {
        "cpu": "10m",
        "memory": "128Mi"
      }
    }
  ]
}
```

---

## 🔍 2. cAdvisor (Container Advisor)

**Built-in kubelet component**

### Chức Năng

```
cAdvisor:
  → Embedded trong kubelet
  → Monitors containers trên Node
  → Collects:
    • CPU usage
    • Memory usage
    • Network I/O
    • Disk I/O
    • Filesystem usage
```

### Metrics Endpoint

```bash
# cAdvisor metrics từ kubelet
https://<node-ip>:10250/metrics/cadvisor

# Require authentication (kubelet certificate)
```

### Sample Metrics

```
# CPU usage
container_cpu_usage_seconds_total{container="nginx",pod="web-abc123"}

# Memory usage
container_memory_working_set_bytes{container="nginx",pod="web-abc123"}

# Network received bytes
container_network_receive_bytes_total{pod="web-abc123"}

# Filesystem usage
container_fs_usage_bytes{container="nginx",pod="web-abc123"}
```

---

## 📈 3. Resource Metrics

### Resource Requests & Limits

**Declared in Pod spec:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    resources:
      requests:      # ← Monitoring tools track this
        cpu: 100m
        memory: 256Mi
      limits:        # ← And this
        cpu: 500m
        memory: 512Mi
```

**Why important for observability:**

```
Requests & Limits → Baseline for alerting

Example alerts:
  • CPU usage > 80% of limit → Alert
  • Memory usage > 90% of limit → Critical
  • Actual usage << requests → Over-provisioned (waste)
```

---

### HPA Metrics

**Horizontal Pod Autoscaler reads metrics:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # ← From Metrics Server
```

**Monitoring tools observe HPA:**
```
Datadog/Dynatrace can:
  → Monitor HPA scaling events
  → Track actual vs desired replicas
  → Correlate scaling with metrics
  → Alert on scaling issues
```

---

## 🎯 4. Custom Metrics

### Custom Metrics API

**For application-specific metrics:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second  # ← Custom metric
      target:
        type: AverageValue
        averageValue: "1000"
```

### Implementation

**Option 1: Prometheus Adapter**
```
Application exposes metrics → Prometheus scrapes
                           → Prometheus Adapter
                           → K8s Custom Metrics API
                           → HPA consumes
```

**Option 2: Datadog Cluster Agent**
```
Datadog Cluster Agent:
  → Implements Custom Metrics API
  → Queries Datadog backend
  → Exposes to K8s HPA
```

---

## 🔌 5. Monitoring Agent Access Patterns

### Datadog Agent Example

**Architecture:**
```
┌──────────────────────────────────────┐
│      Datadog Agent (DaemonSet)       │
│  ┌────────────────────────────────┐  │
│  │ 1. Watch K8s API               │  │
│  │    • List/Watch Pods           │  │
│  │    • List/Watch Services       │  │
│  │    • Read Node info            │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │ 2. Collect from kubelet        │  │
│  │    • cAdvisor metrics          │  │
│  │    • Pod metrics               │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │ 3. Scrape application metrics  │  │
│  │    • Auto-discovery            │  │
│  │    • Service annotations       │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │ 4. Enrich with K8s metadata    │  │
│  │    • Pod labels → tags         │  │
│  │    • Namespace, deployment     │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### Required Permissions (RBAC)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: datadog-agent
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/metrics    # ← Metrics from nodes
  - nodes/stats      # ← cAdvisor stats
  - pods
  - services
  - endpoints
  verbs: ["get", "list", "watch"]

- apiGroups: [""]
  resources:
  - nodes/proxy      # ← Access kubelet API
  verbs: ["get"]

- apiGroups: ["metrics.k8s.io"]
  resources:
  - pods             # ← Metrics Server API
  - nodes
  verbs: ["get", "list"]
```

---

## 📊 6. Metrics Enrichment

### K8s Metadata → Monitoring Tags

**Pod definition:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-abc123
  namespace: production
  labels:
    app: web
    version: v1.2.3
    tier: frontend
    environment: prod
  annotations:
    team: platform
    cost-center: engineering
spec:
  nodeName: node-1
  # ...
```

**Monitoring agent enriches metrics:**
```
Metric: http_requests_total = 1000

Enriched with K8s metadata:
  • pod_name: web-abc123
  • namespace: production
  • app: web
  • version: v1.2.3
  • tier: frontend
  • environment: prod
  • node: node-1
  • cluster: prod-cluster
  • team: platform (from annotation)

Query in Datadog/Dynatrace:
  "Show http_requests_total where environment=prod AND tier=frontend"
```

---

## 🎯 7. Best Practices for Observability

### ✅ DO

**1. Always set resource requests/limits**
```yaml
resources:
  requests:
    cpu: 100m      # Baseline for monitoring
    memory: 256Mi
  limits:
    cpu: 500m      # Alert threshold
    memory: 512Mi
```

**2. Use consistent labels**
```yaml
metadata:
  labels:
    app: web                    # Application name
    version: v1.2.3             # Version
    environment: production     # Environment
    component: api              # Component role
```

**3. Add annotations for monitoring config**
```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
    ad.datadoghq.com/check_names: '["nginx"]'
```

**4. Deploy Metrics Server**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**5. Enable kubelet authentication**
```yaml
# kubelet config
authentication:
  webhook:
    enabled: true
authorization:
  mode: Webhook
```

---

### ❌ DON'T

1. **No resource requests** → Can't calculate utilization %
2. **Random labels** → Hard to query/aggregate
3. **No Metrics Server** → kubectl top won't work, HPA fails
4. **Block kubelet metrics** → Monitoring agents can't collect
5. **Too many metrics** → Cost and performance impact

---

## 🔍 8. Debugging Metrics Collection

### Check Metrics Server

```bash
# Is Metrics Server running?
kubectl get deployment metrics-server -n kube-system

# Metrics Server logs
kubectl logs -n kube-system deployment/metrics-server

# Test Metrics API
kubectl top nodes
kubectl top pods
```

### Check kubelet Metrics

```bash
# On Node (SSH or debug container)
curl -k https://localhost:10250/metrics/cadvisor \
  --cert /var/lib/kubelet/pki/kubelet-client-current.pem \
  --key /var/lib/kubelet/pki/kubelet-client-current.pem

# Should return Prometheus-format metrics
```

### Check Monitoring Agent

```bash
# Datadog Agent status
kubectl exec -it <datadog-agent-pod> -n datadog -- agent status

# Dynatrace OneAgent logs
kubectl logs -n dynatrace <oneagent-pod>

# Check RBAC permissions
kubectl auth can-i list pods --as=system:serviceaccount:datadog:datadog-agent
```

---

## 🎓 Key Takeaways

1. **Metrics Server:** Official K8s metrics (CPU, memory) cho kubectl top và HPA
2. **cAdvisor:** Built-in kubelet, container metrics detailed
3. **Resource metrics:** requests/limits = baseline cho monitoring
4. **Custom Metrics API:** Application metrics cho HPA
5. **Monitoring agents:** DaemonSet, cần RBAC, access kubelet + API Server
6. **Metadata enrichment:** Labels/annotations → monitoring tags
7. **Best practice:** Consistent labels, resource limits, Metrics Server deployed

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. Metrics Server khác gì với cAdvisor?
2. Monitoring agent cần RBAC permissions gì?
3. Tại sao resource requests/limits quan trọng cho monitoring?
4. Custom Metrics API dùng để làm gì?
5. Labels và annotations được monitoring tools dùng như thế nào?
6. kubelet expose metrics ở endpoint nào?

---

## 🚀 Tiếp Theo

👉 [10.2. Logging Architecture](./02-logging-architecture.md)

Tìm hiểu cách K8s handles logs và monitoring tools collect chúng

---

[⬆️ Phần 10: Observability](./README.md) | [🏠 Mục Lục Chính](../README.md)

