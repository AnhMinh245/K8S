# 5.3. Pod Networking - Network Fundamentals

> Hiểu cách Pods communicate trong Kubernetes

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **Pod network model** trong K8s
- ✅ Biết **CNI (Container Network Interface)** là gì
- ✅ Understand **Pod-to-Pod communication**
- ✅ Hiểu **Network Policies** cho security
- ✅ **Troubleshoot** network connectivity
- ✅ **Choose CNI plugin** phù hợp

---

## 📦 Kubernetes Network Model

### K8s Network Requirements

**Kubernetes imposes 3 fundamental requirements:**

```
1. PODS CAN COMMUNICATE với tất cả Pods khác
   - Without NAT (no address translation)
   - Direct Pod-to-Pod communication
   - Across any Nodes

2. NODES CAN COMMUNICATE với tất cả Pods
   - kubelet, system daemons can reach Pods
   - No NAT required

3. POD'S IP = IP OTHERS SEE IT AS
   - Pod sees itself với same IP others see
   - No NAT, no port mapping
   - Transparent networking
```

### Giải Thích Bằng Ví Dụ

**Traditional VM/Docker networking:**
```
Host 1 (IP: 192.168.1.10)
├── Container A (IP: 172.17.0.2)  ← Private IP
├── Container B (IP: 172.17.0.3)  ← Private IP
└── NAT to communicate outside

Container A → Container B on Host 2:
❌ Can't reach directly (private IPs)
✓ Must go through Host IP + port mapping
✓ Complex NAT và routing
```

**Kubernetes networking:**
```
Node 1 (IP: 192.168.1.10)
├── Pod A (IP: 10.244.1.5)  ← Cluster-wide routable
├── Pod B (IP: 10.244.1.6)  ← Cluster-wide routable
└── No NAT required

Node 2 (IP: 192.168.1.11)
├── Pod C (IP: 10.244.2.5)  ← Cluster-wide routable
└── Pod D (IP: 10.244.2.6)  ← Cluster-wide routable

Pod A → Pod C:
✓ Direct communication (10.244.1.5 → 10.244.2.5)
✓ No NAT, no port mapping
✓ Simple, flat network
```

---

## 🌐 CNI (Container Network Interface)

### Định Nghĩa

**CNI** = Specification và libraries để configure network interfaces trong Linux containers.

### Vai Trò

```
CNI Plugin responsibilities:
├── Assign IP address to Pod
├── Setup network interface trong Pod
├── Configure routes
├── Setup network policies
└── Enable Pod-to-Pod communication

Kubernetes → CNI Plugin → Network configured
```

### Popular CNI Plugins

**1. Calico**
```
Features:
✓ Network policies (security)
✓ BGP routing
✓ IPAM (IP Address Management)
✓ Encryption (WireGuard)
✓ Performance (no overlay needed)

Use cases:
- Production clusters
- Need network policies
- Performance critical
- Large scale (1000+ nodes)
```

**2. Flannel**
```
Features:
✓ Simple setup
✓ VXLAN overlay
✓ Minimal config
✗ No network policies

Use cases:
- Development clusters
- Simple networking needs
- Learning K8s
```

**3. Cilium**
```
Features:
✓ eBPF-based (kernel-level)
✓ Network policies (L3-L7)
✓ Service mesh features
✓ Observability
✓ High performance

Use cases:
- Modern infrastructure
- Need advanced features
- Security-focused
- Observability needs
```

**4. Weave Net**
```
Features:
✓ Automatic setup
✓ Encryption
✓ Multicast support
✓ Simple operation

Use cases:
- Quick setup
- Encryption needed
- Cross-cloud networking
```

---

## 🔄 Pod-to-Pod Communication

### Same Node Communication

```
┌────────────────────────────────────┐
│         NODE 1                     │
│                                    │
│  ┌─────────┐      ┌─────────┐    │
│  │ Pod A   │      │ Pod B   │    │
│  │ 10.244.1.5     │ 10.244.1.6    │
│  └────┬────┘      └────┬────┘    │
│       │                 │         │
│       └────────┬────────┘         │
│                │                   │
│          [Linux Bridge]           │
│                │                   │
│          [Node NIC]               │
└────────────────────────────────────┘

Flow: Pod A → Pod B (same Node)
1. Packet from Pod A (10.244.1.5)
2. Goes to Linux bridge (virtual switch)
3. Bridge forwards to Pod B (10.244.1.6)
4. Direct L2 communication (fast!)
```

### Cross-Node Communication

```
┌──────────────────────┐       ┌──────────────────────┐
│     NODE 1           │       │     NODE 2           │
│                      │       │                      │
│  ┌─────────┐        │       │  ┌─────────┐        │
│  │ Pod A   │        │       │  │ Pod C   │        │
│  │ 10.244.1.5       │       │  │ 10.244.2.5       │
│  └────┬────┘        │       │  └────┬────┘        │
│       │              │       │       │              │
│  [Linux Bridge]     │       │  [Linux Bridge]     │
│       │              │       │       │              │
│  [Node NIC]         │       │  [Node NIC]         │
│  192.168.1.10       │       │  192.168.1.11       │
└──────┼──────────────┘       └──────┼──────────────┘
       │                              │
       └──────────[Network]──────────┘

Flow: Pod A → Pod C (different Node)
1. Packet from Pod A (10.244.1.5 → 10.244.2.5)
2. Goes to Linux bridge
3. Bridge sees destination on different Node
4. Routes via Node network (192.168.1.10 → 192.168.1.11)
5. Node 2 receives, forwards to Pod C
6. Pod C receives packet

CNI Plugin handles:
- Routing between Node networks
- Encapsulation (if overlay network)
- Route tables configuration
```

