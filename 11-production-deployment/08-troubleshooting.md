# 11.8. Troubleshooting Production Issues

> Systematic approach để debug và fix production issues

---

## 🎯 Mục Tiêu

- ✅ Systematic debugging workflow
- ✅ Common production issues và fixes
- ✅ Essential kubectl commands
- ✅ Log analysis techniques
- ✅ Performance troubleshooting

---

## 🔍 Debugging Workflow

### Systematic Approach

```
┌─────────────────────────────────────────┐
│   1. IDENTIFY THE PROBLEM               │
│   • What is broken?                     │
│   • When did it start?                  │
│   • What changed recently?              │
└─────────────────────────────────────────┘
            ↓
┌─────────────────────────────────────────┐
│   2. GATHER INFORMATION                 │
│   • Check logs                          │
│   • Check metrics                       │
│   • Check events                        │
│   • Check resource usage                │
└─────────────────────────────────────────┘
            ↓
┌─────────────────────────────────────────┐
│   3. FORM HYPOTHESIS                    │
│   • What could cause this?              │
│   • Most likely root cause?             │
└─────────────────────────────────────────┘
            ↓
┌─────────────────────────────────────────┐
│   4. TEST HYPOTHESIS                    │
│   • Verify theory                       │
│   • Check related components            │
└─────────────────────────────────────────┘
            ↓
┌─────────────────────────────────────────┐
│   5. IMPLEMENT FIX                      │
│   • Apply fix                           │
│   • Monitor results                     │
│   • Verify fix works                    │
└─────────────────────────────────────────┘
            ↓
┌─────────────────────────────────────────┐
│   6. DOCUMENT & PREVENT                 │
│   • Write post-mortem                   │
│   • Update runbooks                     │
│   • Implement prevention                │
└─────────────────────────────────────────┘
```

---

## 🚨 Common Production Issues

### Issue 1: Pod Stuck in Pending State

**Symptoms:**
```bash
$ kubectl get pods
NAME                 READY   STATUS    RESTARTS   AGE
myapp-abc123-xyz     0/1     Pending   0          5m
```

**Debugging Steps:**

```bash
# 1. Describe pod để xem events
kubectl describe pod myapp-abc123-xyz

# Common causes trong events:
# "Insufficient cpu"
# "Insufficient memory"
# "No nodes available"
# "Pod has unbound PersistentVolumeClaims"
```

**Root Causes & Fixes:**

**A. Insufficient Resources:**
```bash
# Check node capacity
kubectl top nodes
kubectl describe nodes

# Fix: Scale cluster hoặc giảm resource requests
kubectl scale deployment myapp --replicas=2  # Temporary
# Hoặc update resource requests trong Deployment
```

**B. Node Selector/Affinity Issues:**
```bash
# Check pod spec
kubectl get pod myapp-abc123-xyz -o yaml | grep -A 10 nodeSelector

# Fix: Remove hoặc adjust nodeSelector/affinity
kubectl edit deployment myapp
# Xóa hoặc sửa nodeSelector section
```

**C. Unbound PVC:**
```bash
# Check PVCs
kubectl get pvc
NAME        STATUS    VOLUME   CAPACITY   STORAGECLASS
data-pvc    Pending   -        -          standard

# Check storage class
kubectl get storageclass

# Fix: Create PV hoặc use dynamic provisioning
kubectl apply -f pv.yaml
```

---

### Issue 2: CrashLoopBackOff

**Symptoms:**
```bash
$ kubectl get pods
NAME                 READY   STATUS             RESTARTS   AGE
myapp-abc123-xyz     0/1     CrashLoopBackOff   5          10m
```

**Debugging:**

```bash
# 1. Check logs
kubectl logs myapp-abc123-xyz
kubectl logs myapp-abc123-xyz --previous  # Logs from crashed container

# 2. Describe pod
kubectl describe pod myapp-abc123-xyz

# 3. Check events
kubectl get events --sort-by='.lastTimestamp' | grep myapp
```

**Common Root Causes:**

**A. Application Error:**
```bash
# Logs show:
# "Error: Cannot connect to database"
# "Fatal: Missing environment variable API_KEY"

# Fix: Check ConfigMap/Secret
kubectl get configmap myapp-config -o yaml
kubectl get secret myapp-secrets -o yaml

# Verify environment variables trong pod
kubectl exec myapp-abc123-xyz -- env
```

**B. Liveness Probe Failing:**
```yaml
# Check probe configuration
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10  # Too short?
  periodSeconds: 5
  failureThreshold: 3

# Fix: Increase initialDelaySeconds
# Application có thể cần thời gian khởi động
livenessProbe:
  initialDelaySeconds: 60  # Increase
```

**C. Out of Memory (OOMKilled):**
```bash
# Describe pod sẽ show:
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137

# Check memory usage
kubectl top pod myapp-abc123-xyz

# Fix: Increase memory limits
resources:
  limits:
    memory: 1Gi  # Increase từ 512Mi
  requests:
    memory: 512Mi
```

---

