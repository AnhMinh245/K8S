# 📘 Phần 6: Configuration - Quản Lý Cấu Hình

> ConfigMaps, Secrets, và configuration management

---

## 🎯 Mục Tiêu Phần Này

✅ **Separate config from code**  
✅ **Manage non-sensitive data** với ConfigMaps  
✅ **Secure sensitive data** với Secrets  
✅ **Best practices** cho configuration management  
✅ **External secrets** integration  

---

## 📚 Nội Dung

### [6.1. ConfigMaps - External Configuration](./01-configmaps.md) ⭐⭐⭐⭐

**ConfigMap = Non-sensitive configuration data**

**Key Points:**
```
✓ Store config as key-value pairs
✓ Create from literals, files, directories
✓ Consume as env vars or volumes
✓ Same image, different configs
✓ Update without rebuild
```

**Commands:**
```bash
# Create
kubectl create configmap app-config --from-literal=KEY=value

# Use in Pod
env:
- name: DB_HOST
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: DB_HOST
```

---

### [6.2. Secrets - Sensitive Data](./02-secrets.md) ⭐⭐⭐⭐⭐

**Secret = Passwords, tokens, keys**

**Key Points:**
```
✓ Base64-encoded
✓ Can be encrypted at rest
✓ RBAC-controlled access
✓ Never commit to Git
✓ Use external secrets management (Vault)
```

**Security:**
```yaml
# Enable encryption at rest
# Use External Secrets Operator
# Sealed Secrets for GitOps
# RBAC: Limit who can read Secrets
# Rotate regularly
```

**Commands:**
```bash
# Create
kubectl create secret generic db-secret \
  --from-literal=password=P@ssw0rd

# Use in Pod
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

---

## 🎯 ConfigMap vs Secret

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Data type** | Non-sensitive | Sensitive |
| **Encoding** | Plain text | Base64 |
| **Examples** | DB host, log level | Passwords, tokens |
| **Encryption** | No | Yes (at rest) |
| **RBAC** | Less strict | Strict |
| **Use when** | Public config | Private credentials |

**Rule:** Never put passwords/tokens trong ConfigMaps!

---

## 🎓 Self-Assessment

**Checkpoint:**
```
□ Create ConfigMaps from literals/files
□ Consume ConfigMaps as env vars/volumes
□ Create Secrets securely
□ Consume Secrets correctly
□ Understand encryption at rest
□ Know external secrets options
```

**If all checked → Ready for Phần 7! 🎉**

---

## 💡 Best Practices

```yaml
# 1. Separate sensitive vs non-sensitive
ConfigMap: DB_HOST, LOG_LEVEL
Secret: DB_PASSWORD, API_KEY

# 2. Use external secrets management
# Vault, AWS Secrets Manager, External Secrets Operator

# 3. Enable encryption at rest
# EncryptionConfiguration for kube-apiserver

# 4. Immutable ConfigMaps (prevent accidents)
immutable: true

# 5. RBAC: Limit Secret access
# Only necessary ServiceAccounts

# 6. Never commit Secrets to Git
# Use Sealed Secrets or External Secrets

# 7. Rotate Secrets regularly
# Automate rotation where possible

# 8. Audit Secret access
# Monitor who accesses Secrets when
```

---

## 🎯 Key Takeaways

1. **ConfigMap = External Config**
   - Separate config from code
   - Non-sensitive data only
   - Easy updates

2. **Secret = Secure Storage**
   - Passwords, tokens, keys
   - Base64-encoded
   - Encrypted at rest (if configured)

3. **Security First**
   - External secrets management
   - RBAC access control
   - Never commit Secrets
   - Rotate regularly

4. **Consumption Patterns**
   - Env vars: Simple, widely supported
   - Volumes: More secure, hot-reload

5. **Production Setup**
   - External Secrets Operator + Vault
   - Sealed Secrets for GitOps
   - Encryption at rest enabled
   - Regular audits

---

## 🚀 Tiếp Theo

**Next:** [Phần 7: Storage →](../07-storage/README.md)

Persistent storage, Volumes, PVs, PVCs!

---

[⬅️ Phần 5: Networking](../05-networking/README.md) | [🏠 Mục Lục Chính](../README.md) | [➡️ Phần 7: Storage](../07-storage/README.md)
