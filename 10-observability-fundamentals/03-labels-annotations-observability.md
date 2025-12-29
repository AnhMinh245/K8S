# 10.3. Labels & Annotations for Observability

> Labeling strategy cho filtering, grouping, và alerting trong monitoring tools

---

## 🎯 Tại Sao Labels Quan Trọng?

**Scenario không có labels chuẩn:**
```
Datadog dashboard:
  • 500 Pods đang chạy
  • Tên random: web-abc123, web-xyz789, api-def456...
  • Question: "Show me CPU của tất cả production frontend Pods"
  • Answer: ❌ Không query được!
```

**Với labels chuẩn:**
```
Query: app:web AND environment:production AND tier:frontend
Result: ✅ Chính xác những Pods cần monitor
```

---

## 🏷️ 1. Standard Label Schema

### K8s Recommended Labels

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    # Application identity
    app.kubernetes.io/name: wordpress          # Component name
    app.kubernetes.io/instance: wordpress-prod # Unique instance
    app.kubernetes.io/version: "5.8.0"         # Version
    app.kubernetes.io/component: frontend      # Architecture tier
    app.kubernetes.io/part-of: blog-platform   # Parent application
    app.kubernetes.io/managed-by: helm         # Management tool
    
    # Custom (shorter)
    app: wordpress
    version: v5.8.0
    environment: production
    tier: frontend
    team: platform
    cost-center: engineering
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: wordpress
      app.kubernetes.io/instance: wordpress-prod
  template:
    metadata:
      labels:
        # Pod labels (must match selector)
        app.kubernetes.io/name: wordpress
        app.kubernetes.io/instance: wordpress-prod
        app.kubernetes.io/version: "5.8.0"
        app.kubernetes.io/component: frontend
        
        # Simpler labels for querying
        app: wordpress
        version: v5.8.0
        environment: production
        tier: frontend
```

---

## 📊 2. Monitoring Tool Tag Mapping

### Datadog Example

**K8s labels → Datadog tags:**

```yaml
# Pod labels
labels:
  app: web
  version: v1.2.3
  environment: production
  tier: frontend
  team: platform
```

**In Datadog:**
```
Tags automatically created:
  • kube_app:web
  • kube_version:v1.2.3
  • kube_environment:production
  • kube_tier:frontend
  • kube_team:platform
  • kube_namespace:default
  • kube_deployment:web
```

**Query in Datadog:**
```
avg:kubernetes.cpu.usage{kube_environment:production,kube_tier:frontend}
```

---

### Dynatrace Example

**K8s labels → Dynatrace tags:**

```yaml
labels:
  app: web
  version: v1.2.3
  environment: production
```

**In Dynatrace:**
```
Automatic tags:
  • [Kubernetes]app: web
  • [Kubernetes]version: v1.2.3
  • [Kubernetes]environment: production
  • [Kubernetes]namespace: default

Management Zones can be created based on these tags
```

---

## 🎨 3. Labeling Best Practices

### ✅ Essential Labels

**Always include these:**

```yaml
labels:
  # 1. Application identifier
  app: web                    # What application?
  
  # 2. Version
  version: v1.2.3             # What version?
  
  # 3. Environment
  environment: production     # Which environment?
  
  # 4. Component/Tier
  component: api              # Which component?
  tier: backend               # Which tier?
  
  # 5. Ownership
  team: platform              # Which team owns?
  
  # 6. Cost tracking
  cost-center: engineering    # Billing/chargeback
  project: project-alpha      # Project code
```

---

### 🎯 Label Naming Conventions

**Good:**
```yaml
labels:
  app: web                    # Short, clear
  environment: prod           # Standard values
  tier: frontend              # Consistent
  team: platform              # Easy to remember
```

**Bad:**
```yaml
labels:
  application-name: web-application-service  # Too long
  env: p                                      # Not clear (p = prod?)
  layer: front-end-tier                       # Inconsistent with others
  owner: team-platform-engineering            # Redundant prefix
```

---

### 🌍 Multi-Environment Strategy

**Consistent labels across environments:**

```yaml
# Development
labels:
  app: web
  environment: dev
  version: v1.2.3-rc1
  tier: frontend

# Staging
labels:
  app: web
  environment: staging
  version: v1.2.3-rc2
  tier: frontend

# Production
labels:
  app: web
  environment: production
  version: v1.2.3
  tier: frontend
```

**Query:**
```
# All environments
app:web

# Production only
app:web AND environment:production

# Compare dev vs prod
app:web by environment
```

---

## 📝 4. Annotations for Configuration

**Labels vs Annotations:**

| Use Case | Labels | Annotations |
|----------|--------|-------------|
| **Querying/Filtering** | ✅ | ❌ |
| **Size limit** | 63 chars | 256 KB |
| **Purpose** | Identifying, grouping | Configuration, metadata |

---

### Monitoring Configuration Annotations

**Prometheus scraping:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
    prometheus.io/scheme: "https"
```

**Datadog checks:**
```yaml
metadata:
  annotations:
    ad.datadoghq.com/check_names: '["nginx"]'
    ad.datadoghq.com/init_configs: '[{}]'
    ad.datadoghq.com/instances: '[{"nginx_status_url": "http://%%host%%:%%port%%/nginx_status"}]'
    ad.datadoghq.com/logs: '[{"source":"nginx","service":"web"}]'
```

