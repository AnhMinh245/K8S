# 11.2. Deployment Strategies

> Zero-downtime deployments và progressive delivery

---

## 🎯 Mục Tiêu

- ✅ Hiểu các deployment strategies
- ✅ Implement Blue-Green deployment
- ✅ Implement Canary deployment
- ✅ Automated rollback
- ✅ A/B testing

---

## 🔄 Rolling Deployment (Default)

### Cách Hoạt Động

**K8s default strategy:**
```
Old version: v1 (3 pods)
New version: v2

Step 1: Create 1 v2 pod
  v1: ███ (3 pods)
  v2: █   (1 pod)

Step 2: Terminate 1 v1 pod
  v1: ██  (2 pods)
  v2: █   (1 pod)

Step 3: Create 1 v2 pod
  v1: ██  (2 pods)
  v2: ██  (2 pods)

... repeat until all v2
  v1:     (0 pods)
  v2: ███ (3 pods)
```

### Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # Max pods trên desired (10 + 2 = 12)
      maxUnavailable: 1  # Max pods unavailable (10 - 1 = 9)
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2
        readinessProbe:  # CRITICAL cho rolling update
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

**Pros:**
```
✓ Zero downtime
✓ Built-in K8s
✓ Automatic rollback nếu health check fail
✓ Resource efficient
```

**Cons:**
```
✗ Cả 2 versions chạy cùng lúc
✗ Không control được traffic %
✗ Rollback chậm (phải rolling back)
```

---

## 🔵🟢 Blue-Green Deployment

### Concept

```
Production Traffic
        ↓
    [Router]
        ↓
   BLUE (v1) ← Currently serving traffic
   
   GREEN (v2) ← New version, ready but not serving

After validation:
    [Router]
        ↓
   GREEN (v2) ← Now serving traffic
   
   BLUE (v1) ← Kept for quick rollback
```

### Implementation với Services

**blue-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  labels:
    app: myapp
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: app
        image: myapp:v1
        ports:
        - containerPort: 8080
```

**green-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  labels:
    app: myapp
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: app
        image: myapp:v2  # New version
        ports:
        - containerPort: 8080
```

**service.yaml (switch traffic):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # Change to 'green' to switch
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

### Deployment Process

```bash
# 1. Deploy green (new version)
kubectl apply -f green-deployment.yaml

# 2. Wait for green to be ready
kubectl rollout status deployment/myapp-green

# 3. Test green internally
kubectl port-forward deployment/myapp-green 8080:8080
curl http://localhost:8080/health

# 4. Switch traffic to green
kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"green"}}}'

# 5. Monitor metrics
# If OK: Delete blue
# If NOT OK: Rollback to blue

# Rollback (instant!)
kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"blue"}}}'

# 6. Cleanup old version
kubectl delete deployment myapp-blue
```

**Pros:**
```
✓ Instant rollback (change selector)
✓ Full testing trước khi switch
✓ Zero downtime
✓ Clean separation
```

**Cons:**
```
✗ Double resources (blue + green)
✗ Database migration phức tạp
✗ All-or-nothing switch
```

---

## 🕯️ Canary Deployment

### Concept

```
Gradually shift traffic:

Stage 1: 95% v1, 5% v2
  v1: ███████████████████ (95%)
  v2: █                   (5%)

Stage 2: 90% v1, 10% v2
  v1: ██████████████████  (90%)
  v2: ██                  (10%)

Stage 3: 50% v1, 50% v2
  v1: ██████████          (50%)
  v2: ██████████          (50%)

Stage 4: 0% v1, 100% v2
  v1:                     (0%)
  v2: ████████████████████ (100%)
```

### Implementation với Istio

**VirtualService:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.example.com
  http:
  - match:
    - headers:
        user-agent:
          regex: ".*Chrome.*"  # Optional: specific users
    route:
    - destination:
        host: myapp
        subset: v2
      weight: 10  # 10% traffic to v2
    - destination:
        host: myapp
        subset: v1
      weight: 90  # 90% traffic to v1
```

**DestinationRule:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

### Automated Canary với Flagger

**Install Flagger:**
```bash
kubectl apply -k github.com/fluxcd/flagger//kustomize/istio
```

**Canary resource:**
```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  
  service:
    port: 80
    targetPort: 8080
  
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 5
    
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m
    
    webhooks:
    - name: load-test
      url: http://flagger-loadtester/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 -c 2 http://myapp-canary/"
```

**Workflow:**
```
1. Deploy new version
   ↓
2. Flagger creates canary deployment
   ↓
3. Route 5% traffic to canary
   ↓
4. Run analysis (metrics + webhooks)
   ├─ Success rate > 99%?
   ├─ Latency < 500ms?
   └─ Load test pass?
   ↓
5. If OK: Increase to 10%, 15%, ... 50%
   If NOT OK: Rollback immediately
   ↓
6. Promote canary to primary
```

**Pros:**
```
✓ Gradual rollout
✓ Real user traffic testing
✓ Automated rollback
✓ Risk mitigation
✓ Metrics-driven
```

**Cons:**
```
✗ Cần service mesh (Istio/Linkerd)
✗ Phức tạp hơn
✗ Cần monitoring stack
```

---

## 🧪 A/B Testing

### Concept

**Khác với Canary:**
- **Canary:** Test stability/performance
- **A/B:** Test features/UX

```
User Segment A → Version A (control)
User Segment B → Version B (variant)

Measure: conversion, engagement, etc.
```

### Implementation

**Route by header:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-ab
spec:
  hosts:
  - myapp.example.com
  http:
  # Beta users → v2
  - match:
    - headers:
        x-user-group:
          exact: "beta"
    route:
    - destination:
        host: myapp
        subset: v2
  
  # Default users → v1
  - route:
    - destination:
        host: myapp
        subset: v1
```

**Route by cookie:**
```yaml
- match:
  - headers:
      cookie:
        regex: ".*experiment=variant-b.*"
  route:
  - destination:
      host: myapp
      subset: v2
```

---

## 🎯 Choosing Strategy

| Strategy | Use Case | Complexity | Cost | Rollback Speed |
|----------|----------|------------|------|----------------|
| **Rolling** | Default, most apps | Low | Low | Slow |
| **Blue-Green** | Critical apps, instant rollback | Medium | High (2x) | Instant |
| **Canary** | Risk mitigation, gradual | High | Medium | Fast |
| **A/B Testing** | Feature testing | High | Medium | Fast |

---

## 🎓 Key Takeaways

**1. Rolling Update:**
- Default K8s strategy
- Good cho most cases
- Requires good health checks

**2. Blue-Green:**
- Instant rollback
- Full testing before switch
- Requires 2x resources

**3. Canary:**
- Gradual rollout
- Metrics-driven
- Automated với Flagger

**4. A/B Testing:**
- Feature validation
- User segment routing
- Requires analytics

---

[⬅️ 11.1. Cluster Setup](./01-cluster-setup.md) | [➡️ 11.3. CI/CD](./03-cicd-integration.md) | [🏠 Mục Lục](../README.md)

