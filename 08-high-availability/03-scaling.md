# 8.3. Scaling - Tự Động Mở Rộng

> Auto-scale applications based on demand

---

## 🎯 Scaling Types

### 1. Manual Scaling

```bash
# Scale Deployment
kubectl scale deployment web --replicas=10

# Scale StatefulSet
kubectl scale statefulset mysql --replicas=5
```

**Use case:** Known traffic patterns

---

### 2. Horizontal Pod Autoscaler (HPA)

**Automatically scale number of Pods**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
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
        averageUtilization: 70
```

**Behavior:**
```
CPU < 70% → Scale down (min 2)
CPU > 70% → Scale up (max 10)
```

---

### 3. Vertical Pod Autoscaler (VPA)

**Automatically adjust CPU/memory requests**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Auto"
```

**What it does:**
```
Pod requests:
  cpu: 100m → 500m (increased)
  memory: 128Mi → 512Mi (increased)
```

**⚠️ Warning:** HPA and VPA shouldn't target same metric!

---

### 4. Cluster Autoscaler

**Automatically add/remove Nodes**

```
Scenario: HPA scaled Pods to 50
Current Nodes: Can't fit 50 Pods (insufficient CPU/memory)
  ↓
Cluster Autoscaler adds Nodes
  ↓
50 Pods scheduled on new Nodes ✅
```

**Only on cloud providers:** AWS, GCP, Azure

---

## 📊 HPA Detailed Example

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scale down
      policies:
      - type: Percent
        value: 50  # Max 50% scale down at once
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0  # Scale up immediately
      policies:
      - type: Percent
        value: 100  # Can double replicas
        periodSeconds: 60
```

**Behavior:**
```
Traffic spike:
  CPU: 85% → Scale up immediately (double Pods)
  
Traffic drops:
  CPU: 30% → Wait 5 minutes → Scale down 50% max
```

---

## 🔧 Commands

```bash
# Create HPA
kubectl autoscale deployment web \
  --min=3 --max=10 --cpu-percent=70

# Check HPA status
kubectl get hpa

# Output:
NAME      REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS
web-hpa   Deployment/web   45%/70%   3         10        5

# Describe HPA
kubectl describe hpa web-hpa

# Delete HPA
kubectl delete hpa web-hpa
```

---

## 📈 Metrics for HPA

### 1. Resource Metrics (CPU, Memory)
```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70
```

### 2. Custom Metrics
```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"
```

### 3. External Metrics
```yaml
metrics:
- type: External
  external:
    metric:
      name: queue_depth
      selector:
        matchLabels:
          queue: worker
    target:
      type: AverageValue
      averageValue: "30"
```

---

## 💡 Best Practices

### ✅ DO

1. **Set resource requests** (HPA needs them)
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

2. **Reasonable min/max**
```yaml
minReplicas: 2   # HA
maxReplicas: 50  # Cost control
```

3. **Stabilization window** (avoid flapping)
```yaml
scaleDown:
  stabilizationWindowSeconds: 300
```

4. **Monitor metrics**
```bash
kubectl top pods
```

### ❌ DON'T

1. **No resource requests** → HPA won't work
2. **HPA + VPA on same metric** → Conflict
3. **Too aggressive scaling** → Cost spike
4. **minReplicas: 1** → No HA

---

## 🎓 Key Takeaways

1. **Manual scaling:** kubectl scale
2. **HPA:** Auto-scale Pods based on metrics
3. **VPA:** Auto-adjust resource requests
4. **Cluster Autoscaler:** Add/remove Nodes
5. **Metrics:** CPU, memory, custom metrics
6. **Stabilization:** Prevent flapping
7. **Resource requests:** Required for HPA

---

**Chúc mừng!** Hoàn thành **Phần 8: High Availability** 🎉

**Bạn đã hoàn thành toàn bộ tài liệu Kubernetes!** 🎉🎉🎉

👉 [**Phần 9: Next Steps - Lộ Trình Tiếp Theo**](../09-next-steps/README.md)

---

[⬅️ 8.2. Health Checks](./02-health-checks.md) | [⬆️ Phần 8](./README.md) | [🏠 Mục Lục Chính](../README.md)

