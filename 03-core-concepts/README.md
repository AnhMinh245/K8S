# 📘 Phần 3: Core Concepts - Khái Niệm Cốt Lõi

> Các building blocks cơ bản của Kubernetes

---

## 🎯 Mục Tiêu Phần Này

Sau khi hoàn thành Phần 3, bạn sẽ:

✅ **Setup được** local Kubernetes cluster  
✅ **Tạo và quản lý** Pods  
✅ **Organize resources** với Namespaces  
✅ **Query và select** với Labels  
✅ **Hands-on practice** với kubectl  

---

## 📚 Nội Dung

### [3.1. Cluster và Nodes](./01-cluster-and-nodes.md) ⭐⭐⭐⭐

**Thời gian:** 45-60 phút

**Nội dung:**
- Cluster là gì
- Master Node vs Worker Node
- Node components và anatomy
- **Setup local cluster** (Minikube/kind/Docker Desktop)
- Node management (labels, taints, conditions)
- kubectl basics để work với Nodes

**Hands-on:**
```bash
# Setup
minikube start

# Explore
kubectl get nodes
kubectl describe node minikube

# Add labels
kubectl label nodes minikube environment=dev
```

**Key Points:**
```
✓ Cluster = Tập hợp Nodes
✓ Master Node = Control Plane
✓ Worker Node = Run applications
✓ Setup local cluster để practice
```

---

### [3.2. Pods - Đơn Vị Cơ Bản](./02-pods.md) ⭐⭐⭐⭐⭐

**Thời gian:** 75-90 phút (CỰC KỲ QUAN TRỌNG!)

**Nội dung:**
- Pod là gì và **TẠI SAO cần Pod** (không chỉ container)
- Single-container vs Multi-container Pods
- Pod lifecycle (Pending → Running → Succeeded/Failed)
- Health checks (Liveness, Readiness, Startup probes)
- **Pod patterns** (Sidecar, Ambassador, Adapter)
- **Troubleshooting** Pods (CrashLoopBackOff, ImagePullBackOff, etc.)

**Hands-on:**
```bash
# Create Pod
kubectl run nginx --image=nginx

# Get Pods
kubectl get pods -o wide

# Logs
kubectl logs nginx

# Execute commands
kubectl exec -it nginx -- /bin/bash

# Port forward
kubectl port-forward nginx 8080:80

# Delete
kubectl delete pod nginx
```

**Key Points:**
```
✓ Pod = Wrapper cho containers
✓ Shared network, volumes, lifecycle
✓ Health probes = Production必 requirement
✓ Multi-container Pods cho sidecar pattern
```

---

### [3.3. Namespaces - Phân Vùng](./03-namespaces.md) ⭐⭐⭐⭐

**Thời gian:** 60-75 phút

**Nội dung:**
- Namespaces là gì và **TẠI SAO cần**
- Default namespaces (default, kube-system, etc.)
- **Organization patterns** (by env, team, app)
- **Resource Quotas** (limit resources per namespace)
- **LimitRange** (default limits cho containers)
- Cross-namespace communication

**Hands-on:**
```bash
# Create namespace
kubectl create ns dev

# Create resources in namespace
kubectl run nginx --image=nginx -n dev

# Set default namespace
kubectl config set-context --current --namespace=dev

# Resource quota
kubectl apply -f resource-quota.yaml

# Cross-namespace access
# Service: backend.production.svc.cluster.local
```

**Key Points:**
```
✓ Namespaces = Logical isolation
✓ Organize by environment/team/app
✓ Resource Quotas prevent starvation
✓ LimitRange sets defaults
✓ Cross-NS access: service.namespace
```

---

### [3.4. Labels & Selectors](./04-labels-selectors.md) ⭐⭐⭐⭐⭐

**Thời gian:** 60-75 phút

**Nội dung:**
- Labels là gì và **TẠI SAO cần**
- Labeling strategies (multi-dimensional, hierarchical)
- **Selectors** (equality-based, set-based)
- Labels trong **Services, Deployments**
- **Labels vs Annotations**
- Best practices

**Hands-on:**
```bash
# Add labels
kubectl label pods nginx tier=frontend

# Query by labels
kubectl get pods -l tier=frontend
kubectl get pods -l 'environment in (prod,staging)'

# Show labels
kubectl get pods --show-labels

# Update labels
kubectl label pods nginx tier=backend --overwrite

# Remove labels
kubectl label pods nginx tier-
```

**Key Points:**
```
✓ Labels = Key-value identification
✓ Selectors = Query language
✓ Used by Services, Deployments
✓ Multi-dimensional labeling
✓ Annotations for metadata
```

