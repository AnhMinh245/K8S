# 10.4. Service Discovery & Monitoring

> Monitoring tools tự động discover targets trong dynamic K8s environment

---

## 🎯 Service Discovery Challenge

**Traditional monitoring (static):**
```
Config file:
  targets:
    - host1:9090
    - host2:9090
    - host3:9090

Problem: Servers come and go → Manual config updates ❌
```

**K8s monitoring (dynamic):**
```
Pods created/deleted constantly
IPs change on restart
Scale up/down frequently

→ Need automatic service discovery! ✅
```

---

## 🔍 1. K8s API as Service Discovery

**Monitoring tools watch K8s API:**

```
┌───────────────────────────────────┐
│      K8s API Server               │
│  • Pods list/watch                │
│  • Services list/watch            │
│  • Endpoints list/watch           │
└───────────┬───────────────────────┘
            │ (watch events)
            ↓
┌───────────────────────────────────┐
│  Monitoring Tool                  │
│  (Datadog / Dynatrace / Prometheus) │
│                                   │
│  1. Watch API for changes         │
│  2. Discover new Pods             │
│  3. Start monitoring              │
│  4. Stop when Pod deleted         │
└───────────────────────────────────┘
```

---

## 📡 2. Discovery Methods

### Method 1: Pod Discovery

**Monitor all Pods matching selector:**

```yaml
# Prometheus ServiceMonitor example
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: web-monitor
spec:
  selector:
    matchLabels:
      app: web          # Discover Pods with label app=web
  endpoints:
  - port: metrics       # Port name
    interval: 30s
    path: /metrics
```

**What happens:**
```
1. Prometheus queries K8s API:
   GET /api/v1/pods?labelSelector=app=web

2. K8s returns list of Pods

3. For each Pod:
   - Extract IP: 10.1.1.5
   - Extract port: 9090 (from port name "metrics")
   - Scrape: http://10.1.1.5:9090/metrics

4. Watch for changes:
   - New Pod created → Start monitoring
   - Pod deleted → Stop monitoring
```

---

### Method 2: Service Discovery

**Monitor via Service (more stable):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
spec:
  selector:
    app: web
  ports:
  - name: metrics
    port: 9090
    targetPort: 9090
```

**Benefits:**
```
Service endpoints automatically updated:
  • New Pod added → Endpoint added
  • Pod deleted → Endpoint removed
  • Pod not ready → Endpoint removed
  
Monitoring tool scrapes Service endpoints
```

---

### Method 3: Annotations-Based Discovery

**Datadog Autodiscovery:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  annotations:
    ad.datadoghq.com/check_names: '["nginx"]'
    ad.datadoghq.com/init_configs: '[{}]'
    ad.datadoghq.com/instances: |
      [{
        "nginx_status_url": "http://%%host%%:%%port%%/nginx_status"
      }]
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
      name: http
```

**Datadog Agent discovers:**
```
1. Watch K8s API for Pods

2. Find Pod with annotation: ad.datadoghq.com/check_names

3. Extract config from annotations

4. Resolve templates:
   %%host%% → Pod IP (10.1.1.5)
   %%port%% → Container port (80)

5. Start check:
   http://10.1.1.5:80/nginx_status
```

---

## 🎯 3. Endpoints Object

**K8s Endpoints track ready Pods:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

**Corresponding Endpoints:**
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: web-service  # Same name as Service
subsets:
- addresses:
  - ip: 10.1.1.5
    nodeName: node-1
    targetRef:
      kind: Pod
      name: web-abc123
      namespace: default
  - ip: 10.1.1.6
    nodeName: node-2
    targetRef:
      kind: Pod
      name: web-xyz789
  ports:
  - port: 8080
    protocol: TCP
```

**Monitoring uses Endpoints:**
```bash
# Get Endpoints
kubectl get endpoints web-service

# Monitoring tool queries:
GET /api/v1/namespaces/default/endpoints/web-service

# Scrape each address:
  http://10.1.1.5:8080/metrics
  http://10.1.1.6:8080/metrics
```

---

## 🔄 4. Dynamic Updates

**Scenario: Pod scales up:**

```
T=0: Deployment has 2 Pods
  Endpoints: [10.1.1.5, 10.1.1.6]
  Monitoring: Scraping 2 targets

T=1: kubectl scale deployment web --replicas=3
  New Pod created: web-def456 (10.1.1.7)

T=2: Pod becomes Ready
  Endpoints updated: [10.1.1.5, 10.1.1.6, 10.1.1.7]

T=3: Monitoring tool watches Endpoints
  MODIFIED event received
  Add new target: http://10.1.1.7:9090/metrics
  Now scraping 3 targets ✅
```

---

## 📊 5. DNS-Based Discovery

**K8s DNS for service discovery:**

```
Service: web-service.default.svc.cluster.local

DNS query returns:
  • Service ClusterIP (load balanced)
  • Or Headless Service → All Pod IPs
```

### Headless Service for Direct Pod Access

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None  # Headless
  selector:
    app: web
  ports:
  - port: 9090
```

