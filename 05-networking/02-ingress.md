# 5.2. Ingress - HTTP/HTTPS Routing Layer

> Expose HTTP/HTTPS services với advanced routing

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **Ingress là gì** và **TẠI SAO cần**
- ✅ **Setup Ingress Controller** (NGINX, Traefik)
- ✅ Implement **path-based** và **host-based routing**
- ✅ Configure **TLS/SSL** termination
- ✅ **Best practices** cho production
- ✅ **Troubleshoot** Ingress issues

---

## 📦 Ingress Là Gì?

### Định Nghĩa

**Ingress** = API object manages external HTTP/HTTPS access to Services, provides:
- **HTTP/HTTPS routing**
- **Virtual hosting** (multiple domains)
- **Path-based routing**
- **TLS/SSL termination**
- **Load balancing**

### Giải Thích Bằng Ví Dụ

**Problem Without Ingress:**

```
🌐 EXPOSE NHIỀU SERVICES EXTERNALLY

Cách 1: LoadBalancer cho mỗi Service
├── Service 1 (LoadBalancer) → External IP: 35.xx.xx.1  $$$
├── Service 2 (LoadBalancer) → External IP: 35.xx.xx.2  $$$
├── Service 3 (LoadBalancer) → External IP: 35.xx.xx.3  $$$
└── Service 4 (LoadBalancer) → External IP: 35.xx.xx.4  $$$

Problems:
❌ One LoadBalancer per Service (expensive!)
❌ Multiple public IPs to manage
❌ No HTTP routing (can't do /api vs /web)
❌ No virtual hosting (can't do api.example.com vs web.example.com)
❌ TLS termination ở mỗi Service
❌ Cost $$$ scales với số Services
```

**Solution With Ingress:**

```
🌐 ONE INGRESS = MANY SERVICES

                   ┌─────────────────────┐
Internet → 35.xx.xx.1 │  INGRESS (Single LB) │ $
                   └──────────┬──────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
    example.com/api   example.com/web   api.example.com
            ↓                 ↓                 ↓
      Service 1         Service 2         Service 3
       (ClusterIP)       (ClusterIP)       (ClusterIP)
            ↓                 ↓                 ↓
         Pods              Pods              Pods

Benefits:
✓ One LoadBalancer for all (cheap!)
✓ One public IP
✓ HTTP routing (path-based, host-based)
✓ Virtual hosting (multiple domains)
✓ TLS termination centralized
✓ Cost $ (one LB vs many)
```

---

## 🤔 TẠI SAO Cần Ingress?

### LoadBalancer Limitations

```yaml
# Without Ingress: Multiple LoadBalancers
---
# Service 1
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: LoadBalancer  # $$$
  selector:
    app: api
  ports:
  - port: 80

# Service 2
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer  # $$$
  selector:
    app: web
  ports:
  - port: 80

# Cost: 2 LoadBalancers
# Management: 2 IPs, 2 DNS entries
# Routing: Can't do path-based routing
```

### Ingress Solution

```yaml
# With Ingress: One LoadBalancer
---
# Services (ClusterIP - internal only)
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP  # Internal
  selector:
    app: api
  ports:
  - port: 80

---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP  # Internal
  selector:
    app: web
  ports:
  - port: 80

---
# Ingress (Routes external traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80

# Cost: 1 LoadBalancer (Ingress Controller)
# Management: 1 IP, 1 DNS entry
# Routing: Path-based, host-based, TLS, etc.
```

---

## 🏗️ Ingress Architecture

### Components

```
┌────────────────────────────────────────────────────┐
│              INGRESS ARCHITECTURE                  │
└────────────────────────────────────────────────────┘

Internet
   ↓
[DNS: example.com → 35.xx.xx.xx]
   ↓
┌──────────────────────────────────────────┐
│  Cloud Load Balancer                     │
│  IP: 35.xx.xx.xx                        │
│  Type: LoadBalancer Service             │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│  INGRESS CONTROLLER                      │
│  (NGINX, Traefik, HAProxy)              │
│  - Watches Ingress resources            │
│  - Configures routing rules             │
│  - Handles TLS termination              │
│  - Load balances requests               │
└──────────────┬───────────────────────────┘
               │
      ┌────────┴────────┐
      │  Ingress Rules  │
      │  (API objects)  │
      └────────┬────────┘
               │
     ┌─────────┼─────────┐
     │         │         │
example.com/api │   api.example.com
     ↓         ↓         ↓
  Service A  Service B  Service C
  (ClusterIP)(ClusterIP)(ClusterIP)
     ↓         ↓         ↓
   Pods      Pods      Pods
```

---

## ⚙️ Ingress Controller Setup

### Install NGINX Ingress Controller

**Method 1: Helm (Recommended)**

