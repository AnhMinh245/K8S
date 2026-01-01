# 5.1. Services - Service Discovery & Load Balancing

> Expose và access applications trong Kubernetes

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **Services là gì** và **TẠI SAO cần**
- ✅ Biết **4 types** của Services
- ✅ Implement **service discovery** và load balancing
- ✅ **Expose applications** internally và externally
- ✅ Understand **Endpoints** và DNS
- ✅ **Troubleshoot** network connectivity

---

## 📦 Service Là Gì?

### Định Nghĩa

**Service** = Stable network endpoint để access một set of Pods, provides:
- **Stable IP address** (ClusterIP)
- **Stable DNS name**
- **Load balancing** across Pods
- **Service discovery**

### Giải Thích Bằng Ví Dụ

**Problem Without Services:**

```
🏢 CÔNG TY VỚI NHÂN VIÊN LUÂN CHUYỂN

Backend Pods (like employees):
├── backend-abc123 - IP: 10.244.1.5 (hôm nay)
├── backend-def456 - IP: 10.244.2.8 (hôm nay)
└── backend-ghi789 - IP: 10.244.3.9 (hôm nay)

Frontend cần connect:
frontend → http://10.244.1.5:8080  (hardcode IP!)

Problems:
❌ Pod restarts → New IP (10.244.4.10)
❌ Frontend breaks! (old IP invalid)
❌ Scale up → New Pods với new IPs
❌ Frontend doesn't know about them
❌ Need load balancing → Manual implementation
❌ Service discovery → Impossible!
```

**Solution With Services:**

```
🏢 CÔNG TY VỚI SWITCHBOARD OPERATOR

Service (like switchboard):
├── Name: backend-service
├── Stable IP: 10.96.100.50 (never changes!)
├── DNS: backend-service.default.svc.cluster.local
└── Tracks all backend Pods automatically

Backend Pods:
├── backend-abc123 - IP: 10.244.1.5
├── backend-def456 - IP: 10.244.2.8
└── backend-ghi789 - IP: 10.244.3.9

Frontend connects:
frontend → http://backend-service:8080

Benefits:
✓ Service IP stable (never changes)
✓ DNS name stable
✓ Pod restart → Service updates automatically
✓ Scale up → New Pods added automatically
✓ Load balancing built-in!
✓ Service discovery automatic!
```

---

## 🤔 TẠI SAO Cần Services?

### Pod IP Problems

```bash
# Create Deployment
kubectl create deployment backend --image=nginx --replicas=3

# Get Pod IPs
kubectl get pods -o wide
NAME                     READY   IP           NODE
backend-abc123           1/1     10.244.1.5   node-1
backend-def456           1/1     10.244.2.8   node-2
backend-ghi789           1/1     10.244.3.9   node-3

# Frontend connects to backend:
# ❌ curl http://10.244.1.5  (which Pod? How to load balance?)

# Pod crashes và restarts:
kubectl delete pod backend-abc123

# New Pod với NEW IP:
backend-jkl999           1/1     10.244.4.10  node-1

# ❌ Old IP 10.244.1.5 is now invalid!
# ❌ Frontend connections break!
# ❌ Need to manually update all references!
```

### Service Solution

```bash
# Create Service
kubectl expose deployment backend --port=80

# Service gets stable ClusterIP
kubectl get service backend
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
backend   ClusterIP   10.96.100.50   <none>        80/TCP    10s

# Frontend connects:
# ✓ curl http://backend  (DNS name!)
# ✓ curl http://10.96.100.50  (Stable IP!)

# Service automatically:
# ✓ Tracks all backend Pods
# ✓ Load balances requests
# ✓ Updates when Pods change
# ✓ Provides stable endpoint

# Pod crashes? No problem!
# Service automatically routes to other Pods
# New Pod? Automatically added to Service!
```

---

## 🎯 Service Types

### 1. ClusterIP (Default)

**Internal access only (within cluster)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP  # Default type
  selector:
    app: backend
  ports:
  - port: 80        # Service port
    targetPort: 8080  # Pod port
```

**Characteristics:**
```
ClusterIP: 10.96.100.50 (internal, stable)
DNS: backend-service.default.svc.cluster.local
Access: Only from within cluster

Use cases:
✓ Internal microservice communication
✓ Backend APIs
✓ Databases (internal only)
✓ Inter-service communication
```

**Traffic flow:**
```
Pod A (in cluster)
  ↓
curl http://backend-service:80
  ↓
Service (ClusterIP: 10.96.100.50)
  ↓ Load balances to
