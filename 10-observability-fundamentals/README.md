# 📘 Phần 10: Observability Fundamentals

> K8s foundations cho monitoring & observability

---

## 🎯 Mục Tiêu

✅ **Metrics architecture** (Metrics Server, cAdvisor)  
✅ **Logging architecture** và patterns  
✅ **Labels & Annotations** cho observability  
✅ **Service Discovery** cho monitoring  
✅ **Deploy monitoring agents** (Datadog, Dynatrace)  
✅ **Events & Audit Logs**  

---

## 📚 Core Concepts

### Metrics trong K8s

**Metrics Server:**
- Collects resource metrics (CPU, RAM)
- Powers `kubectl top`
- Required for HPA

**cAdvisor:**
- Built into kubelet
- Container-level metrics
- Automatic collection

**Custom Metrics:**
- Application metrics (requests/sec, latency)
- Via Prometheus or similar
- Powers advanced HPA

---

### Logging Architecture

**Container Logs:**
```bash
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container>
kubectl logs <pod-name> --previous
```

**Log Collection:**
- DaemonSet pattern (Fluentd, Filebeat)
- Collect from all Nodes
- Ship to central storage (Loki, Elasticsearch)

---

### Labels for Observability

```yaml
metadata:
  labels:
    # Standard labels
    app.kubernetes.io/name: webapp
    app.kubernetes.io/instance: webapp-prod
    app.kubernetes.io/version: "1.2.0"
    app.kubernetes.io/component: backend
    
    # Custom labels
    team: platform
    environment: production
    
  annotations:
    # Prometheus scraping
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
```

---

## 🔍 Service Discovery

**K8s Service Discovery for Monitoring:**

```yaml
# Prometheus ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: webapp-monitor
spec:
  selector:
    matchLabels:
      app: webapp
  endpoints:
  - port: metrics
    interval: 30s
```

---

## 🎮 Deploy Monitoring Agents

### Datadog Agent

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: datadog-agent
spec:
  selector:
    matchLabels:
      app: datadog-agent
  template:
    metadata:
      labels:
        app: datadog-agent
    spec:
      containers:
      - name: agent
        image: datadog/agent:latest
        env:
        - name: DD_API_KEY
          valueFrom:
            secretKeyRef:
              name: datadog-secret
              key: api-key
        - name: DD_SITE
          value: "datadoghq.com"
        - name: DD_LOGS_ENABLED
          value: "true"
        - name: DD_APM_ENABLED
          value: "true"
```

### Dynatrace OneAgent

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dynatrace
---
apiVersion: dynatrace.com/v1beta1
kind: DynaKube
metadata:
  name: dynakube
  namespace: dynatrace
spec:
  apiUrl: https://xxxxx.live.dynatrace.com/api
  tokens: dynatrace-tokens
  oneAgent:
    classicFullStack:
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
```

---

## 📊 Events & Audit Logs

**K8s Events:**
```bash
# View events
kubectl get events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n <namespace>

# Watch events
kubectl get events --watch

# Describe resource includes events
kubectl describe pod <pod-name>
```

**Audit Logs:**
- kube-apiserver logs all API requests
- Configure audit policy
- Ship to SIEM (Splunk, ELK)

---

## 🎯 Observability Stack

**Complete stack example:**

```
┌─────────────────────────────────────┐
│    APPLICATIONS (with metrics)     │
└──────────────┬──────────────────────┘
               │
┌──────────────┴──────────────────────┐
│         COLLECTION LAYER            │
│  ┌────────────┐    ┌─────────────┐ │
│  │ Prometheus │    │  Fluentd    │ │
│  │ (Metrics)  │    │  (Logs)     │ │
│  └────────────┘    └─────────────┘ │
└──────────────┬──────────────────────┘
               │
┌──────────────┴──────────────────────┐
│         STORAGE LAYER               │
│  ┌────────────┐    ┌─────────────┐ │
│  │ Prometheus │    │    Loki     │ │
│  │   TSDB     │    │             │ │
│  └────────────┘    └─────────────┘ │
└──────────────┬──────────────────────┘
               │
┌──────────────┴──────────────────────┐
│      VISUALIZATION LAYER            │
│         ┌──────────┐                │
│         │ Grafana  │                │
│         └──────────┘                │
└─────────────────────────────────────┘
```

---

## 💡 Best Practices

```yaml
1. LABELS: Consistent labeling strategy
   ✓ app.kubernetes.io/* labels
   ✓ Environment, team, version

2. METRICS: Expose application metrics
   ✓ /metrics endpoint
   ✓ Prometheus format
   ✓ Business metrics

3. LOGS: Structured logging
   ✓ JSON format
   ✓ Include context (trace IDs)
   ✓ Log levels (ERROR, WARN, INFO)

4. TRACES: Distributed tracing
   ✓ OpenTelemetry
   ✓ Trace IDs across services
   ✓ Jaeger or similar

5. DASHBOARDS: Actionable dashboards
   ✓ Golden signals (latency, traffic, errors, saturation)
   ✓ SLIs/SLOs
   ✓ Alerts on actionable conditions
```

---

## 🚀 Quick Setup

```bash
# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Test
kubectl top nodes
kubectl top pods

# Install Prometheus (Helm)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack

# Access Grafana
kubectl port-forward svc/prometheus-grafana 3000:80
# Open http://localhost:3000 (admin/prom-operator)
```

---

[⬅️ Phần 9](../09-next-steps/README.md) | [🏠 Mục Lục](../README.md) | [➡️ Phần 11](../11-production-deployment/README.md)
