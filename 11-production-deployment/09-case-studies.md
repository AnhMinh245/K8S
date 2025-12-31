# 11.9. Real-World Case Studies

> Kinh nghiệm thực tế từ production deployments

---

## 🎯 Mục Tiêu

- ✅ Học từ real-world scenarios
- ✅ Architecture decisions
- ✅ Challenges và solutions
- ✅ Lessons learned

---

## 🛒 Case Study 1: E-Commerce Platform

### Background

**Company:** Online retailer  
**Scale:** 10M users, 100K orders/day  
**Challenge:** Black Friday traffic (10x normal)  

### Architecture

```
┌─────────────────────────────────────────┐
│         PRODUCTION CLUSTER              │
├─────────────────────────────────────────┤
│                                         │
│  Frontend (React SPA)                   │
│  ├─ Deployment: 10-50 pods (HPA)       │
│  ├─ Ingress: NGINX + cert-manager      │
│  └─ CDN: CloudFlare                    │
│                                         │
│  Backend Services (Microservices)       │
│  ├─ Order Service (10-30 pods)         │
│  ├─ Payment Service (5-20 pods)        │
│  ├─ Inventory Service (5-15 pods)      │
│  └─ Notification Service (3-10 pods)   │
│                                         │
│  Databases                              │
│  ├─ PostgreSQL (StatefulSet)           │
│  ├─ Redis (cache)                      │
│  └─ Elasticsearch (search)             │
│                                         │
│  Message Queue                          │
│  └─ RabbitMQ (StatefulSet)             │
│                                         │
└─────────────────────────────────────────┘
```

### Key Decisions

**1. Microservices Architecture**
```
Why: Scalability, independent deployment
Challenge: Service discovery, distributed tracing
Solution: Istio service mesh
```

**2. Autoscaling Strategy**
```yaml
# HPA based on custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 10
  maxReplicas: 50
  metrics:
  - type: Pods
    pods:
      metric:
        name: orders_per_second
      target:
        type: AverageValue
        averageValue: "100"  # 100 orders/sec per pod
```

**3. Database Strategy**
```
Primary: PostgreSQL (StatefulSet)
Read replicas: 5 replicas
Cache: Redis (80% hit rate)
```

### Black Friday Preparation

**Pre-event:**
```bash
# 1. Pre-scale infrastructure
kubectl scale deployment frontend --replicas=30
kubectl scale deployment order-service --replicas=20

# 2. Warm up cache
# Run cache warming job

# 3. Load testing
# Simulate 10x traffic with k6

# 4. War room setup
# Incident channel, on-call rotation
```

**During Event:**
```
✓ Peak: 150K concurrent users
✓ Autoscaling worked perfectly
✓ 0 downtime
✓ P95 latency < 500ms
✓ 0 critical incidents
```

### Lessons Learned

**What Worked:**
```
✓ HPA with custom metrics
✓ Pre-scaling before event
✓ Extensive load testing
✓ Monitoring dashboards
✓ Runbooks for common issues
```

**What Didn't:**
```
✗ Database connection pool exhausted (fixed: increased pool size)
✗ Redis memory spike (fixed: added memory limits + eviction policy)
✗ Some alerts too noisy (fixed: adjusted thresholds)
```

---

## 🏢 Case Study 2: SaaS Platform Migration

### Background

**Company:** B2B SaaS  
**Migration:** Monolith → Microservices on K8s  
**Timeline:** 6 months  
**Users:** 50K companies, 500K end users  

### Migration Strategy

**Phase 1: Strangler Pattern**
```
Monolith (VM)
    ↓
API Gateway (K8s)
    ├─ Route to Monolith (default)
    └─ Route to new services (gradually)
```

**Phase 2: Service by Service**
```
Month 1-2: Auth Service → K8s
Month 3-4: User Service → K8s
Month 5-6: Core Business Logic → K8s
```

### Architecture

```yaml
# Multi-tenant architecture
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-acme
  labels:
    tenant: acme
    tier: premium
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: tenant-acme
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    pods: "50"
```

### Challenges & Solutions

**Challenge 1: Data Migration**
```
Problem: 500GB database
Solution: 
  • Dual-write (old + new DB)
  • Background migration job
  • Verification scripts
  • Gradual cutover
```