```bash
# Add Helm repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Verify
kubectl get pods -n ingress-nginx
kubectl get service -n ingress-nginx

# Output:
# NAME                                 TYPE           EXTERNAL-IP
# nginx-ingress-ingress-nginx-controller LoadBalancer   35.xx.xx.xx
```

**Method 2: kubectl apply**

```bash
# Apply official manifest
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Verify
kubectl get pods -n ingress-nginx
kubectl get service ingress-nginx-controller -n ingress-nginx
```

**For Minikube:**

```bash
# Enable Ingress addon
minikube addons enable ingress

# Verify
kubectl get pods -n ingress-nginx
```

---

## 📝 Ingress YAML Examples

### Path-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      # example.com/api → api-service
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      
      # example.com/web → web-service
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      
      # example.com/ → frontend-service (default)
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

**Traffic flow:**
```
http://example.com/api/users → api-service (port 80)
http://example.com/web/dashboard → web-service (port 80)
http://example.com/ → frontend-service (port 80)
```

---

### Host-Based Routing (Virtual Hosting)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
  # api.example.com → api-service
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
  
  # web.example.com → web-service
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  
  # admin.example.com → admin-service
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

**Traffic flow:**
```
http://api.example.com/users → api-service
http://web.example.com/dashboard → web-service
http://admin.example.com/settings → admin-service
```

---

### TLS/SSL Termination

**Step 1: Create TLS Secret**

```bash
# Generate self-signed cert (for testing)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=example.com/O=example"

# Create Secret
kubectl create secret tls example-tls \
  --cert=tls.crt \
  --key=tls.key
```

**Step 2: Configure Ingress với TLS**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    - www.example.com
    secretName: example-tls  # References TLS Secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

**Traffic flow:**
```
https://example.com → TLS termination at Ingress → http://web-service (internal)

Benefits:
✓ HTTPS external (encrypted)
✓ HTTP internal (faster, simpler)
✓ Centralized TLS management
```

---

### Default Backend (404 handler)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: with-default-backend
spec:
  defaultBackend:  # Handles unmatched requests
    service:
      name: error-service
      port:
        number: 80
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

**Traffic flow:**
```
http://example.com/api → api-service
http://example.com/unknown → error-service (default backend)
```

---

## 🎮 Hands-On: Complete Ingress Setup

### Scenario: Multi-Service Application

```yaml
# 1. Create Deployments và Services
---
# API Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args:
        - "-text=API Response"
        ports:
        - containerPort: 5678

---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 5678

---
# Web Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: hashicorp/http-echo
        args:
        - "-text=Web Response"
        ports:
        - containerPort: 5678

---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 5678

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

```bash
# Apply all
kubectl apply -f complete-ingress.yaml

# Get Ingress IP
kubectl get ingress main-ingress

# Output:
# NAME           CLASS   HOSTS   ADDRESS         PORTS   AGE
# main-ingress   nginx   *       35.xxx.xxx.xxx  80      1m

# Test API endpoint
curl http://35.xxx.xxx.xxx/api
# Output: API Response

# Test Web endpoint
curl http://35.xxx.xxx.xxx/web
# Output: Web Response
```

---

## 🔧 Ingress Annotations

### Common NGINX Annotations

```yaml
metadata:
  annotations:
    # Rewrite target path
    nginx.ingress.kubernetes.io/rewrite-target: /
    
    # SSL redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    
    # Custom timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    
    # Auth (Basic Auth)
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
    
    # Whitelist IPs
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"
    
    # Custom headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Custom-Header: value";
```

---

## 🐛 Troubleshooting Ingress

### Issue 1: Ingress Has No Address

```bash
$ kubectl get ingress
NAME           CLASS   HOSTS   ADDRESS   PORTS   AGE
main-ingress   nginx   *       <none>    80      5m

# No ADDRESS assigned

# Debug:
# 1. Check Ingress Controller running
$ kubectl get pods -n ingress-nginx
# Should see ingress-controller Pod Running

# 2. Check Ingress Controller Service
$ kubectl get service -n ingress-nginx
# Should have EXTERNAL-IP (or pending if LoadBalancer provisioning)

# 3. If local cluster (minikube):
$ minikube tunnel
# Creates route to LoadBalancer services

# 4. Check events
$ kubectl describe ingress main-ingress
```

---

### Issue 2: 404 Not Found

```bash
# Ingress exists, but returns 404
$ curl http://35.xxx.xxx.xxx/api
# 404 page not found

# Debug:
# 1. Check Ingress rules
$ kubectl describe ingress main-ingress
# Verify paths configured correctly

# 2. Check Service exists
$ kubectl get service api-service
# Should exist

# 3. Check Service Endpoints
$ kubectl get endpoints api-service
# Should have Pod IPs

