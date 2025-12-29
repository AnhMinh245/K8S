# 6.2. Secret - Quản Lý Sensitive Data

> Secret for passwords, tokens, certificates

---

## 🎯 Secret vs ConfigMap

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Purpose** | Non-sensitive config | Passwords, tokens, keys |
| **Encoding** | Plain text | Base64 encoded |
| **Encryption** | No | Yes (if enabled in etcd) |
| **Access control** | Standard RBAC | Stricter RBAC |

---

## 📝 Create Secret

### 1. From Literal

```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123
```

### 2. From Files

```bash
echo -n 'admin' > username.txt
echo -n 'secret123' > password.txt

kubectl create secret generic db-secret \
  --from-file=username=username.txt \
  --from-file=password=password.txt
```

### 3. From YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=       # base64("admin")
  password: c2VjcmV0MTIz   # base64("secret123")
```

**Encode/Decode:**
```bash
# Encode
echo -n 'admin' | base64
# YWRtaW4=

# Decode
echo 'YWRtaW4=' | base64 --decode
# admin
```

---

## 🔧 Use Secret

### 1. Environment Variables

```yaml
spec:
  containers:
  - name: app
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

### 2. Mount as Files

```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: secret
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret
    secret:
      secretName: db-secret

# Result:
# /etc/secrets/username
# /etc/secrets/password
```

---

## 🔒 Secret Types

### 1. Opaque (Default)
```yaml
type: Opaque
# Generic key-value secrets
```

### 2. Docker Registry
```bash
kubectl create secret docker-registry regcred \
  --docker-server=docker.io \
  --docker-username=myuser \
  --docker-password=mypass \
  --docker-email=my@email.com
```

### 3. TLS Certificate
```bash
kubectl create secret tls tls-secret \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```

---

## 🛡️ Security Best Practices

### ✅ DO

1. **Enable encryption at rest** (etcd)
```yaml
# kube-apiserver config
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

2. **Use external secret management**
- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- Google Secret Manager

3. **RBAC restrictions**
```yaml
# Only specific ServiceAccount can read secrets
```

4. **Don't commit Secrets to Git**
```bash
# .gitignore
*.secret.yaml
secrets/
```

5. **Rotate regularly**
```bash
# Update secrets periodically
kubectl create secret generic db-secret --from-literal=password=newpass123 --dry-run=client -o yaml | kubectl apply -f -
```

### ❌ DON'T

1. **Store in plain text files** in repo
2. **Echo secrets in logs**
3. **Give broad Secret access**
4. **Use default ServiceAccount** for sensitive apps

---

## ⚠️ Important Notes

**Base64 ≠ Encryption!**
```bash
# Anyone can decode
echo 'c2VjcmV0MTIz' | base64 --decode
# secret123

# Base64 is encoding, NOT security
```

**Secrets visible to:**
- Cluster admins
- Anyone with RBAC access
- Compromise of etcd = compromise of secrets

**Solution:** Use external secret managers (Vault, AWS Secrets Manager)

---

## 🎓 Key Takeaways

1. **Secret:** For sensitive data (passwords, tokens)
2. **Base64 encoded:** Not encrypted by default!
3. **Types:** Opaque, docker-registry, TLS
4. **Use as:** Env vars or mounted files
5. **Best practice:** External secret management
6. **RBAC:** Restrict who can read Secrets
7. **Never commit:** To version control

---

**Chúc mừng!** Hoàn thành **Phần 6: Configuration** 🎉

👉 [**Phần 7: Storage**](../07-storage/README.md)

---

[⬅️ 6.1. ConfigMap](./01-configmap.md) | [⬆️ Phần 6](./README.md) | [🏠 Mục Lục](../README.md)