### Issue 3: Service Không Accessible

**Symptoms:**
```bash
# Curl to service fails
$ kubectl run test --rm -it --image=busybox -- wget -O- http://myapp-service
wget: can't connect to remote host: Connection refused
```

**Debugging:**

```bash
# 1. Check service
kubectl get svc myapp-service
kubectl describe svc myapp-service

# 2. Check endpoints
kubectl get endpoints myapp-service

# No endpoints? → Service selector không match Pods
```

**Root Causes:**

**A. Service Selector Mismatch:**
```bash
# Check service selector
kubectl get svc myapp-service -o yaml | grep -A 5 selector

# Check pod labels
kubectl get pods --show-labels

# Fix: Update service selector to match pod labels
kubectl edit svc myapp-service
```

**B. Pods Not Ready:**
```bash
# Check pod status
kubectl get pods -l app=myapp

# Pods Running but NOT Ready? Check readiness probe
kubectl describe pod myapp-abc123-xyz | grep -A 10 Readiness

# Fix readiness probe issues
```

**C. Network Policy Blocking:**
```bash
# Check network policies
kubectl get networkpolicies

# Describe policy
kubectl describe networkpolicy myapp-netpol

# Fix: Adjust network policy to allow traffic
```

---

### Issue 4: High CPU/Memory Usage

**Symptoms:**
```bash
$ kubectl top pods
NAME                 CPU(cores)   MEMORY(bytes)
myapp-abc123-xyz     980m         1900Mi

# Pod using 98% của limit!
```

**Debugging:**

```bash
# 1. Check resource limits
kubectl describe pod myapp-abc123-xyz | grep -A 10 Limits

# 2. Profile application
kubectl exec myapp-abc123-xyz -- top
kubectl exec myapp-abc123-xyz -- ps aux

# 3. Check for memory leaks
# Monitor memory over time
watch kubectl top pod myapp-abc123-xyz
```

**Fixes:**

**A. Increase Limits (short-term):**
```yaml
resources:
  limits:
    cpu: 2000m      # Increase
    memory: 3Gi     # Increase
  requests:
    cpu: 1000m
    memory: 1Gi
```

**B. Optimize Application (long-term):**
```
• Profile code (CPU/memory profiling)
• Fix memory leaks
• Optimize database queries
• Add caching
• Use connection pooling
```

**C. Scale Horizontally:**
```bash
# Add more replicas thay vì increase limits
kubectl scale deployment myapp --replicas=5
```

---

### Issue 5: ImagePullBackOff

**Symptoms:**
```bash
$ kubectl get pods
NAME                 READY   STATUS             RESTARTS   AGE
myapp-abc123-xyz     0/1     ImagePullBackOff   0          2m
```

**Debugging:**

```bash
# Describe pod
kubectl describe pod myapp-abc123-xyz

# Events sẽ show:
# Failed to pull image "gcr.io/project/myapp:v1.0"
# Error: unauthorized / not found / timeout
```

**Root Causes:**

**A. Image Doesn't Exist:**
```bash
# Check image tag
kubectl get deployment myapp -o yaml | grep image:

# Verify image exists
docker pull gcr.io/project/myapp:v1.0

# Fix: Correct image tag
kubectl set image deployment/myapp myapp=gcr.io/project/myapp:v1.1
```

**B. Authentication Issues:**
```bash
# Check image pull secrets
kubectl get secrets
kubectl describe secret regcred

# Fix: Create correct secret
kubectl create secret docker-registry regcred \
  --docker-server=gcr.io \
  --docker-username=_json_key \
  --docker-password="$(cat key.json)"

# Add to pod spec
imagePullSecrets:
- name: regcred
```

**C. Rate Limit (Docker Hub):**
```bash
# Docker Hub rate limits: 100 pulls/6h

# Fix: Use authenticated pulls
kubectl create secret docker-registry dockerhub \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_PASSWORD
```

---

### Issue 6: Persistent Volume Issues

**Symptoms:**
```bash
# Pod pending with PVC issue
$ kubectl get pods
NAME                 READY   STATUS    RESTARTS   AGE
myapp-abc123-xyz     0/1     Pending   0          10m

$ kubectl get pvc
NAME        STATUS    VOLUME   CAPACITY
data-pvc    Pending   -        -
```

**Debugging:**

```bash
# Describe PVC
kubectl describe pvc data-pvc

# Events:
# "no persistent volumes available"
# "storageclass not found"
# "volume size exceeded quota"
```

**Fixes:**

**A. No Available PV:**
```bash
# Check PVs
kubectl get pv

# Create PV manually or use StorageClass for dynamic provisioning
```

**B. StorageClass Issues:**
```bash
# Check StorageClass
kubectl get storageclass

# Fix: Create or use correct StorageClass
kubectl apply -f storageclass.yaml

# Update PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  storageClassName: fast-ssd  # Specify correct class
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

---

## 🛠️ Essential Kubectl Commands

### Quick Reference

**Pod Debugging:**
```bash
# Get pods with details
kubectl get pods -o wide
kubectl get pods --all-namespaces