┌─────────┬─────────┬─────────┐
│ Pod 1   │ Pod 2   │ Pod 3   │
│ :8080   │ :8080   │ :8080   │
└─────────┴─────────┴─────────┘
```

---

### 2. NodePort

**External access via Node IP + Port**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - port: 80          # Service port
    targetPort: 8080  # Pod port
    nodePort: 30080   # Node port (30000-32767)
```

**Characteristics:**
```
ClusterIP: 10.96.100.60 (internal)
NodePort: 30080 (on ALL Nodes)
Access: <NodeIP>:30080 from external

Use cases:
✓ Development/testing
✓ Direct Node access
✓ No load balancer available
✓ Non-HTTP services
```

**Traffic flow:**
```
External user
  ↓
http://<NodeIP>:30080
  ↓
ANY Node in cluster (port 30080)
  ↓
Service (forwards to Pods on ANY Node)
  ↓
┌─────────┬─────────┬─────────┐
│ Pod 1   │ Pod 2   │ Pod 3   │
│ Node 1  │ Node 2  │ Node 1  │
└─────────┴─────────┴─────────┘

Note: Can hit Node 2:30080 → Route to Pod on Node 1!
```

---

### 3. LoadBalancer

**External access via cloud load balancer**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
```

**Characteristics:**
```
ClusterIP: 10.96.100.70 (internal)
NodePort: 31234 (allocated automatically)
External IP: 35.xxx.xxx.xxx (cloud LB)
Access: http://35.xxx.xxx.xxx from internet

Use cases:
✓ Production web applications
✓ Public-facing services
✓ Cloud environments (GCP, AWS, Azure)
✓ Need managed load balancing
```

**Traffic flow:**
```
Internet user
  ↓
http://35.xxx.xxx.xxx (Cloud Load Balancer)
  ↓ Load balances to
┌─────────┬─────────┬─────────┐
│ Node 1  │ Node 2  │ Node 3  │
│ :31234  │ :31234  │ :31234  │
└─────────┴─────────┴─────────┘
  ↓ Each Node forwards to
┌─────────┬─────────┬─────────┐
│ Pod 1   │ Pod 2   │ Pod 3   │
│ :8080   │ :8080   │ :8080   │
└─────────┴─────────┴─────────┘

Double load balancing:
1. Cloud LB → Nodes
2. NodePort → Pods
```

---

### 4. ExternalName

**DNS alias to external service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: db.example.com  # External DNS
```

**Characteristics:**
```
No ClusterIP
No Endpoints
DNS CNAME record

Use cases:
✓ Migrate from external to internal
✓ Access external databases
✓ Legacy systems integration
✓ Multi-cluster services
```

**Traffic flow:**
```
Pod
  ↓
curl http://external-database
  ↓
DNS lookup: external-database → db.example.com
  ↓
http://db.example.com (external)
  ↓
External service
```

---

## 📝 Service YAML Complete Examples

### ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  labels:
    app: backend
spec:
  type: ClusterIP  # Optional (default)
  selector:
    app: backend   # Match Pods với label này
    tier: api
  ports:
  - name: http
    protocol: TCP
    port: 80       # Port Service listens on
    targetPort: 8080  # Port on Pod
  sessionAffinity: ClientIP  # Optional: Sticky sessions
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

### NodePort Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
    nodePort: 30080  # Optional: auto-assigned nếu không specify
  # Port range: 30000-32767
```

### LoadBalancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  annotations:
    # Cloud-specific annotations
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: "/healthz"
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
  # Optional: Request specific IP
  loadBalancerIP: "35.xxx.xxx.xxx"
  # Optional: Whitelist source IPs
  loadBalancerSourceRanges:
  - "10.0.0.0/8"
  - "192.168.0.0/16"
```

---

## 🎮 Hands-On: Working với Services

### Create ClusterIP Service

```bash
# Method 1: kubectl expose
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --port=80 --target-port=80

# Method 2: YAML file
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
EOF

# Check Service
kubectl get service nginx-service

# Output:
# NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# nginx-service   ClusterIP   10.96.100.123   <none>        80/TCP    10s

# Test from within cluster
kubectl run curl --image=curlimages/curl -it --rm -- curl http://nginx-service

# Output:
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# ...
```

### Create NodePort Service

```bash
# Create Deployment first
kubectl create deployment webapp --image=nginx --replicas=2

# Expose as NodePort
kubectl expose deployment webapp --type=NodePort --port=80

# Get NodePort
kubectl get service webapp

# Output:
# NAME     TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# webapp   NodePort   10.96.100.200   <none>        80:31456/TCP   10s
#                                                         ↑ NodePort

# Get Node IP
kubectl get nodes -o wide
# INTERNAL-IP: 192.168.49.2

# Access from external
curl http://192.168.49.2:31456

# Works! (from outside cluster)
```