---

## 🔒 Network Policies

### Định Nghĩa

**Network Policy** = Firewall rules cho Pods, control ingress và egress traffic.

### Default Behavior (No NetworkPolicies)

```
Without Network Policies:
├── ALL Pods can communicate với ALL Pods
├── ALL Pods can reach external internet
├── ALL external traffic can reach Pods (if exposed)
└── No restrictions (open network)

→ Not secure for production!
```

### Tại Sao Cần Network Policies?

```
Scenario: Microservices Application

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Frontend   │────→│   Backend   │────→│  Database   │
│   Pods      │     │    Pods     │     │    Pods     │
└─────────────┘     └─────────────┘     └─────────────┘

Without Network Policies:
❌ Frontend can directly access Database (security risk!)
❌ Database accepts traffic from any Pod
❌ No segmentation
❌ Lateral movement if compromised

With Network Policies:
✓ Frontend can ONLY access Backend
✓ Backend can ONLY access Database
✓ Database accepts ONLY from Backend
✓ Network segmentation
✓ Reduced blast radius
```

---

### Network Policy Example

**Deny All Ingress (Default Deny)**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}  # Apply to ALL Pods in namespace
  policyTypes:
  - Ingress
  # No ingress rules = deny all
```

**Allow Specific Ingress**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend  # Apply to backend Pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend  # Allow from frontend Pods only
    ports:
    - protocol: TCP
      port: 8080  # Allow only port 8080
```

**Traffic flow:**
```
Frontend Pod (app=frontend)
  ↓ ALLOWED (matches rule)
Backend Pod (app=backend, port 8080)

Database Pod (app=database)
  ↓ DENIED (no rule)
Backend Pod
```

---

**Allow from Specific Namespace**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web  # Apply to web Pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring  # Allow from monitoring namespace
    ports:
    - protocol: TCP
      port: 9090  # Metrics port
```

---

**Egress Policy (Control Outbound)**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database  # Can only reach database Pods
    ports:
    - protocol: TCP
      port: 5432
  - to:  # Allow DNS (required!)
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

---

## 🎮 Hands-On: Network Policies

### Scenario: 3-Tier Application

```yaml
# 1. Create namespaces
---
apiVersion: v1
kind: Namespace
metadata:
  name: app
  labels:
    name: app

---
# 2. Frontend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      tier: frontend
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
# 3. Backend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      tier: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args: ["-text=Backend API"]
        ports:
        - containerPort: 5678

---
# 4. Database Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
      tier: database
  template:
    metadata:
      labels:
        app: database
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
# 5. Network Policy: Frontend can only access Backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: app
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
          tier: frontend  # Only frontend Pods
    ports:
    - protocol: TCP
      port: 5678

---
# 6. Network Policy: Database can only be accessed by Backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: app
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
          tier: backend  # Only backend Pods
    ports:
    - protocol: TCP
      port: 5432
```

**Test Network Policies:**

```bash
# Apply all
kubectl apply -f network-policies.yaml

# Get Pods
kubectl get pods -n app -o wide

# Test: Frontend → Backend (SHOULD WORK)
kubectl exec -n app -it frontend-xxx -- curl backend-service:5678
# Success!

# Test: Frontend → Database (SHOULD FAIL)
kubectl exec -n app -it frontend-xxx -- nc -zv database-service 5432
# Connection refused (blocked by Network Policy)

# Test: Backend → Database (SHOULD WORK)
kubectl exec -n app -it backend-xxx -- nc -zv database-service 5432
# Connection succeeded
```

---

## 🐛 Troubleshooting Networking

### Issue 1: Pod Can't Reach Another Pod

```bash
# Symptoms
$ kubectl exec pod-a -- curl http://pod-b-ip
# Connection timeout

# Debug steps:

# 1. Check Pod IPs
$ kubectl get pods -o wide
# Verify IPs assigned

# 2. Check CNI plugin running
$ kubectl get pods -n kube-system | grep -E 'calico|flannel|weave'
# Should be Running

# 3. Ping test (if allowed)
$ kubectl exec pod-a -- ping pod-b-ip
# Tests L3 connectivity

# 4. Check Network Policies
$ kubectl get networkpolicy -A
# Any policies blocking traffic?

$ kubectl describe networkpolicy <policy-name>
# Check if pods match selectors

# 5. Check routes on Node
$ kubectl get nodes -o wide
# SSH to Node, check routes
ip route show