**Challenge 2: Zero Downtime**
```
Solution:
  • Feature flags
  • Canary deployments
  • Instant rollback capability
  • Extensive monitoring
```

**Challenge 3: Multi-tenancy**
```
Solution:
  • Namespace per tenant
  • Resource quotas
  • Network policies
  • Tenant-specific configs
```

### Results

```
✓ Zero downtime during migration
✓ 40% cost reduction (vs VMs)
✓ 3x faster deployments
✓ 99.95% → 99.99% uptime
✓ Better resource utilization
```

---

## 🎮 Case Study 3: Gaming Backend

### Background

**Company:** Mobile game  
**Scale:** 5M DAU (Daily Active Users)  
**Challenge:** Spiky traffic, real-time gameplay  

### Architecture

```
Global Load Balancer
        ↓
┌───────────────────────────────┐
│   Regional Clusters (3)       │
├───────────────────────────────┤
│                               │
│  Game Servers (StatefulSet)   │
│  ├─ Sticky sessions           │
│  ├─ 1000 pods                 │
│  └─ Autoscaling: 500-2000     │
│                               │
│  Matchmaking Service          │
│  ├─ Queue management          │
│  └─ Low latency required      │
│                               │
│  Leaderboard Service          │
│  ├─ Redis Cluster             │
│  └─ High read throughput      │
│                               │
└───────────────────────────────┘
```

### Key Optimizations

**1. Pod Affinity (Sticky Sessions)**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: game-server
spec:
  serviceName: game-server
  replicas: 1000
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - game-server
              topologyKey: kubernetes.io/hostname
```

**2. DaemonSet for Monitoring**
```yaml
# Low-overhead monitoring agent on every node
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: game-metrics
spec:
  template:
    spec:
      containers:
      - name: metrics
        image: game-metrics:v1
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
```

**3. Spot Instances for Non-Critical**
```
Game Servers: On-demand (critical)
Analytics: Spot instances (70% cheaper)
Batch jobs: Spot instances
```

### Incident: Traffic Spike

**Event:** Viral TikTok video → 10x traffic  
**Impact:** Cluster autoscaler couldn't keep up  

**Response:**
```bash
# 1. Emergency scale
kubectl scale statefulset game-server --replicas=2000

# 2. Request more quota from cloud provider

# 3. Enable aggressive autoscaling
kubectl patch hpa game-server-hpa --patch '
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
'
```

**Prevention:**
```
✓ Pre-provisioned spare capacity
✓ Faster autoscaling policies
✓ Better monitoring/alerting
✓ Runbook for traffic spikes
```

---

## 📊 Common Patterns

### Pattern 1: Gradual Rollout

```
All migrations used:
• Feature flags
• Canary deployments
• Extensive monitoring
• Quick rollback capability
```

### Pattern 2: Observability First

```
Before production:
• Metrics collection
• Log aggregation
• Distributed tracing
• Alerting rules
• Dashboards
```

### Pattern 3: Cost Optimization

```
• Right-size resources
• Use spot instances
• Cluster autoscaling
• Namespace quotas
• Regular cost reviews
```

---

## 🎓 Key Lessons

**Technical:**
```
✓ Start with monitoring
✓ Automate everything
✓ Test failure scenarios
✓ Have rollback plans
✓ Use GitOps
```

**Process:**
```
✓ Gradual migrations
✓ Feature flags
✓ War room for big events
✓ Post-mortems
✓ Runbooks
```

**People:**
```
✓ Train team on K8s
✓ On-call rotation
✓ Clear ownership
✓ Documentation
✓ Knowledge sharing
```

---

## 💡 Recommendations

**For Startups:**
```
• Start with managed K8s (GKE/EKS)
• Use simple architecture first
• Focus on core features
• Optimize later
```

**For Enterprises:**
```
• Gradual migration (strangler pattern)
• Multi-cluster for isolation
• Invest in platform team
• Compliance from day 1
```

**For High-Traffic:**
```
• Multi-region deployment
• Aggressive autoscaling
• Extensive load testing
• Chaos engineering
```

---

[⬅️ 11.8. Troubleshooting](./08-troubleshooting.md) | [🏠 Mục Lục](../README.md)