**DNS returns all Pod IPs:**
```bash
$ nslookup web-headless.default.svc.cluster.local

Name: web-headless.default.svc.cluster.local
Address: 10.1.1.5
Address: 10.1.1.6
Address: 10.1.1.7
```

**Monitoring scrapes each IP individually:**
```
http://10.1.1.5:9090/metrics
http://10.1.1.6:9090/metrics
http://10.1.1.7:9090/metrics
```

---

## 🔑 6. RBAC for Service Discovery

**Monitoring tool needs permissions:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-discovery
rules:
# Discover Pods
- apiGroups: [""]
  resources:
  - pods
  verbs: ["get", "list", "watch"]

# Discover Services
- apiGroups: [""]
  resources:
  - services
  - endpoints
  verbs: ["get", "list", "watch"]

# Discover Nodes
- apiGroups: [""]
  resources:
  - nodes
  verbs: ["get", "list", "watch"]

# Read config (ConfigMaps with monitoring config)
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]

# Optional: Custom resources (ServiceMonitor, etc.)
- apiGroups: ["monitoring.coreos.com"]
  resources:
  - servicemonitors
  - podmonitors
  verbs: ["get", "list", "watch"]
```

---

## 🎯 7. Monitoring Target Selection

### Label-Based Filtering

**Only monitor production Pods:**

```yaml
# Prometheus scrape config
scrape_configs:
- job_name: 'kubernetes-pods'
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  # Only scrape Pods with label environment=production
  - source_labels: [__meta_kubernetes_pod_label_environment]
    regex: production
    action: keep
  # Only scrape if annotation prometheus.io/scrape=true
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    regex: "true"
    action: keep
```

---

### Port Selection

**Multiple ports on Pod:**

```yaml
spec:
  containers:
  - name: app
    ports:
    - name: http
      containerPort: 8080
    - name: metrics
      containerPort: 9090
    - name: health
      containerPort: 8081
```

**Scrape specific port:**
```yaml
# Prometheus: Scrape port named "metrics"
endpoints:
- port: metrics  # Port name

# Or by annotation
annotations:
  prometheus.io/port: "9090"  # Port number
```

---

## 📈 8. Monitoring Multiple Clusters

**Multi-cluster observability:**

```
┌──────────────────────────────────────────┐
│     Central Monitoring (Datadog SaaS)    │
└───────┬──────────────────────────────────┘
        │
   ┌────┴────┬──────────┬──────────┐
   │         │          │          │
┌──▼──┐  ┌──▼──┐   ┌──▼──┐   ┌───▼──┐
│Cluster│ │Cluster│ │Cluster│ │Cluster│
│ US   │ │ EU    │ │ ASIA  │ │ DEV  │
└──────┘ └───────┘ └───────┘ └──────┘

Each cluster:
  • Datadog Agent (DaemonSet)
  • Cluster Agent (Deployment)
  • Auto-discover local Pods
  • Report to central Datadog
  • Tagged with: cluster, region
```

**Query across clusters:**
```
avg:kubernetes.cpu.usage{cluster:*} by {cluster}

Filter by cluster:
  cluster:us-prod
  cluster:eu-prod
  cluster:asia-prod
```

---

## 💡 Best Practices

### ✅ DO

**1. Use annotations for opt-in monitoring**
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9090"
  prometheus.io/path: "/metrics"
```

**2. Name ports for easy discovery**
```yaml
ports:
- name: http
  containerPort: 8080
- name: metrics
  containerPort: 9090
```

**3. Implement readiness probes**
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080

# Not ready → Removed from Endpoints → Not monitored
```

**4. Label consistently**
```yaml
labels:
  app: web
  environment: production
  tier: frontend

# Easy to filter monitoring targets
```

**5. Use ServiceMonitors (if using Prometheus Operator)**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: web-monitor
spec:
  selector:
    matchLabels:
      app: web
  endpoints:
  - port: metrics
```

---

### ❌ DON'T

1. **Monitor every Pod** → Too much data, cost
2. **No labels** → Can't filter
3. **Change ports frequently** → Breaks discovery
4. **No readiness probe** → Monitor unhealthy Pods
5. **Hardcode IPs** → Pods have dynamic IPs!

---

## 🎓 Key Takeaways

1. **Service discovery:** Automatic target discovery via K8s API
2. **Endpoints:** Track ready Pods behind Service
3. **Annotations:** Configure monitoring per Pod/Service
4. **Labels:** Filter monitoring targets
5. **RBAC:** Monitoring needs list/watch permissions
6. **Dynamic:** Monitoring adapts as Pods scale
7. **DNS:** Alternative discovery method (Headless Service)

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. Monitoring tools discover targets như thế nào?
2. Endpoints object dùng để làm gì?
3. Annotations nào dùng cho Prometheus discovery?
4. Tại sao cần readiness probe cho monitoring?
5. RBAC permissions cần cho service discovery?
6. Headless Service khác gì normal Service?

---

## 🚀 Tiếp Theo

👉 [10.5. Deploying Monitoring Agents](./05-deploying-monitoring-agents.md)

---

[⬅️ 10.3. Labels](./03-labels-annotations-observability.md) | [⬆️ Phần 10](./README.md) | [🏠 Mục Lục](../README.md)

