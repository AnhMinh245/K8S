# 11.7. Cost Optimization

> Giảm chi phí K8s mà vẫn đảm bảo performance

---

## 🎯 Mục Tiêu

- ✅ Right-size resources
- ✅ Use spot instances
- ✅ Cost monitoring
- ✅ Autoscaling optimization
- ✅ Storage optimization

---

## 📊 Cost Monitoring

### Install Kubecost

```bash
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost \
  --create-namespace \
  --set kubecostToken="<token>"
```

### Key Metrics

```
• Cost per namespace
• Cost per deployment
• Cost per pod
• Idle resource cost
• Recommendations
```

---

## 💰 Resource Right-Sizing

### Analyze Current Usage

```bash
# Check actual usage
kubectl top pods -n production

# Compare với requests/limits
kubectl describe pod <pod> | grep -A 5 "Requests\|Limits"
```

### Vertical Pod Autoscaler (VPA)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"  # or "Recommend"
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2000m
        memory: 2Gi
```

---

## 🎰 Spot Instances

### GKE Spot Node Pool

```hcl
resource "google_container_node_pool" "spot" {
  name    = "spot-pool"
  cluster = google_container_cluster.primary.id
  
  autoscaling {
    min_node_count = 0
    max_node_count = 10
  }
  
  node_config {
    machine_type = "n2-standard-8"
    spot         = true  # 60-91% cheaper!
    
    taint {
      key    = "spot"
      value  = "true"
      effect = "NoSchedule"
    }
  }
}
```

### Tolerate Spot Nodes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-job
spec:
  template:
    spec:
      tolerations:
      - key: "spot"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      
      nodeSelector:
        cloud.google.com/gke-spot: "true"
```

---

## 📦 Storage Optimization

### Use Appropriate Storage Classes

```yaml
# Standard (cheap, slow)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
---
# SSD (expensive, fast)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

### Cleanup Unused PVs

```bash
# Find unbound PVs
kubectl get pv | grep Released

# Delete
kubectl delete pv <pv-name>
```

---

## 🔄 Autoscaling Best Practices

### HPA Configuration

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Not too low!
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scale down
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```

---

## 💡 Cost Optimization Tips

### DO ✅

```
✓ Use spot instances cho non-critical workloads
✓ Right-size resources (VPA)
✓ Set resource requests/limits
✓ Use cluster autoscaler
✓ Delete unused resources
✓ Use appropriate storage classes
✓ Enable HPA
✓ Monitor costs regularly
✓ Use namespace quotas
✓ Schedule batch jobs off-peak
```

### DON'T ❌

```
✗ Over-provision resources
✗ Use SSD for everything
✗ Keep idle resources
✗ Ignore cost alerts
✗ Use :latest tag (causes unnecessary pulls)
✗ Run dev/test 24/7
```

---

## 📈 Cost Savings Examples

**Scenario 1: Right-sizing**
```
Before: 10 pods × 2 CPU × $0.03/hour = $14.40/day
After:  10 pods × 1 CPU × $0.03/hour = $7.20/day
Savings: 50% ($216/month)
```

**Scenario 2: Spot Instances**
```
Before: 10 nodes × $100/month = $1000/month
After:  10 spot nodes × $30/month = $300/month
Savings: 70% ($700/month)
```

**Scenario 3: Cluster Autoscaling**
```
Before: 20 nodes running 24/7 = $2000/month
After:  5-20 nodes (avg 12) = $1200/month
Savings: 40% ($800/month)
```

---

## 🎓 Key Takeaways

**1. Monitor:** Use Kubecost  
**2. Right-size:** VPA recommendations  
**3. Spot:** 60-90% cheaper  
**4. Autoscale:** HPA + Cluster Autoscaler  
**5. Cleanup:** Unused resources  

---

[⬅️ 11.6. Backup](./06-backup-dr.md) | [➡️ 11.8. Troubleshooting](./08-troubleshooting.md) | [🏠 Mục Lục](../README.md)