# 6. Check iptables (if using kube-proxy iptables)
sudo iptables-save | grep <service-name>
```

---

### Issue 2: Service Not Reachable

```bash
# Service exists but can't reach
$ kubectl get service api
# Exists

$ kubectl exec test-pod -- curl http://api
# Connection refused

# Debug:

# 1. Check Endpoints
$ kubectl get endpoints api
# Should list Pod IPs
# If empty → No Pods match Service selector

# 2. Check Service ports
$ kubectl describe service api
# Verify Port and TargetPort correct

# 3. Test direct Pod access
$ kubectl get pods -l app=api -o wide
# Get Pod IP

$ kubectl exec test-pod -- curl http://<pod-ip>:<pod-port>
# If works → Service config issue
# If fails → Pod issue

# 4. Check kube-proxy
$ kubectl get pods -n kube-system | grep kube-proxy
# Should be Running on all Nodes

$ kubectl logs -n kube-system kube-proxy-xxx
# Check for errors
```

---

### Issue 3: External Traffic Can't Reach NodePort

```bash
# NodePort Service exists
$ kubectl get service webapp
# TYPE: NodePort, PORT: 80:30080/TCP

# But can't access from external
$ curl http://<node-ip>:30080
# Connection refused

# Debug:

# 1. Check firewall on Nodes
# Ensure port 30080 open

# 2. Check Service Endpoints
$ kubectl get endpoints webapp
# Should have Pod IPs

# 3. Test from within Node
$ ssh node-ip
$ curl http://localhost:30080
# If works → Firewall issue
# If fails → Service/Pod issue

# 4. Check Node IP is correct
$ kubectl get nodes -o wide
# Use INTERNAL-IP or EXTERNAL-IP

# 5. For cloud providers, check Security Groups
# AWS: Security Group rules
# GCP: Firewall rules
# Azure: NSG rules
```

---

## 💡 Best Practices

### 1. Implement Network Policies

```yaml
# Start with default deny
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

# Then whitelist specific traffic
```

### 2. Separate Namespaces

```bash
# Don't put everything in 'default'
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database

# Use namespace isolation
```

### 3. Label Everything

```yaml
metadata:
  labels:
    app: webapp
    tier: frontend
    environment: production
    version: v1.0

# Makes Network Policy selectors easier
```

### 4. Test Network Policies

```bash
# Before applying to production:
# 1. Test in staging
# 2. Monitor for deniedpackets
# 3. Gradually roll out
# 4. Have rollback plan
```

---

## 🎓 Kiểm Tra Hiểu Biết

**1. CNI Plugin làm gì?**
<details>
<summary>Xem đáp án</summary>

CNI Plugin configures network cho Pods:
1. Assigns IP address
2. Creates network interface
3. Configures routes
4. Enables Pod-to-Pod communication
5. Implements network policies

Examples: Calico, Flannel, Cilium, Weave

K8s delegates networking to CNI plugins!
</details>

**2. Network Policy: podSelector: {} nghĩa là gì?**
<details>
<summary>Xem đáp án</summary>

**Empty podSelector = ALL Pods trong namespace**

```yaml
spec:
  podSelector: {}  # Matches ALL Pods

vs

spec:
  podSelector:
    matchLabels:
      app: backend  # Matches only app=backend Pods
```

Use empty selector cho default deny policies.
</details>

**3. Làm sao test Network Policy đang block traffic?**
<details>
<summary>Xem đáp án</summary>

```bash
# Method 1: Direct test
kubectl exec source-pod -- curl http://destination-pod-ip
# If denied → Network Policy blocking

# Method 2: Check CNI logs (Calico example)
kubectl logs -n kube-system calico-node-xxx | grep denied

# Method 3: Temporarily delete policy
kubectl delete networkpolicy <policy-name>
# Test again
# If works → Policy was blocking

# Method 4: Describe policy
kubectl describe networkpolicy <policy-name>
# Check selectors match correctly
```
</details>

---

## 🎯 Key Takeaways

1. **K8s Network Model**
   - Pod-to-Pod communication without NAT
   - Flat network (all Pods can reach each other)
   - CNI plugins implement networking

2. **CNI Plugins**
   - Calico: Production, policies
   - Flannel: Simple, dev
   - Cilium: Modern, eBPF
   - Weave: Encryption

3. **Network Policies = Firewall**
   - Control ingress/egress traffic
   - Pod-level segmentation
   - Requires CNI support (not all plugins support)

4. **Default: No Restrictions**
   - Without Network Policies: Open
   - Implement default deny + whitelist
   - Security best practice

5. **Troubleshooting Order**
   - Check Pod IPs assigned
   - Check CNI plugin running
   - Check Network Policies
   - Check Service/Endpoints
   - Check kube-proxy

---

## 🚀 Hoàn Thành Phần 5!

Networking fundamentals mastered!

**Next:** [Phần 6: Configuration →](../06-configuration/README.md)

Learn về ConfigMaps, Secrets, và configuration management!

---

[⬅️ 5.2. Ingress](./02-ingress.md) | [🏠 Mục Lục](../README.md) | [📂 Phần 5: Networking](./README.md) | [➡️ Phần 6: Configuration](../06-configuration/README.md)

