# 11.4. Security Best Practices

> Hardening Kubernetes cluster cho production

---

## 🎯 Mục Tiêu

- ✅ Implement RBAC
- ✅ Pod Security Standards
- ✅ Network Policies
- ✅ Secrets management
- ✅ Image security
- ✅ Compliance

---

## 🔐 RBAC (Role-Based Access Control)

### Concepts

```
User/ServiceAccount
        ↓
    [RBAC]
        ↓
  Role/ClusterRole (what can be done)
        +
  RoleBinding/ClusterRoleBinding (who can do it)
        ↓
    Resources (pods, services, etc.)
```

### Example: Developer Role

**Role (namespace-scoped):**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
# Pods
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]

- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]

# Deployments
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]

# Services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch"]

# ConfigMaps (read-only)
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]

# Secrets (NO access)
# Not included = denied
```

**RoleBinding:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development
subjects:
- kind: User
  name: john@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole (cluster-wide)

**Read-only cluster viewer:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-viewer
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["get", "list", "watch"]

- apiGroups: ["apps"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]

- apiGroups: ["batch"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

### ServiceAccount cho Pods

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: myapp-sa
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      serviceAccountName: myapp-sa  # Use custom SA
      automountServiceAccountToken: true
      containers:
      - name: app
        image: myapp:v1
```

---

## 🛡️ Pod Security Standards

### Three Levels

**1. Privileged:** Unrestricted (không recommend)  
**2. Baseline:** Minimal restrictions  
**3. Restricted:** Heavily restricted (production recommend)  

### Enforce với Pod Security Admission

**Namespace labels:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted policy
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    
    # Warn if baseline violations
    pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/warn-version: latest
    
    # Audit all
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

### Secure Pod Spec

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  template:
    spec:
      # Security context (pod-level)
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      
      containers:
      - name: app
        image: myapp:v1
        
        # Security context (container-level)
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
        
        # Resource limits (prevent DoS)
        resources:
          limits:
            cpu: 1000m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 128Mi
        
        # Volume mounts
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
```

---

## 🌐 Network Policies

### Default Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # Apply to all pods
  policyTypes:
  - Ingress
  - Egress
```

### Allow Specific Traffic

**Frontend → Backend:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  # Allow from frontend
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  
  egress:
  # Allow to database
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

---

## 🔑 Secrets Management

### External Secrets Operator

**Install:**
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets-system \
  --create-namespace
```

**SecretStore (Google Secret Manager):**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcpsm
  namespace: production
spec:
  provider:
    gcpsm:
      projectID: "my-project"
      auth:
        workloadIdentity:
          clusterLocation: us-central1
          clusterName: production-cluster
          serviceAccountRef:
            name: external-secrets-sa
```

**ExternalSecret:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcpsm
    kind: SecretStore
  
  target:
    name: app-secrets
    creationPolicy: Owner
  
  data:
  - secretKey: database-password
    remoteRef:
      key: prod-db-password
  
  - secretKey: api-key
    remoteRef:
      key: prod-api-key
```

### HashiCorp Vault Integration

**Vault Agent Injector:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-secret-database: "secret/data/database"
        vault.hashicorp.com/agent-inject-template-database: |
          {{- with secret "secret/data/database" -}}
          export DB_PASSWORD="{{ .Data.data.password }}"
          {{- end }}
    spec:
      serviceAccountName: myapp
      containers:
      - name: app
        image: myapp:v1
        command: ["/bin/sh"]
        args:
        - -c
        - source /vault/secrets/database && ./app
```

---

## 🖼️ Image Security

### Image Scanning với Trivy

```bash
# Scan image
trivy image myapp:v1

# Scan trong CI/CD
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:v1
```

### Image Signing với Cosign

```bash
# Generate key pair
cosign generate-key-pair

# Sign image
cosign sign --key cosign.key gcr.io/project/myapp:v1

# Verify
cosign verify --key cosign.pub gcr.io/project/myapp:v1
```

### Policy Enforcement

**Kyverno policy:**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-signature
spec:
  validationFailureAction: enforce
  rules:
  - name: verify-signature
    match:
      resources:
        kinds:
        - Pod
    verifyImages:
    - imageReferences:
      - "gcr.io/myproject/*"
      attestors:
      - count: 1
        entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              ...
              -----END PUBLIC KEY-----
```

---

## 📋 Compliance

### CIS Kubernetes Benchmark

**kube-bench:**
```bash
# Run benchmark
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Check results
kubectl logs job/kube-bench
```

### Audit Logging

**Enable audit:**
```yaml
# kube-apiserver flags
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=10
--audit-log-maxsize=100
```

**audit-policy.yaml:**
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log metadata for all requests
- level: Metadata
  omitStages:
  - RequestReceived

# Log request/response for secrets
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# Don't log read requests for configmaps
- level: None
  resources:
  - group: ""
    resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
```

---

## ✅ Security Checklist

### Cluster Level
```
□ RBAC enabled
□ Anonymous auth disabled
□ Admission controllers enabled
□ Audit logging configured
□ API server secured
□ etcd encrypted
□ Network policies enforced
```

### Workload Level
```
□ Non-root containers
□ Read-only root filesystem
□ No privilege escalation
□ Drop all capabilities
□ Resource limits set
□ Security context configured
□ Secrets not in env vars
```

### Image Security
```
□ Images scanned
□ Images signed
□ Private registry
□ No :latest tag
□ Minimal base images
```

### Network Security
```
□ Network policies
□ TLS everywhere
□ mTLS (service mesh)
□ Ingress with TLS
```

---

## 🎓 Key Takeaways

**1. RBAC:**
- Least privilege principle
- ServiceAccounts cho pods
- Regular audits

**2. Pod Security:**
- Enforce restricted policy
- Non-root containers
- Drop capabilities

**3. Network Policies:**
- Default deny
- Explicit allow
- Egress control

**4. Secrets:**
- External secrets operator
- Never in Git
- Rotate regularly

**5. Compliance:**
- CIS benchmarks
- Audit logging
- Regular scanning

---

[⬅️ 11.3. CI/CD](./03-cicd-integration.md) | [➡️ 11.5. Monitoring](./05-monitoring-logging.md) | [🏠 Mục Lục](../README.md)

