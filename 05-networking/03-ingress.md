# 5.3. Ingress - HTTP/HTTPS Router

> Ingress provides HTTP/HTTPS routing to Services

---

## 🎯 Vấn Đề Ingress Giải Quyết

**Problem với nhiều Services:**
```
10 Services cần expose:
  web-service → LoadBalancer (1 IP)
  api-service → LoadBalancer (1 IP)
  blog-service → LoadBalancer (1 IP)
  ...
  
Result: 10 LoadBalancers = 10 IPs = $$$$ expensive!
```

**Solution: Ingress**
```
1 Ingress (1 LoadBalancer, 1 IP)
  → Routes traffic based on URL:
    
    example.com/         → web-service
    example.com/api/     → api-service
    blog.example.com     → blog-service
    
Cost: 1 LoadBalancer instead of 10! 💰
```

---

## 📝 Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
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
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: blog.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80
```

---

## 🔄 Ingress Controller

**Ingress = Rules only, need Controller to execute**

### Popular Controllers

**1. NGINX Ingress Controller** (Most popular)
- Stable, feature-rich
- Good performance

**2. Traefik**
- Modern, easy config
- Auto-discovery

**3. HAProxy**
- High performance
- Advanced features

**4. Cloud-specific**
- AWS ALB Ingress Controller
- GCE Ingress Controller

### Install NGINX Ingress

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

---

## 🌐 Routing Types

### 1. Path-Based Routing

```yaml
rules:
- host: example.com
  http:
    paths:
    - path: /web
      backend:
        service:
          name: web-service
    - path: /api
      backend:
        service:
          name: api-service
```

**Traffic:**
```
http://example.com/web  → web-service
http://example.com/api  → api-service
```

---

### 2. Host-Based Routing

```yaml
rules:
- host: web.example.com
  http:
    paths:
    - path: /
      backend:
        service:
          name: web-service

- host: api.example.com
  http:
    paths:
    - path: /
      backend:
        service:
          name: api-service
```

**Traffic:**
```
http://web.example.com  → web-service
http://api.example.com  → api-service
```

---

## 🔒 TLS/SSL

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: web-service
```

**Result:**
```
https://example.com → TLS termination at Ingress
                    → Forward to web-service (HTTP)
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
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
---
# Service (ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  rules:
  - host: myapp.example.com
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

**Access:**
```
http://myapp.example.com → Ingress Controller
                         → web-service
                         → web Pods
```

---

## 🎓 Key Takeaways

1. **Ingress:** HTTP/HTTPS router for Services
2. **1 IP:** Multiple services via 1 LoadBalancer
3. **Routing:** Path-based or Host-based
4. **Controller needed:** NGINX, Traefik, etc.
5. **TLS termination:** SSL at Ingress
6. **Cost effective:** Better than multiple LoadBalancers

---

**Chúc mừng!** Hoàn thành **Phần 5: Networking** 🎉

👉 [**Phần 6: Configuration**](../06-configuration/README.md)

---

[⬅️ 5.2. Service](./02-services.md) | [⬆️ Phần 5](./README.md) | [🏠 Mục Lục](../README.md)