---

## 🗺️ Learning Path

### Recommended Order

```
1. Start: README.md (file này)
   ↓
2. 3.1. Cluster & Nodes (setup cluster)
   ↓
3. 3.2. Pods (most important!)
   ↓
4. 3.3. Namespaces (organize)
   ↓
5. 3.4. Labels (query)
   ↓
6. Practice, practice, practice!
   ↓
7. Next: Phần 4 - Workloads
```

### Cách Học Hiệu Quả

**1. Setup First**
```
✓ Install minikube/kind/Docker Desktop
✓ Verify kubectl works
✓ Get comfortable với terminal
```

**2. Type Every Command**
```
✓ Don't copy-paste blindly
✓ Type commands to build muscle memory
✓ Experiment với different flags
✓ Break things và fix them!
```

**3. Practice Scenarios**
```
✓ Create Pods trong different namespaces
✓ Label và query Pods
✓ Debug CrashLoopBackOff
✓ Test health checks
✓ Port forward để access apps
```

---

## 🎓 Self-Assessment

### Checkpoint: Sẵn Sàng Phần 4?

**1. Cluster Setup**
```
□ Cluster đã running (minikube/kind)
□ kubectl commands work
□ Hiểu Node components
```

**2. Pods**
```
□ Create Pods với kubectl run
□ Get logs với kubectl logs
□ Execute commands với kubectl exec
□ Hiểu Pod lifecycle
□ Debug common issues
```

**3. Namespaces**
```
□ Create namespaces
□ Deploy resources to specific namespace
□ Set default namespace
□ Understand resource quotas
```

**4. Labels**
```
□ Add/update/remove labels
□ Query với selectors
□ Understand Labels vs Annotations
□ Know labeling best practices
```

**If all checked → Ready for Phần 4! 🎉**

---

## 💪 Consolidated Exercises

### Exercise 1: Complete Workflow

```bash
# 1. Setup cluster
minikube start

# 2. Create namespaces
kubectl create ns dev
kubectl create ns prod

# 3. Create Pods với labels
kubectl run webapp --image=nginx -n dev \
  --labels="app=webapp,tier=frontend,env=dev"

kubectl run api --image=nginx -n dev \
  --labels="app=api,tier=backend,env=dev"

kubectl run webapp --image=nginx -n prod \
  --labels="app=webapp,tier=frontend,env=prod"

# 4. Query exercises
kubectl get pods --all-namespaces --show-labels
kubectl get pods -n dev -l tier=frontend
kubectl get pods -A -l app=webapp

# 5. Namespace switching
kubectl config set-context --current --namespace=dev
kubectl get pods  # Shows dev namespace

# 6. Add more labels
kubectl label pods webapp version=v1.0
kubectl label pods api version=v2.1

# 7. Complex queries
kubectl get pods -l 'app in (webapp,api),tier=frontend'

# 8. Cleanup
kubectl delete ns dev prod
```

### Exercise 2: Multi-Container Pod

```yaml
# multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-sidecar
  namespace: dev
  labels:
    app: webapp
    pattern: sidecar
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}
  
  containers:
  # Main application
  - name: webapp
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 3
  
  # Sidecar: Log shipping
  - name: log-shipper
    image: busybox
    command: ['sh', '-c', 'tail -f /var/log/nginx/access.log']
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
      readOnly: true
```

```bash
# Create namespace
kubectl create ns dev

# Apply
kubectl apply -f multi-container-pod.yaml

# Check Pod
kubectl get pods -n dev

# Logs từ main container
kubectl logs -n dev webapp-with-sidecar -c webapp

# Logs từ sidecar
kubectl logs -n dev webapp-with-sidecar -c log-shipper

# Port forward và test
kubectl port-forward -n dev webapp-with-sidecar 8080:80
# Access http://localhost:8080

# Cleanup
kubectl delete -f multi-container-pod.yaml
kubectl delete ns dev
```

### Exercise 3: Resource Quotas và Limits

```yaml
# limited-namespace.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: limited
  labels:
    env: test

---
# Resource Quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: limited
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
    pods: "10"

---
# Limit Range (defaults)
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: limited
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: 1000m
      memory: 1Gi
    min:
      cpu: 50m
      memory: 64Mi
```

```bash
# Apply
kubectl apply -f limited-namespace.yaml

# Check quota
kubectl describe quota -n limited
kubectl describe limitrange -n limited

# Create Pod without resources (gets defaults!)
kubectl run nginx --image=nginx -n limited

# Check applied resources
kubectl get pod nginx -n limited -o yaml | grep -A 10 resources

# Try exceed quota (create 11 Pods)
for i in {1..11}; do
  kubectl run pod-$i --image=nginx -n limited
done
# 11th Pod should fail quota!

# Cleanup
kubectl delete ns limited
```

