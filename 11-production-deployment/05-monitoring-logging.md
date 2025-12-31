# 11.5. Monitoring & Logging Stack

> Complete observability cho production K8s

---

## 🎯 Mục Tiêu

- ✅ Setup Prometheus + Grafana
- ✅ Log aggregation
- ✅ Distributed tracing
- ✅ Alerting
- ✅ SLI/SLO/SLA

---

## 📊 Prometheus + Grafana

### Install kube-prometheus-stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

### Custom ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Key Metrics

**Golden Signals:**
```promql
# Latency (p95)
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket[5m])
)

# Traffic (requests/sec)
rate(http_requests_total[1m])

# Errors (error rate)
rate(http_requests_total{status=~"5.."}[1m])
/
rate(http_requests_total[1m])

# Saturation (CPU usage)
avg(rate(container_cpu_usage_seconds_total[5m])) by (pod)
```

---

## 📝 Logging với Loki

### Install Loki Stack

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace logging \
  --create-namespace \
  --set promtail.enabled=true \
  --set grafana.enabled=false
```

### Query Logs

```logql
# All logs from namespace
{namespace="production"}

# Error logs
{namespace="production"} |= "error"

# JSON parsing
{app="myapp"} | json | level="error"

# Rate of errors
rate({app="myapp"} |= "error" [5m])
```

---

## 🔍 Distributed Tracing với Jaeger

### Install Jaeger Operator

```bash
kubectl create namespace observability
kubectl create -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.49.0/jaeger-operator.yaml -n observability
```

### Jaeger Instance

```yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
  namespace: observability
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
```

---

## 🚨 Alerting

### PrometheusRule

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: monitoring
spec:
  groups:
  - name: myapp
    interval: 30s
    rules:
    # High error rate
    - alert: HighErrorRate
      expr: |
        rate(http_requests_total{status=~"5.."}[5m])
        /
        rate(http_requests_total[5m])
        > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate on {{ $labels.pod }}"
        description: "Error rate is {{ $value | humanizePercentage }}"
    
    # High latency
    - alert: HighLatency
      expr: |
        histogram_quantile(0.95,
          rate(http_request_duration_seconds_bucket[5m])
        ) > 1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High latency on {{ $labels.pod }}"
    
    # Pod down
    - alert: PodDown
      expr: up{job="myapp"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is down"
```

### AlertManager Config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
    
    route:
      group_by: ['alertname', 'cluster']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'pagerduty'
      
      routes:
      - match:
          severity: critical
        receiver: 'pagerduty'
        continue: true
      
      - match:
          severity: warning
        receiver: 'slack'
    
    receivers:
    - name: 'pagerduty'
      pagerduty_configs:
      - service_key: '<key>'
    
    - name: 'slack'
      slack_configs:
      - api_url: '<webhook>'
        channel: '#alerts'
```

---

## 📈 SLI/SLO/SLA

### Service Level Indicators (SLI)

```yaml
# Availability SLI
sli_availability:
  expr: |
    sum(rate(http_requests_total{status!~"5.."}[30d]))
    /
    sum(rate(http_requests_total[30d]))

# Latency SLI (p95 < 500ms)
sli_latency:
  expr: |
    histogram_quantile(0.95,
      rate(http_request_duration_seconds_bucket[30d])
    ) < 0.5
```

### Service Level Objectives (SLO)

```
Availability SLO: 99.9% (3 nines)
  = 43.2 minutes downtime/month

Latency SLO: 95% of requests < 500ms
```

### Error Budget

```
Error Budget = 1 - SLO
  = 1 - 0.999
  = 0.001
  = 0.1% of requests can fail

Monthly requests: 100M
Error budget: 100K failed requests
```

---

## 🎓 Key Takeaways

**1. Metrics:** Prometheus + Grafana  
**2. Logs:** Loki + Promtail  
**3. Traces:** Jaeger  
**4. Alerts:** AlertManager  
**5. SLO:** Define và track  

---

[⬅️ 11.4. Security](./04-security-hardening.md) | [➡️ 11.6. Backup](./06-backup-dr.md) | [🏠 Mục Lục](../README.md)