# 4. Check backend Service port
$ kubectl get service api-service -o yaml | grep port:
# Verify port matches Ingress backend port

# 5. Test Service directly (from within cluster)
$ kubectl run curl --image=curlimages/curl -it --rm -- curl http://api-service
# If this works, issue is Ingress config

# Common fixes:
# - Fix path in Ingress
# - Fix Service port
# - Add rewrite-target annotation if needed
```

---

### Issue 3: TLS Not Working

```bash
# HTTPS returns certificate error
$ curl https://example.com
# SSL certificate problem

# Debug:
# 1. Check TLS Secret exists
$ kubectl get secret example-tls
# Should exist

# 2. Check Secret data
$ kubectl get secret example-tls -o yaml
# Should have tls.crt and tls.key

# 3. Check Ingress TLS config
$ kubectl describe ingress tls-ingress
# TLS:
#   example-tls terminates example.com

# 4. Check certificate validity
$ kubectl get secret example-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
# Not Before: ...
# Not After: ...  (check not expired)

# 5. Check certificate CN/SAN
$ kubectl get secret example-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text | grep -A1 "Subject:"
# Should match host in Ingress
```

---

## 💡 Best Practices

### 1. Use IngressClass

```yaml
# Recommended (K8s 1.18+)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx  # Explicit class
  rules: [...]

# vs deprecated annotation
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx  # Deprecated
```

### 2. Centralize TLS Certificates

```yaml
# Use cert-manager for automatic cert issuance
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-cert
spec:
  secretName: example-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - example.com
  - www.example.com
```

### 3. Set Resource Limits

```yaml
# Ingress Controller Deployment
spec:
  template:
    spec:
      containers:
      - name: controller
        resources:
          requests:
            cpu: 100m
            memory: 90Mi
          limits:
            cpu: 200m
            memory: 200Mi
```

### 4. Monitor Ingress

```bash
# Enable metrics
kubectl get service -n ingress-nginx
# Port 10254 for metrics

# Prometheus scraping
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "10254"
```

---

## 🎓 Kiểm Tra Hiểu Biết

**1. Ingress vs LoadBalancer Service?**
<details>
<summary>Xem đáp án</summary>

**LoadBalancer Service:**
- Layer 4 (TCP/UDP)
- One LoadBalancer per Service
- No HTTP routing
- Cost: $ per Service

**Ingress:**
- Layer 7 (HTTP/HTTPS)
- One LoadBalancer for many Services
- HTTP routing (path, host)
- TLS termination
- Cost: $ for one Ingress Controller

**Use:** Ingress for HTTP/HTTPS services (cheaper, more features)
</details>

**2. Path /api vs /api/ - có khác không?**
<details>
<summary>Xem đáp án</summary>

**Depends on pathType:**

```yaml
# Prefix (matches /api, /api/, /api/users)
pathType: Prefix
path: /api

# Exact (matches only /api, not /api/)
pathType: Exact
path: /api

# ImplementationSpecific (depends on Ingress Controller)
pathType: ImplementationSpecific

Recommendation: Use Prefix for flexibility
```
</details>

**3. Làm sao route example.com và www.example.com về same Service?**
<details>
<summary>Xem đáp án</summary>

**Option 1: Multiple rules**
```yaml
rules:
- host: example.com
  http:
    paths: [...]
- host: www.example.com
  http:
    paths: [...]
```

**Option 2: TLS với multiple hosts**
```yaml
tls:
- hosts:
  - example.com
  - www.example.com
  secretName: example-tls
rules:
- host: example.com
  http: [...]
- host: www.example.com
  http: [...]
```
</details>

---

## 🎯 Key Takeaways

1. **Ingress = HTTP Router**
   - Layer 7 load balancing
   - Path và host-based routing
   - TLS termination

2. **Requires Ingress Controller**
   - NGINX, Traefik, HAProxy
   - Deploy first before Ingress
   - One controller for multiple Ingresses

3. **Cost-Effective**
   - One LoadBalancer for many Services
   - vs One LoadBalancer per Service
   - Significant savings

4. **Path Types**
   - Prefix: Matches /path và /path/*
   - Exact: Matches only /path
   - ImplementationSpecific: Controller-dependent

5. **TLS Centralized**
   - TLS Secret với crt + key
   - Termination at Ingress
   - Internal traffic plain HTTP

---

## 🚀 Tiếp Theo

Ingress routing mastered! Next: Pod Networking fundamentals!

**Next:** [5.3. Pod Networking →](./03-pod-networking.md)

---

[⬅️ 5.1. Services](./01-services.md) | [🏠 Mục Lục](../README.md) | [📂 Phần 5: Networking](./README.md) | [➡️ 5.3. Pod Networking](./03-pod-networking.md)