---

## 🎯 Key Takeaways - Phần 3

### 10 Điều Quan Trọng Nhất

**1. Cluster = Nodes**
```
Master Nodes (Control Plane) + Worker Nodes (Apps)
Setup local: minikube, kind, Docker Desktop
```

**2. Pod = Smallest Unit**
```
Wraps containers
Shared network, volumes, lifecycle
Building block của mọi thứ
```

**3. Multi-Container Patterns**
```
Sidecar: Main + Helper
Ambassador: Main + Proxy
Adapter: Main + Transformer
```

**4. Pod Lifecycle**
```
Pending → Running → Succeeded/Failed
RestartPolicy: Always, OnFailure, Never
```

**5. Health Checks Critical**
```
Liveness: Kill if unhealthy
Readiness: Remove from traffic
Startup: Initial check
```

**6. Namespaces = Organization**
```
Logical isolation
By environment/team/app
Resource quotas per NS
```

**7. Cross-Namespace Access**
```
Service DNS: service.namespace.svc.cluster.local
Example: backend.production
```

**8. Labels = Identification**
```
Key-value pairs
Query với selectors
Used by Services, Deployments
```

**9. Selectors = Query**
```
Equality: key=value
Set-based: key in (v1,v2)
Powerful combinations
```

**10. kubectl Basics**
```
get, describe, logs, exec, port-forward
-n namespace
-l selector
--show-labels
```

---

## 📚 kubectl Cheat Sheet

### Essential Commands

```bash
# Pods
kubectl get pods [-n <namespace>] [-l <selector>]
kubectl describe pod <pod-name> [-n <namespace>]
kubectl logs <pod-name> [-n <namespace>] [-c <container>]
kubectl exec -it <pod-name> [-n <namespace>] -- /bin/bash
kubectl delete pod <pod-name> [-n <namespace>]

# Namespaces
kubectl get namespaces
kubectl create namespace <name>
kubectl delete namespace <name>
kubectl config set-context --current --namespace=<name>

# Labels
kubectl label pods <pod-name> <key>=<value>
kubectl label pods <pod-name> <key>=<value> --overwrite
kubectl label pods <pod-name> <key>-
kubectl get pods --show-labels
kubectl get pods -l <selector>

# Nodes
kubectl get nodes
kubectl describe node <node-name>
kubectl label nodes <node-name> <key>=<value>

# Resource Quotas
kubectl get resourcequota [-n <namespace>]
kubectl describe resourcequota <name> [-n <namespace>]

# General
kubectl get all [-n <namespace>]
kubectl explain <resource>
kubectl api-resources
```

---

## ❓ FAQs

**Q: Minikube vs kind vs Docker Desktop?**
```
Minikube:
✓ Easy, most features
✓ Good for learning
✓ Addons available
Use: Learning, development

kind:
✓ Fast, lightweight
✓ Good for CI/CD
✓ Multi-node clusters
Use: Testing, automation

Docker Desktop:
✓ Easiest setup (GUI)
✓ Integrated với Docker
Use: Mac/Windows users
```

**Q: Khi nào dùng multi-container Pod?**
```
Use when containers:
✓ Must share volumes
✓ Must communicate via localhost
✓ Tightly coupled lifecycle
✓ Helper/sidecar pattern

Don't use for:
✗ Independent services (use separate Pods)
✗ Different scaling needs
✗ Different lifecycles
```

**Q: Resource Quota vs LimitRange?**
```
ResourceQuota:
- Namespace-level totals
- Example: Max 50 Pods total

LimitRange:
- Per-container defaults
- Example: Each container gets 100m CPU default

Use both together for complete control!
```

**Q: Labels có case-sensitive không?**
```
YES!

app=Webapp ≠ app=webapp

Best practice: Lowercase labels
```

---

## 🚀 Tiếp Theo

**Completed:** Core Concepts ✅

**Next:** [Phần 4: Workloads →](../04-workloads/README.md)

Học về:
- Deployments (rolling updates!)
- ReplicaSets (maintain replicas)
- StatefulSets (databases)
- DaemonSets (node services)
- Jobs & CronJobs

Ready to manage workloads at scale! 🎯

---

[⬅️ Phần 2: Architecture](../02-architecture/README.md) | [🏠 Mục Lục Chính](../README.md) | [➡️ Phần 4: Workloads](../04-workloads/README.md)