# Describe pod (events, conditions)
kubectl describe pod <pod-name>

# Logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl logs <pod-name> -f  # Follow
kubectl logs <pod-name> -c <container-name>  # Multi-container

# Execute commands in pod
kubectl exec <pod-name> -- ls /app
kubectl exec -it <pod-name> -- /bin/bash

# Debug with ephemeral container (K8s 1.23+)
kubectl debug <pod-name> -it --image=busybox --target=<container-name>

# Copy files
kubectl cp <pod-name>:/app/log.txt ./log.txt
kubectl cp ./config.yaml <pod-name>:/app/config.yaml

# Port forward
kubectl port-forward <pod-name> 8080:80
```

**Node Debugging:**
```bash
# Node status
kubectl get nodes
kubectl top nodes
kubectl describe node <node-name>

# Cordon/drain node
kubectl cordon <node-name>  # Prevent scheduling
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node-name>  # Allow scheduling

# SSH to node (GKE)
gcloud compute ssh <node-name>
```

**Events và Logs:**
```bash
# Cluster events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector type=Warning

# Component logs
kubectl logs -n kube-system <kube-apiserver-pod>
kubectl logs -n kube-system <kube-controller-manager-pod>
```

**Resource Usage:**
```bash
# Metrics
kubectl top nodes
kubectl top pods
kubectl top pods --containers

# Resource requests/limits
kubectl describe node <node-name> | grep -A 5 "Allocated resources"
```

---

## 📊 Performance Troubleshooting

### Slow Application Response

**Investigation:**

```bash
# 1. Check pod CPU/memory
kubectl top pod <pod-name>

# 2. Check application logs
kubectl logs <pod-name> | grep -i "slow\|timeout\|error"

# 3. Network latency
kubectl exec <pod-name> -- ping <service-name>
kubectl exec <pod-name> -- curl -w "@curl-format.txt" http://service-name

# curl-format.txt:
#     time_namelookup:  %{time_namelookup}\n
#        time_connect:  %{time_connect}\n
#     time_appconnect:  %{time_appconnect}\n
#    time_pretransfer:  %{time_pretransfer}\n
#       time_redirect:  %{time_redirect}\n
#  time_starttransfer:  %{time_starttransfer}\n
#                     ----------\n
#          time_total:  %{time_total}\n
```

**Common Causes:**

1. **Database Performance:**
   - Slow queries
   - Missing indexes
   - Connection pool exhausted

2. **External API Calls:**
   - Third-party API slow/down
   - Timeout settings too high
   - No circuit breaker

3. **Resource Constraints:**
   - CPU throttling
   - Memory pressure
   - Disk I/O bottleneck

---

### Network Troubleshooting

**Tools:**

```bash
# Deploy debug pod
kubectl run debug --rm -it --image=nicolaka/netshoot -- /bin/bash

# Inside debug pod:
# DNS resolution
nslookup myservice
dig myservice.default.svc.cluster.local

# Connectivity
ping myservice
curl http://myservice:80

# Trace route
traceroute myservice

# Port scanning
nc -zv myservice 80

# tcpdump
tcpdump -i any port 80
```

---

## 📝 Incident Response Checklist

### During Incident

```
□ Acknowledge alert
□ Assess severity
□ Start incident channel
□ Identify impacted services
□ Check recent changes
□ Gather logs và metrics
□ Form hypothesis
□ Test hypothesis
□ Implement fix
□ Monitor results
□ Verify fix
□ Communicate status
```

### Post-Incident

```
□ Write post-mortem
□ Timeline of events
□ Root cause analysis
□ What went well
□ What didn't go well
□ Action items
□ Assign owners
□ Set deadlines
□ Follow up
```

---

## 🎯 Pro Tips

**1. Use kubectl aliases:**
```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias kx='kubectl exec -it'
```

**2. Set context aliases:**
```bash
alias kprod='kubectl config use-context production'
alias kstag='kubectl config use-context staging'
```

**3. Watch resources:**
```bash
watch kubectl get pods
kubectl get pods -w  # Watch mode
```

**4. JSON path queries:**
```bash
# Get pod IPs
kubectl get pods -o jsonpath='{.items[*].status.podIP}'

# Get container images
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
```

---

## 🎓 Key Takeaways

**1. Systematic Approach:**
- Don't panic
- Follow debugging workflow
- Document findings

**2. Common Issues:**
- Pending: Resources, scheduling
- CrashLoopBackOff: App errors, probes
- ImagePullBackOff: Auth, image not found
- High usage: Optimize or scale

**3. Essential Tools:**
- kubectl logs, describe, exec
- top, events
- Port forwarding
- Debug pods

**4. Prevention:**
- Monitor proactively
- Set up alerts
- Regular testing
- Keep runbooks updated

---

[⬅️ 11.7. Cost Optimization](./07-cost-optimization.md) | [➡️ 11.9. Case Studies](./09-case-studies.md) | [🏠 Mục Lục](../README.md)

