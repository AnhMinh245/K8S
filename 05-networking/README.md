# 📘 Phần 5: Networking - Kết Nối Ứng Dụng

> Service discovery, load balancing, và network policies

---

## 🎯 Mục Tiêu Phần Này

Sau khi hoàn thành Phần 5, bạn sẽ:

✅ **Expose applications** với Services  
✅ **HTTP routing** với Ingress  
✅ **Understand Pod networking** fundamentals  
✅ **Implement network security** với Network Policies  
✅ **Troubleshoot** network connectivity  
✅ **Production-ready networking** setup  

---

## 📚 Nội Dung

### [5.1. Services - Service Discovery](./01-services.md) ⭐⭐⭐⭐⭐

**Thời gian:** 75-90 phút (QUAN TRỌNG!)

**Nội dung:**
- Service là gì và TẠI SAO cần
- 4 Service types: ClusterIP, NodePort, LoadBalancer, ExternalName
- Service discovery và DNS
- Load balancing automatic
- Endpoints tracking
- Troubleshooting Services

**Key Concepts:**
```
✓ Service = Stable endpoint cho Pods
✓ ClusterIP: Internal only (default)
✓ NodePort: External via Node IP:Port
✓ LoadBalancer: External via Cloud LB
✓ DNS automatic: service.namespace.svc.cluster.local
✓ Endpoints = Pod IPs
```

**Use Cases:**
- ClusterIP: Internal microservices
- NodePort: Development, simple external access
- LoadBalancer: Production external services
- ExternalName: External service alias

**Commands:**
```bash
# Create Service
kubectl expose deployment app --port=80 --type=ClusterIP

# Get Services
kubectl get service
kubectl get endpoints

# Test connectivity
kubectl run curl --image=curlimages/curl -it --rm -- curl http://service-name
```

---

### [5.2. Ingress - HTTP Routing](./02-ingress.md) ⭐⭐⭐⭐⭐

**Thời gian:** 75-90 phút (QUAN TRỌNG!)

**Nội dung:**
- Ingress là gì và TẠI SAO cần
- Ingress Controller setup (NGINX, Traefik)
- Path-based routing
- Host-based routing (virtual hosting)
- TLS/SSL termination
- Annotations và advanced features

**Key Concepts:**
```
✓ Ingress = Layer 7 (HTTP/HTTPS) load balancer
✓ One LoadBalancer cho many Services
✓ Path routing: /api → api-service
✓ Host routing: api.example.com → api-service
✓ TLS termination centralized
✓ Cost-effective vs multiple LoadBalancers
```

**Traffic Flow:**
```
Internet → Ingress Controller (LB)
  → Ingress rules (path/host matching)
    → Backend Services (ClusterIP)
      → Pods
```

**Commands:**
```bash
# Install Ingress Controller
helm install nginx-ingress ingress-nginx/ingress-nginx

# Create Ingress
kubectl apply -f ingress.yaml

# Get Ingress
kubectl get ingress
kubectl describe ingress <name>
```

---

### [5.3. Pod Networking - Fundamentals](./03-pod-networking.md) ⭐⭐⭐⭐

**Thời gian:** 60-75 phút

**Nội dung:**
- K8s network model
- CNI (Container Network Interface)
- Pod-to-Pod communication (same Node, cross-Node)
- Network Policies (firewall rules)
- Security segmentation
- CNI plugins comparison (Calico, Flannel, Cilium)

**Key Concepts:**
```
✓ K8s network model: No NAT, flat network
✓ CNI plugin handles networking
✓ Pod IPs routable cluster-wide
✓ Network Policies = Pod-level firewall
✓ Default: No restrictions (open network)
✓ Security: Implement default deny + whitelist
```

**Network Requirements:**
1. Pods can communicate với all Pods (no NAT)
2. Nodes can communicate với all Pods
3. Pod's IP = IP others see it as

**Commands:**
```bash
# Check CNI plugin
kubectl get pods -n kube-system | grep -E 'calico|flannel'

# Get Pod IPs
kubectl get pods -o wide

# Network Policies
kubectl get networkpolicy
kubectl describe networkpolicy <name>

# Test connectivity
kubectl exec pod-a -- curl http://pod-b-ip
```

