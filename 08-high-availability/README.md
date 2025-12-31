# 📘 Phần 8: High Availability & Scaling

> Self-healing, health checks, và horizontal scaling

---

## 🎯 Mục Tiêu

✅ **Self-healing** automatic  
✅ **Health checks** (Liveness, Readiness, Startup)  
✅ **Horizontal Pod Autoscaler (HPA)**  
✅ **Production HA** best practices  

---

## 📚 Key Concepts

### Self-Healing
**K8s automatically:**
- Restarts crashed containers
- Replaces failed Pods
- Reschedules on node failure
- Maintains desired state

### Health Probes

**Liveness Probe:** Is container alive?
- Fail → Kill & restart container
- Use: Detect deadlocks, hung processes

**Readiness Probe:** Ready for traffic?
- Fail → Remove from Service (don't kill)
- Use: Warm-up, temporary unavailability

**Startup Probe:** Has started successfully?
- Fail → Keep retrying, then kill if timeout
- Use: Slow-starting containers

---

## 💡 Quick Examples

**Health Checks:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: app
    image: webapp:v1
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 3
```

**HPA (Autoscaling):**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

---

## 🎯 Production Checklist

```yaml
✓ All Pods have readiness probes
✓ All Pods have liveness probes  
✓ Resource requests/limits set
✓ HPA configured (min 2+ replicas)
✓ PodDisruptionBudget for critical apps
✓ Multiple replicas across zones
✓ Monitor autoscaling metrics
```

---

[⬅️ Phần 7](../07-storage/README.md) | [🏠 Mục Lục](../README.md) | [➡️ Phần 9](../09-next-steps/README.md)