### Create LoadBalancer Service

```bash
# Create Deployment
kubectl create deployment frontend --image=nginx --replicas=3

# Expose as LoadBalancer
kubectl expose deployment frontend --type=LoadBalancer --port=80

# Check Service
kubectl get service frontend -w

# Output (wait for EXTERNAL-IP):
# NAME       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# frontend   LoadBalancer   10.96.100.150   <pending>     80:30123/TCP   10s
# frontend   LoadBalancer   10.96.100.150   35.xx.xx.xx   80:30123/TCP   2m
#                                            ↑ Cloud LB IP assigned!

# Access from internet
curl http://35.xx.xx.xx

# (Works from anywhere!)
```

---

## 🔍 Service Discovery

### DNS Names

**K8s automatically creates DNS records:**

```
Service: backend-service
Namespace: default

DNS Names (all work):
├── backend-service (same namespace)
├── backend-service.default (specify namespace)
├── backend-service.default.svc (specify svc)
└── backend-service.default.svc.cluster.local (FQDN)
```

**Example:**

```bash
# Create Service in 'default' namespace
kubectl create deployment api --image=nginx
kubectl expose deployment api --port=80

# Create Pod in 'default' namespace
kubectl run curl --image=curlimages/curl -it --rm -- sh

# Inside Pod, all these work:
curl http://api
curl http://api.default
curl http://api.default.svc
curl http://api.default.svc.cluster.local

# All resolve to same ClusterIP!
```

**Cross-namespace:**

```bash
# Create Service in 'production' namespace
kubectl create namespace production
kubectl create deployment api --image=nginx -n production
kubectl expose deployment api --port=80 -n production

# Access from 'default' namespace
kubectl run curl --image=curlimages/curl -it --rm -- sh

# Must specify namespace:
curl http://api.production
curl http://api.production.svc.cluster.local

# Short name 'api' won't work (different namespace)
```

---

## 🔗 Endpoints

### Endpoints Là Gì?

**Endpoints** = List of Pod IPs that Service routes to

```bash
# Create Service
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --port=80

# View Endpoints
kubectl get endpoints nginx

# Output:
# NAME    ENDPOINTS                                   AGE
# nginx   10.244.1.5:80,10.244.2.8:80,10.244.3.9:80  30s
#         ↑ Pod IPs that Service load balances to

# Describe for details
kubectl describe endpoints nginx

# Output:
# Name:         nginx
# Namespace:    default
# Subsets:
#   Addresses:          10.244.1.5,10.244.2.8,10.244.3.9
#   NotReadyAddresses:  <none>
#   Ports:
#     Name     Port  Protocol
#     ----     ----  --------
#     <unset>  80    TCP
```

**Endpoints update automatically:**

```bash
# Scale Deployment
kubectl scale deployment nginx --replicas=5

# Endpoints updated automatically!
kubectl get endpoints nginx
# Now shows 5 IPs

# Delete a Pod
kubectl delete pod nginx-abc123

# Endpoints updates (removes dead Pod, adds new Pod)
kubectl get endpoints nginx
```

---

## 🐛 Troubleshooting Services

### Issue 1: Service Has No Endpoints

```bash
$ kubectl get service my-app
NAME     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
my-app   ClusterIP   10.96.100.100   <none>        80/TCP    5m

$ kubectl get endpoints my-app
NAME     ENDPOINTS   AGE
my-app   <none>      5m

# No Endpoints! Service can't route traffic

# Debug:
# 1. Check Service selector
$ kubectl describe service my-app | grep Selector
Selector: app=my-app,tier=frontend

# 2. Check Pod labels
$ kubectl get pods --show-labels
NAME        READY   STATUS    LABELS
my-pod      1/1     Running   app=my-app,tier=backend  ← Mismatch!
#                                           ↑ tier=backend not frontend

# Fix: Update selector or Pod labels
kubectl label pod my-pod tier=frontend --overwrite

# Endpoints now populated!
$ kubectl get endpoints my-app
ENDPOINTS         AGE
10.244.1.5:80     1s
```

---

### Issue 2: Can't Access Service