---

## 🗺️ Networking Decision Tree

### Expose Application: Which Method?

```
START
  ↓
Internal only (within cluster)?
  ├─ YES → ClusterIP Service ✓
  │
  └─ NO → HTTP/HTTPS traffic?
            ├─ YES → Multiple services? Path routing?
            │         ├─ YES → Ingress ✓ (cost-effective)
            │         └─ NO → LoadBalancer Service (simple)
            │
            └─ NO → TCP/UDP non-HTTP?
                      └─ LoadBalancer Service ✓
                          (or NodePort for dev)
```

### Quick Reference Table

| Scenario | Solution | Type |
|----------|----------|------|
| **Internal microservices** | ClusterIP Service | Default |
| **External HTTP API** | Ingress + ClusterIP | Production |
| **External non-HTTP** | LoadBalancer Service | Production |
| **Multiple HTTP services** | Ingress (one LB, many services) | Cost-effective |
| **Development/testing** | NodePort Service | Dev/test |
| **Database (internal)** | ClusterIP Service + Headless | StatefulSet |
| **Legacy external service** | ExternalName Service | Integration |

---

## 🎓 Self-Assessment

### Checkpoint: Sẵn Sàng Phần 6?

**1. Services**
```
□ Understand 4 Service types
□ Create ClusterIP, NodePort, LoadBalancer
□ DNS names (service.namespace.svc.cluster.local)
□ Check Endpoints
□ Troubleshoot Service connectivity
```

**2. Ingress**
```
□ Setup Ingress Controller
□ Path-based routing
□ Host-based routing
□ TLS termination
□ Understand annotations
```

**3. Networking**
```
□ Understand K8s network model
□ Know CNI plugin role
□ Pod-to-Pod communication
□ Create Network Policies
□ Troubleshoot network issues
```

**If all checked → Ready for Phần 6! 🎉**

---

## 💪 Consolidated Exercises

### Exercise 1: Complete 3-Tier Application

```yaml
# 1. Database (StatefulSet + ClusterIP + Headless Service)
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None  # Headless
  selector:
    app: postgres
  ports:
  - port: 5432

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        tier: database
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          value: password
        ports:
        - containerPort: 5432

---
# 2. Backend API (Deployment + ClusterIP Service)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      containers:
      - name: api
        image: api-server:v1
        ports:
        - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080

---
# 3. Frontend (Deployment + ClusterIP Service)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80

---
# 4. Ingress (External access)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80

---
# 5. Network Policies (Security)
# Backend can only be accessed by Frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# Database can only be accessed by Backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
```

```bash
# Deploy all
kubectl apply -f complete-app.yaml

# Verify Services
kubectl get services

# Verify Ingress
kubectl get ingress

# Test external access
curl http://myapp.example.com/api
curl http://myapp.example.com/

# Test Network Policies
# Frontend → Backend (should work)
kubectl exec -it frontend-xxx -- curl backend-service

# Frontend → Database (should fail)
kubectl exec -it frontend-xxx -- nc -zv postgres 5432
```

---

### Exercise 2: Multi-Domain Ingress

