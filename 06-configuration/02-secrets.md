# 6.2. Secrets - Sensitive Data Management

> Manage passwords, tokens, và sensitive configuration

---

## 🎯 Mục Tiêu Học

- ✅ Hiểu **Secrets là gì** và **khác ConfigMap**
- ✅ Tạo và manage **Secrets securely**
- ✅ Consume Secrets as **env vars, volumes**
- ✅ **Security best practices**
- ✅ **Encrypt Secrets** at rest

---

## 📦 Secret Là Gì?

**Secret** = K8s object lưu sensitive data (passwords, tokens, keys) dạng base64-encoded.

### Secret vs ConfigMap

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Purpose** | Non-sensitive config | Sensitive data |
| **Encoding** | Plain text | Base64-encoded |
| **Example** | Database host, log level | Passwords, API tokens |
| **At rest** | Plain text | Can be encrypted |
| **Size limit** | 1MB | 1MB |

**Rule:** Never put passwords/tokens trong ConfigMaps!

---

## 📝 Create Secrets

### From Literals

```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=P@ssw0rd

kubectl get secret db-secret -o yaml
# Data base64-encoded:
# data:
#   username: YWRtaW4=
#   password: UEBzc3cwcmQ=
```

### From Files

```bash
# Create files
echo -n 'admin' > username.txt
echo -n 'P@ssw0rd' > password.txt

kubectl create secret generic db-secret \
  --from-file=username.txt \
  --from-file=password.txt

# Or TLS secret
kubectl create secret tls my-tls-secret \
  --cert=cert.crt \
  --key=cert.key
```

### YAML Manifest

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # Base64-encoded values
  username: YWRtaW4=  # admin
  password: UEBzc3cwcmQ=  # P@ssw0rd

# Or use stringData (auto-encodes)
stringData:
  username: admin
  password: P@ssw0rd
```

**Encode/decode base64:**

```bash
# Encode
echo -n 'P@ssw0rd' | base64
# Output: UEBzc3cwcmQ=

# Decode
echo 'UEBzc3cwcmQ=' | base64 -d
# Output: P@ssw0rd
```

---

## 🎮 Consume Secrets

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
    # Single secret value
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    
    # All keys as env vars
    envFrom:
    - secretRef:
        name: db-secret
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
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true  # Important!
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret

# Creates files:
# /etc/secrets/username
# /etc/secrets/password
```

---

## 🔐 Secret Types

```yaml
# 1. Opaque (generic)
type: Opaque

# 2. TLS certificates
type: kubernetes.io/tls
data:
  tls.crt: <base64-cert>
  tls.key: <base64-key>

# 3. Docker registry credentials
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-config>

# 4. Basic auth
type: kubernetes.io/basic-auth
data:
  username: <base64>
  password: <base64>

# 5. SSH auth
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: <base64>

# 6. Service account token
type: kubernetes.io/service-account-token
```

---

## 🛡️ Security Best Practices

### 1. Enable Encryption at Rest

```yaml
# EncryptionConfiguration
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <32-byte-base64-key>
      - identity: {}  # Fallback
```

### 2. RBAC: Limit Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  # DO NOT allow "create", "update", "delete"
```

### 3. Use External Secrets Management

**Options:**
- **HashiCorp Vault**: Enterprise secrets management
- **AWS Secrets Manager**: AWS integration
- **Azure Key Vault**: Azure integration
- **Google Secret Manager**: GCP integration
- **External Secrets Operator**: Sync external → K8s Secrets

**Example: External Secrets Operator**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-secret  # K8s Secret name
  data:
  - secretKey: password
    remoteRef:
      key: database/password
```

### 4. Avoid Hardcoding

```yaml
# ❌ Bad: Hardcoded in YAML
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
stringData:
  password: "P@ssw0rd"  # Never commit to Git!

# ✓ Good: Reference external secret
# Use External Secrets Operator or Sealed Secrets
```

### 5. Use Sealed Secrets

```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# Encrypt Secret
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# Safe to commit sealed-secret.yaml to Git!
```

---

## 🐛 Troubleshooting

### Secret Not Found

```bash
# Check Secret exists
kubectl get secret db-secret

# Check namespace
kubectl get secret db-secret -n <namespace>

# Describe for details
kubectl describe secret db-secret
```

### Wrong Encoding

```bash
# Verify base64 encoding
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d

# Should output correct password
```

### Permission Denied

```bash
# Check RBAC permissions
kubectl auth can-i get secrets

# Check ServiceAccount permissions
kubectl describe serviceaccount <sa-name>
```

---

## 🎯 Key Takeaways

1. **Secrets = Sensitive Data**
   - Passwords, tokens, keys
   - Base64-encoded
   - Can be encrypted at rest

2. **vs ConfigMap**
   - Secrets: Sensitive
   - ConfigMap: Non-sensitive
   - Never put passwords in ConfigMaps!

3. **Security**
   - Enable encryption at rest
   - RBAC: Limit access
   - Use external secrets management (Vault)
   - Never commit to Git

4. **Consumption**
   - Environment variables (simple)
   - Volumes (files, more secure)
   - Always readOnly for volumes

5. **Best Practices**
   - External Secrets Operator
   - Sealed Secrets for GitOps
   - Rotate regularly
   - Audit access

---

[⬅️ 6.1. ConfigMaps](./01-configmaps.md) | [🏠 Mục Lục](../README.md) | [➡️ Phần 7: Storage](../07-storage/README.md)
