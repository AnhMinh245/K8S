# 📘 Phần 11: Production Deployment

> Deploy K8s applications to production

---

## 🎯 Mục Tiêu

✅ **Production cluster setup** (GKE/EKS)  
✅ **CI/CD integration** (GitLab CI, ArgoCD)  
✅ **Security hardening** (RBAC, PSS, Network Policies)  
✅ **Monitoring & Logging** stack  
✅ **Backup & DR** strategies  
✅ **Cost optimization**  
✅ **Troubleshooting** production issues  

---

## 📚 Production Checklist

### Cluster Setup
```yaml
✓ Managed K8s (GKE, EKS, AKS)
✓ Multi-zone for HA
✓ Node pools với autoscaling
✓ Network policies enabled
✓ Private cluster (if possible)
✓ Terraform for IaC
```

### Security
```yaml
✓ RBAC configured
✓ Pod Security Standards enforced
✓ Network Policies implemented
✓ Secrets encrypted at rest
✓ Image scanning (Trivy)
✓ Admission controllers (OPA)
✓ Regular security audits
```

### Observability
```yaml
✓ Metrics: Prometheus + Grafana
✓ Logs: Loki or ELK
✓ Traces: Jaeger
✓ Alerting: AlertManager
✓ Dashboards: Golden signals
✓ SLIs/SLOs defined
```

### CI/CD
```yaml
✓ GitOps (ArgoCD or Flux)
✓ Automated testing
✓ Security scanning
✓ Progressive delivery (canary)
✓ Rollback automation
✓ Image signing/verification
```

### Backup & DR
```yaml
✓ etcd backups (automated)
✓ Application data backups (Velero)
✓ Multi-region strategy
✓ RTO/RPO defined
✓ DR testing regular
```

---

## 🚀 Deployment Strategies

### Rolling Update (Default)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0

# Zero downtime
# Gradual rollout
# Easy rollback
```

### Blue-Green
```
Blue (current): 100% traffic
Green (new): Deploy, test
Switch: Route 100% to Green
Rollback: Switch back to Blue
```

### Canary
```
v1.0: 90% traffic
v2.0: 10% traffic (canary)
Monitor metrics
Gradually increase v2.0 traffic
Full rollout or rollback based on metrics
```

---

## 🏗️ Infrastructure as Code

**Terraform Example:**
```hcl
# GKE Cluster
resource "google_container_cluster" "primary" {
  name     = "production-cluster"
  location = "us-central1"
  
  # Multi-zone for HA
  node_locations = [
    "us-central1-a",
    "us-central1-b",
    "us-central1-c",
  ]
  
  # Network
  network_policy {
    enabled = true
  }
  
  # Workload Identity
  workload_identity_config {
    workload_pool = "${var.project}.svc.id.goog"
  }
}

# Node Pool
resource "google_container_node_pool" "primary_nodes" {
  cluster    = google_container_cluster.primary.name
  node_count = 3
  
  autoscaling {
    min_node_count = 3
    max_node_count = 10
  }
  
  node_config {
    machine_type = "n1-standard-4"
    disk_size_gb = 100
    
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
    
    labels = {
      environment = "production"
    }
  }
}
```

---

## 🔒 Security Hardening

**Pod Security Standards:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Network Policy (Default Deny):**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**RBAC:**
```yaml
# Principle of least privilege
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "deployments", "services"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
  # NO delete in production!
```

---

## 📊 Monitoring Stack

**Prometheus + Grafana:**
```bash
# Install kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values custom-values.yaml

# Includes:
# - Prometheus (metrics)
# - Grafana (dashboards)
# - AlertManager (alerts)
# - Node Exporter (node metrics)
# - kube-state-metrics (K8s metrics)
```

**Essential Dashboards:**
- Cluster overview
- Node metrics
- Pod metrics
- Deployment status
- Ingress traffic
- etcd health

---

## 🔄 GitOps với ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/k8s-manifests
    targetRevision: main
    path: production/webapp
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

---

## 💰 Cost Optimization

**Strategies:**
```yaml
1. RIGHT-SIZING
   ✓ Set appropriate resource requests/limits
   ✓ Monitor actual usage
   ✓ Adjust based on data

2. AUTOSCALING
   ✓ HPA for Pods
   ✓ Cluster Autoscaler for Nodes
   ✓ VPA for resource optimization

3. SPOT/PREEMPTIBLE INSTANCES
   ✓ Use for non-critical workloads
   ✓ Can save 60-80% costs
   ✓ Implement graceful handling

4. RESOURCE CLEANUP
   ✓ Delete unused PVCs
   ✓ Remove old Docker images
   ✓ Clean up failed Jobs

5. MONITORING
   ✓ Kubecost for cost visibility
   ✓ Track costs per namespace/team
   ✓ Set budget alerts
```

---

## 🐛 Production Troubleshooting

**Common Issues:**

```bash
# 1. Pod CrashLoopBackOff
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous

# 2. ImagePullBackOff
kubectl describe pod <pod-name>
# Check imagePullSecrets

# 3. Service not reachable
kubectl get endpoints <service-name>
# Check selector matches Pod labels

# 4. High resource usage
kubectl top pods
kubectl top nodes
# Check HPA status

# 5. Network issues
kubectl exec <pod> -- curl <service>
# Check Network Policies

# 6. PVC pending
kubectl describe pvc <pvc-name>
# Check StorageClass exists

# 7. Node NotReady
kubectl describe node <node-name>
# Check kubelet logs
```

---

## 🎯 Production Day-1 Checklist

```yaml
Before Go-Live:
✓ Load testing completed
✓ Disaster recovery tested
✓ Monitoring/alerting configured
✓ Runbooks documented
✓ On-call rotation established
✓ Backup verified
✓ Security scan passed
✓ Cost estimates reviewed
✓ Stakeholder approval

After Go-Live:
✓ Monitor metrics closely
✓ Check logs for errors
✓ Verify backup running
✓ Test rollback procedure
✓ Document incidents
✓ Collect feedback
✓ Plan improvements
```

---

## 🎉 You're Production Ready!

**Remember:**
- Start small, iterate
- Monitor everything
- Automate toil
- Document learnings
- Share knowledge
- Stay updated

**Good luck with your production deployment! 🚀**

---

[⬅️ Phần 10](../10-observability-fundamentals/README.md) | [🏠 Mục Lục](../README.md)
