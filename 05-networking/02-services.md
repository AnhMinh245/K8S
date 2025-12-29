# 5.2. Service - Expose Ứng Dụng

> Service provides stable endpoint cho Pods

---

## 🎯 Vấn Đề Service Giải Quyết

**Problem:**
```
Deployment: 3 Pods
  web-abc123: 10.1.1.5 (today)
  web-xyz789: 10.1.1.6
  web-def456: 10.1.1.7

Tomorrow:
  web-abc123 crashed → Replaced with new IP
  web-ghi012: 10.1.1.8 (NEW IP!)
  
Client: Which IP to call? 🤔
```

**Solution: Service**
```
Service: web-service
  ClusterIP: 10.0.0.100 (STABLE)
  → Load balance to: 10.1.1.5, 10.1.1.6, 10.1.1.7
  
Client: Always call 10.0.0.100
  → Service routes to healthy Pods ✅
```

---

## 📝 Service Types

### 1. ClusterIP (Default)

**Internal service, cluster-only access**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP  # Default
  selector:
    app: web
  ports:
  - port: 80        # Service port
    targetPort: 8080  # Pod port
```

**Access:**
```bash
# Within cluster
curl http://web-service:80
curl http://10.0.0.100:80

# From outside cluster
❌ Not accessible
```

**Use case:** Internal microservices communication

---

### 2. NodePort

**Expose on each Node's IP:PORT**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # 30000-32767 range
```

**Access:**
```bash
# From outside
http://<NodeIP>:30080
http://192.168.1.10:30080
http://192.168.1.11:30080  # Any Node IP works

# Automatically creates ClusterIP too
http://web-nodeport:80  (internal)
```

**Use case:** Development, testing, small clusters

---

### 3. LoadBalancer

**Cloud load balancer (AWS ELB, GCP LB, Azure LB)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

**Access:**
```bash
# External IP provided by cloud
http://a1b2c3d4.us-west-2.elb.amazonaws.com

# Automatically creates NodePort + ClusterIP
```

**Use case:** Production, expose single service to internet

---

### 4. ExternalName

**Map to external DNS name**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com
```

**Access:**
```bash
# Pods call
curl http://external-db:3306

# K8s resolves to
curl http://database.example.com:3306
```

**Use case:** External databases, APIs outside cluster

---

## 🔄 Service Workflow

```
1. Create Service
   kubectl apply -f service.yaml

2. Service gets ClusterIP
   IP: 10.0.0.100

3. Endpoints Controller watches
   Finds Pods matching selector (app=web)
   
4. Creates Endpoints object
   Endpoints:
     - 10.1.1.5:8080
     - 10.1.1.6:8080
     - 10.1.1.7:8080

5. kube-proxy on each Node
   Creates iptables/IPVS rules
   
6. Traffic flows
   Client → 10.0.0.100:80
   → kube-proxy routes to one of:
     10.1.1.5, 10.1.1.6, or 10.1.1.7
```

---

## 🎯 Complete Example

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web  # ← Service selector matches this
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web  # ← Matches Deployment labels
  ports:
  - protocol: TCP
    port: 80        # Service port
    targetPort: 80  # Container port
  type: ClusterIP
```

**Result:**
```
3 Pods created with label app=web
Service finds these Pods via selector
Traffic to web-service:80 → Load balanced to 3 Pods
```

---

## 🌐 Headless Service

**No ClusterIP, direct Pod IPs**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None  # ← Headless
  selector:
    app: mysql
  ports:
  - port: 3306
```

**DNS returns all Pod IPs:**
```bash
nslookup mysql

# Returns:
mysql.default.svc.cluster.local
  Address: 10.1.1.5
  Address: 10.1.1.6
  Address: 10.1.1.7
```

**Use case:** StatefulSet, need to address specific Pods

---

## 📊 Session Affinity

**Stick to same Pod**

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
```

**Behavior:**
```
Client A (IP: 1.2.3.4):
  Request 1 → Pod-1
  Request 2 → Pod-1 (same Pod!)
  Request 3 → Pod-1

Client B (IP: 5.6.7.8):
  Request 1 → Pod-2
  Request 2 → Pod-2
```

---

## 💡 Best Practices

### ✅ DO

1. **Use ClusterIP for internal services**
2. **Use LoadBalancer sparingly** (cost per LB)
3. **Set proper selectors** (match Pod labels)
4. **Name ports** for readability
```yaml
ports:
- name: http
  port: 80
- name: https
  port: 443
```

5. **Health checks on Pods** (readiness probes)

### ❌ DON'T

1. **Multiple LoadBalancer Services** (use Ingress instead)
2. **NodePort in production** (use LoadBalancer or Ingress)
3. **Wrong selectors** (Service finds no Pods)
4. **Forget targetPort** if Pod port different

---

## 🎓 Key Takeaways

1. **Service:** Stable endpoint for Pods
2. **ClusterIP:** Internal (default)
3. **NodePort:** Expose on Node IP:PORT
4. **LoadBalancer:** Cloud LB, external access
5. **Selector:** Finds Pods by labels
6. **Endpoints:** List of Pod IPs
7. **kube-proxy:** Implements Service routing

---

[⬅️ 5.1. Pod Networking](./01-pod-networking.md) | [➡️ 5.3. Ingress](./03-ingress.md) | [🏠 Mục Lục](../README.md)