```yaml
# Host-based routing cho multiple apps
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-domain-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app1.example.com
    - app2.example.com
    secretName: multi-domain-tls
  rules:
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

---

## 🎯 Key Takeaways - Phần 5

### 10 Điều Quan Trọng Nhất

**1. Service = Stable Endpoint**
```
Pod IPs change → Service IP stable
DNS name stable
Load balancing automatic
```

**2. 4 Service Types**
```
ClusterIP: Internal (default)
NodePort: External via Node
LoadBalancer: External via Cloud LB
ExternalName: DNS alias
```

**3. Ingress = HTTP Router**
```
One LoadBalancer for many Services
Path routing: /api, /web
Host routing: api.example.com
TLS termination centralized
Cost-effective
```

**4. Ingress Requires Controller**
```
NGINX, Traefik, HAProxy
Deploy controller first
One controller handles multiple Ingresses
```

**5. DNS Automatic**
```
service-name (same namespace)
service-name.namespace (cross-namespace)
service-name.namespace.svc.cluster.local (FQDN)
```

**6. K8s Network Model**
```
Pod-to-Pod without NAT
Flat network (all Pods can communicate)
CNI plugin implements networking
```

**7. CNI Plugins**
```
Calico: Production, Network Policies
Flannel: Simple, development
Cilium: Modern, eBPF
Choose based on requirements
```

**8. Network Policies = Firewall**
```
Control Pod traffic (ingress/egress)
Default: No restrictions (open)
Security: Default deny + whitelist
```

**9. Troubleshooting Order**
```
1. Check Service/Endpoints
2. Check Pod IPs và status
3. Check Network Policies
4. Check CNI plugin
5. Check kube-proxy
```

**10. Production Setup**
```
✓ Use Ingress for HTTP services
✓ Implement Network Policies
✓ Monitor networking metrics
✓ Plan IP address space (CIDR)
✓ TLS termination at Ingress
```

---

## 📚 Commands Cheat Sheet

### Services

```bash
# Create
kubectl expose deployment app --port=80 --type=ClusterIP
kubectl create service clusterip app --tcp=80:8080

# Get
kubectl get services
kubectl get endpoints
kubectl describe service <name>

# Test
kubectl run curl --image=curlimages/curl -it --rm -- curl http://service-name

# Delete
kubectl delete service <name>
```

### Ingress

```bash
# Install Controller
helm install nginx-ingress ingress-nginx/ingress-nginx

# Get
kubectl get ingress
kubectl describe ingress <name>

# Get Ingress IP
kubectl get ingress <name> -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Test
curl http://<ingress-ip>/<path>
```

### Networking

```bash
# Get Pod IPs
kubectl get pods -o wide

# Network Policies
kubectl get networkpolicy
kubectl describe networkpolicy <name>

# Test connectivity
kubectl exec pod-a -- curl http://pod-b-ip
kubectl exec pod-a -- nc -zv service-name port

# Check CNI
kubectl get pods -n kube-system | grep -E 'calico|flannel|cilium'
```

---

## ❓ FAQs

**Q: Service ClusterIP vs Headless Service?**
```
ClusterIP (normal):
- Has ClusterIP assigned
- Load balances to Pods
- DNS resolves to ClusterIP

Headless (clusterIP: None):
- No ClusterIP assigned
- DNS resolves to individual Pod IPs
- Use for: StatefulSets, direct Pod access
```

**Q: Ingress vs LoadBalancer Service - khi nào dùng gì?**
```
Use Ingress when:
✓ HTTP/HTTPS services
✓ Multiple services (cost-effective)
✓ Need path/host routing
✓ TLS termination centralized

Use LoadBalancer when:
✓ Non-HTTP (TCP/UDP)
✓ Single service external access
✓ Simple setup
```

**Q: Network Policy không work - tại sao?**
```
Reasons:
1. CNI plugin doesn't support Network Policies
   - Flannel: No support
   - Calico, Cilium, Weave: Support

2. Policy selectors don't match Pods
   - Check labels match

3. Forgot to allow DNS
   - Must allow port 53 UDP to kube-system

Check: kubectl get networkpolicy -A
```

**Q: Pod-to-Pod same Node vs cross-Node - khác nhau không?**
```
Same Node:
- Direct L2 communication (fast)
- Via Linux bridge

Cross-Node:
- Via Node network
- May use overlay (VXLAN)
- Slightly higher latency

Both use same Pod IPs (no NAT)
Transparent to applications
```

---

## 🚀 Tiếp Theo

**Completed:** Networking - Kết nối ứng dụng ✅

**Next:** [Phần 6: Configuration →](../06-configuration/README.md)

Learn about:
- ConfigMaps (external configuration)
- Secrets (sensitive data)
- Environment variables
- Configuration best practices

Manage application configuration! ⚙️

---

[⬅️ Phần 4: Workloads](../04-workloads/README.md) | [🏠 Mục Lục Chính](../README.md) | [➡️ Phần 6: Configuration](../06-configuration/README.md)
