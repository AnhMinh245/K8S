# 6.1. ConfigMap

> Tách biệt configuration khỏi container image

---

## 🎯 Vấn Đề ConfigMap Giải Quyết

**Problem:**
```
Hard-code config trong code:
  DATABASE_URL = "mysql://prod-db:3306"
  
Change config → Rebuild image → Redeploy
  → Slow, inefficient ❌
```

**Solution: ConfigMap**
```
Config externalized:
  ConfigMap: database_url = "mysql://prod-db:3306"
  
Change config → Update ConfigMap → Restart Pods
  → Fast, no rebuild ✅
```

---

## 📝 Create ConfigMap

### 1. From Literal Values

```bash
kubectl create configmap app-config \
  --from-literal=database_url=mysql://db:3306 \
  --from-literal=log_level=info
```

### 2. From File

```bash
# app.properties
database_url=mysql://db:3306
log_level=info
cache_ttl=3600

kubectl create configmap app-config \
  --from-file=app.properties
```

### 3. From YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "mysql://db:3306"
  log_level: "info"
  cache_ttl: "3600"
  app.conf: |
    server {
      listen 80;
      server_name example.com;
    }
```

---

## 🔧 Use ConfigMap

### 1. Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    env:
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_url
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log_level
```

### 2. All Keys as Env Vars

```yaml
spec:
  containers:
  - name: app
    envFrom:
    - configMapRef:
        name: app-config
```

### 3. Mount as Volume (Files)

```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config

# Result:
# /etc/config/database_url (file)
# /etc/config/log_level (file)
# /etc/config/app.conf (file)
```

---

## 💡 Best Practices

### ✅ DO
- Separate config per environment (dev, prod)
- Version control ConfigMaps
- Use for non-sensitive data only

### ❌ DON'T
- Store passwords, API keys (use Secret)
- Make ConfigMaps too large (1 MB limit)

---

[➡️ 6.2. Secret](./02-secrets.md) | [🏠 Mục Lục](../README.md)