```bash
# Service exists
$ kubectl get service api
NAME   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
api    ClusterIP   10.96.100.200   <none>        80/TCP    5m

# But can't access
$ kubectl run curl --image=curlimages/curl -it --rm -- curl http://api
# Timeout or connection refused

# Debug steps:

# 1. Check Endpoints exist
$ kubectl get endpoints api
# If <none> → See Issue 1

# 2. Check Pods are Ready
$ kubectl get pods -l app=api
NAME        READY   STATUS    RESTARTS   AGE
api-abc12   0/1     Running   0          5m  ← Not Ready!

# If not Ready → Service won't route to it

# 3. Check Pod port
$ kubectl describe service api | grep TargetPort
TargetPort: 8080/TCP

$ kubectl describe pod api-abc12 | grep Port
Port: 80/TCP  ← Mismatch! Pod listens on 80, Service targets 8080

# Fix: Update Service targetPort
kubectl edit service api
# Change targetPort: 8080 → targetPort: 80

# 4. Check network policies
$ kubectl get networkpolicy
# If exists, might block traffic

# 5. Test direct Pod access
$ kubectl get pod api-abc12 -o wide
IP: 10.244.1.5

$ kubectl run curl --image=curlimages/curl -it --rm -- curl http://10.244.1.5
# If direct access works, issue is Service config
```

---

### Issue 3: LoadBalancer Stuck Pending

```bash
$ kubectl get service frontend
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
frontend   LoadBalancer   10.96.100.150   <pending>     80:30123/TCP   10m

# EXTERNAL-IP stuck at <pending>

# Reasons:
1. Cloud provider integration not configured
   - Check cloud-controller-manager running
   - kubectl get pods -n kube-system | grep cloud-controller

2. Running on local cluster (minikube, kind)
   - LoadBalancer type needs cloud provider
   - Use NodePort instead or minikube tunnel

3. Quota/permission issues
   - Check cloud provider quotas
   - Check IAM permissions

# For minikube (workaround):
minikube tunnel
# In another terminal, check Service again
kubectl get service frontend
# Should show EXTERNAL-IP now

# For production: Check cloud provider setup
```

---

## 💡 Best Practices

### Service Names

```yaml
# Good: Descriptive names
apiVersion: v1
kind: Service
metadata:
  name: user-api-service  # Clear purpose
  name: order-backend     # Clear role

# Bad: Generic names
  name: service1          # What does it do?
  name: svc              # Too generic
```

### Port Naming

```yaml
# Good: Named ports
spec:
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: metrics
    port: 9090
    targetPort: 9090

# Easier to understand và reference
```

### Session Affinity

```yaml
# Use when need sticky sessions
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
```

### Health Checks

```yaml
# Ensure Pods are Ready before receiving traffic
spec:
  template:
    spec:
      containers:
      - name: app
        readinessProbe:  # Required!
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3
```

---

## 🎓 Kiểm Tra Hiểu Biết

**1. ClusterIP vs NodePort vs LoadBalancer?**
<details>
<summary>Xem đáp án</summary>

**ClusterIP:**
- Internal only
- Stable ClusterIP
- DNS name
- Use: Internal services

**NodePort:**
- External via Node IP:Port
- Allocates port 30000-32767
- Use: Development, direct access

**LoadBalancer:**
- External via Cloud LB
- Requires cloud provider
- Auto-assigns External IP
- Use: Production, public services

**Hierarchy:** LoadBalancer includes NodePort includes ClusterIP
</details>

**2. Service selector không match Pod labels, chuyện gì xảy ra?**
<details>
<summary>Xem đáp án</summary>

**No Endpoints created!**

```
Service selector: app=backend, tier=api
Pod labels: app=backend, tier=frontend

No match → Endpoints empty → Service can't route traffic

Fix: Update selector hoặc Pod labels để match
```
</details>

**3. DNS name để access Service từ different namespace?**
<details>
<summary>Xem đáp án</summary>

**Must include namespace:**

```
Service: api-service
Namespace: production

From default namespace:
✓ api-service.production
✓ api-service.production.svc.cluster.local

Wrong:
✗ api-service (only works in same namespace)
```
</details>

---

## 🎯 Key Takeaways

1. **Service = Stable Endpoint**
   - Stable IP + DNS
   - Tracks Pods automatically
   - Load balancing built-in

2. **4 Service Types**
   - ClusterIP: Internal only
   - NodePort: External via Node
   - LoadBalancer: External via LB
   - ExternalName: DNS alias

3. **Selector Matching Critical**
   - Service selector must match Pod labels
   - No match = No Endpoints = No traffic

4. **DNS Automatic**
   - service-name.namespace.svc.cluster.local
   - Cross-namespace: include namespace

5. **Endpoints = Pod IPs**
   - Automatically updated
   - Only Ready Pods included
   - Check when troubleshooting

---

## 🚀 Tiếp Theo

Services mastered! Next: Ingress - HTTP/HTTPS routing!

**Next:** [5.2. Ingress →](./02-ingress.md)

---

[⬅️ Phần 4: Workloads](../04-workloads/README.md) | [🏠 Mục Lục](../README.md) | [📂 Phần 5: Networking](./README.md) | [➡️ 5.2. Ingress](./02-ingress.md)

