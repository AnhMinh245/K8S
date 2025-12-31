# 6.1. ConfigMaps - External Configuration

> Manage application configuration externally

---

## 🎯 Mục Tiêu Học

- ✅ Hiểu **ConfigMap là gì** và **TẠI SAO cần**
- ✅ Tạo ConfigMaps từ **literals, files, directories**
- ✅ Consume ConfigMaps as **env vars, volumes**
- ✅ **Best practices** cho configuration management
- ✅ **Troubleshoot** ConfigMap issues

---

## 📦 ConfigMap Là Gì?

**ConfigMap** = K8s object lưu configuration data dạng key-value pairs.

### TẠI SAO Cần?

**Without ConfigMap:**
```dockerfile
# Config hardcoded trong image
ENV DATABASE_HOST=postgres.prod.com
ENV MAX_CONNECTIONS=100
ENV LOG_LEVEL=info

Problems:
❌ Different envs need different configs
❌ Must rebuild image for config changes
❌ Can't separate code from config
❌ Hard to manage
```

**With ConfigMap:**
```yaml
# Config external, inject at runtime
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: postgres.prod.com
  MAX_CONNECTIONS: "100"
  LOG_LEVEL: info

Benefits:
✓ Same image, different configs
✓ Change config without rebuild
✓ Separation of concerns
✓ Easy to manage
```

---

## 📝 Create ConfigMaps

### From Literals

```bash
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=postgres \
  --from-literal=MAX_CONNECTIONS=100

kubectl get configmap app-config -o yaml
```

### From File

```bash
# app.conf
DATABASE_HOST=postgres
MAX_CONNECTIONS=100
LOG_LEVEL=info

kubectl create configmap app-config --from-file=app.conf

# Or specify key
kubectl create configmap app-config --from-file=config=app.conf
```

### From Directory

```bash
# configs/
# ├── database.conf
# ├── cache.conf
# └── logging.conf

kubectl create configmap app-config --from-file=configs/

# Creates ConfigMap với 3 keys (filenames)
```

### YAML Manifest

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Simple key-value
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  DATABASE_PORT: "5432"
  MAX_CONNECTIONS: "100"
  
  # Multi-line value
  app.conf: |
    database:
      host: postgres
      port: 5432
    cache:
      host: redis
      port: 6379
    logging:
      level: info
```

---

## 🎮 Consume ConfigMaps

### As Environment Variables

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
    # Single env var from ConfigMap
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    
    # All keys as env vars
    envFrom:
    - configMapRef:
        name: app-config
```

### As Volume (Files)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config

# Files created:
# /etc/config/DATABASE_HOST
# /etc/config/MAX_CONNECTIONS
# /etc/config/app.conf
```

---

## 💡 Best Practices

```yaml
# 1. Namespace ConfigMaps
metadata:
  name: prod-config
  namespace: production

# 2. Use labels
metadata:
  labels:
    app: webapp
    env: production

# 3. Immutable ConfigMaps (K8s 1.19+)
immutable: true  # Prevents accidental changes

# 4. Separate sensitive data
# Use ConfigMap for non-sensitive
# Use Secret for passwords, tokens
```

---

## 🎯 Key Takeaways

1. **ConfigMap = External Config**
   - Separate config from code
   - Same image, different configs
   - Easy updates

2. **Creation Methods**
   - Literals: Quick single values
   - Files: Full config files
   - Directories: Multiple files
   - YAML: Declarative

3. **Consumption**
   - Environment variables: Simple
   - Volumes: Files, hot-reload possible

4. **vs Secrets**
   - ConfigMap: Non-sensitive data
   - Secret: Passwords, tokens

---

[🏠 Mục Lục](../README.md) | [➡️ 6.2. Secrets](./02-secrets.md)