**Dynatrace annotations:**
```yaml
metadata:
  annotations:
    metrics.dynatrace.com/scrape: "true"
    metrics.dynatrace.com/port: "9090"
    metrics.dynatrace.com/path: "/metrics"
```

**Metadata annotations:**
```yaml
metadata:
  annotations:
    description: "Production web frontend"
    contact: "team-platform@company.com"
    documentation: "https://wiki.company.com/web-service"
    runbook: "https://runbook.company.com/web-incidents"
    pagerduty: "P123456"
```

---

## 🔔 5. Alerting Strategy with Labels

### Datadog Monitor Example

**Alert on high CPU for production frontend:**

```json
{
  "name": "High CPU Usage - Production Frontend",
  "type": "metric alert",
  "query": "avg(last_5m):avg:kubernetes.cpu.usage{environment:production,tier:frontend} by {pod_name} > 0.8",
  "message": "CPU usage is high on {{pod_name.name}}\nTeam: @team-platform\nEnvironment: production\nRunbook: https://runbook.com/high-cpu",
  "tags": [
    "environment:production",
    "tier:frontend",
    "severity:warning"
  ]
}
```

**Benefits of good labels:**
- ✅ Target specific environments (prod only)
- ✅ Target specific tiers (frontend only)
- ✅ Route alerts to correct team
- ✅ Different thresholds per environment

---

### Multi-Dimensional Alerting

**CPU alert with different thresholds:**

```
Production frontend:
  • Threshold: 80%
  • Severity: Critical
  • Notify: PagerDuty
  • Query: environment:production AND tier:frontend

Production backend:
  • Threshold: 70%  (more CPU-intensive)
  • Severity: Critical
  • Notify: PagerDuty
  • Query: environment:production AND tier:backend

Staging:
  • Threshold: 90%  (less critical)
  • Severity: Warning
  • Notify: Slack only
  • Query: environment:staging
```

---

## 📈 6. Dashboards & Visualizations

### Datadog Dashboard Example

**With proper labels, create dashboards:**

```
Dashboard: "Production Overview"

Row 1: CPU Usage
  • Timeseries: avg:kubernetes.cpu.usage{environment:production} by {app}
  • Top List: top 10 CPU consumers by {pod_name}

Row 2: Memory Usage
  • Heatmap: kubernetes.memory.usage{environment:production} by {app,version}
  • Table: Memory by tier (frontend vs backend vs database)

Row 3: Request Rate
  • Timeseries: sum:http.requests{environment:production} by {app}
  • Change: % increase week-over-week

Row 4: Errors
  • Timeseries: sum:http.errors{environment:production} by {app}
  • Top List: Error rate by service

All queries possible because of consistent labels! ✅
```

---

### Grouping & Filtering

**Group by labels:**
```
# CPU by application
avg:cpu{environment:prod} by {app}

# Memory by tier
avg:memory{environment:prod} by {tier}

# Requests by version (A/B testing)
sum:requests{app:web} by {version}

# Errors by team (accountability)
sum:errors{environment:prod} by {team}
```

---

## 🎯 7. Label Management at Scale

### Label Injection via Admission Controller

**Automatically add labels to all Pods:**

```yaml
# Using Kyverno policy
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
spec:
  rules:
  - name: add-labels
    match:
      resources:
        kinds:
        - Pod
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            +(injected-by): kyverno
            +(cluster): prod-cluster
            +(region): us-west-2
```

**Benefits:**
- ✅ Enforce consistency
- ✅ Add operational labels automatically
- ✅ No developer intervention needed

---

### Label Validation

**Require certain labels:**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: enforce
  rules:
  - name: check-labels
    match:
      resources:
        kinds:
        - Deployment
    validate:
      message: "Labels 'app', 'environment', and 'team' are required"
      pattern:
        metadata:
          labels:
            app: "*"
            environment: "dev|staging|production"
            team: "*"
```

---

## 🎓 Key Takeaways

1. **Labels = Filtering:** Essential for querying metrics/logs
2. **Standard schema:** Use `app.kubernetes.io/*` labels
3. **Essential labels:** app, version, environment, tier, team
4. **Annotations:** For configuration (Prometheus scrape, Datadog checks)
5. **Alerting:** Labels enable targeted alerts
6. **Dashboards:** Consistent labels = useful dashboards
7. **Enforcement:** Use admission controllers to ensure compliance

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. Labels và Annotations khác nhau như thế nào?
2. Essential labels cần có cho observability?
3. Prometheus sử dụng annotations gì để scrape metrics?
4. Làm sao enforce required labels?
5. Label naming conventions nên như thế nào?
6. Multi-environment labeling strategy?

---

## 🚀 Tiếp Theo

👉 [10.4. Service Discovery & Monitoring](./04-service-discovery-monitoring.md)

---

[⬅️ 10.2. Logging](./02-logging-architecture.md) | [⬆️ Phần 10](./README.md) | [🏠 Mục Lục](../README.md)

