# 10.5. Deploying Monitoring Agents

> Deploy Datadog, Dynatrace, Prometheus agents trong K8s

---

## 🎯 Agent Deployment Pattern

**DaemonSet = Standard pattern for monitoring agents:**

```
Cluster: 5 Nodes
  Node 1 → Monitoring Agent Pod
  Node 2 → Monitoring Agent Pod
  Node 3 → Monitoring Agent Pod
  Node 4 → Monitoring Agent Pod
  Node 5 → Monitoring Agent Pod

Every Node has exactly 1 agent Pod
```

---

## 📦 1. Datadog Agent Deployment

### Architecture

```
┌──────────────────────────────────────────┐
│        Datadog Cluster Agent             │  ← Deployment (1-2 replicas)
│  • Cluster-level metrics                 │  • Coordinates node agents
│  • Custom metrics API                    │  • Reduces API calls
│  • Event collection                      │
└──────────────┬───────────────────────────┘
               │
               ↓ (coordinates)
┌──────────────────────────────────────────┐
│      Datadog Node Agent (DaemonSet)      │
│  ┌────────────────────────────────────┐  │
│  │ • Container metrics (cAdvisor)     │  │
│  │ • Log collection                   │  │
│  │ • APM traces                       │  │
│  │ • Process monitoring               │  │
│  │ • Network monitoring               │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

### DaemonSet Configuration

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: datadog-agent
  namespace: datadog
spec:
  selector:
    matchLabels:
      app: datadog-agent
  template:
    metadata:
      labels:
        app: datadog-agent
    spec:
      serviceAccountName: datadog-agent
      containers:
      - name: agent
        image: gcr.io/datadoghq/agent:latest
        env:
        # API key from Secret
        - name: DD_API_KEY
          valueFrom:
            secretKeyRef:
              name: datadog-secret
              key: api-key
        # Kubernetes integration
        - name: DD_KUBERNETES_KUBELET_HOST
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        - name: DD_COLLECT_KUBERNETES_EVENTS
          value: "true"
        - name: DD_LEADER_ELECTION
          value: "true"
        # Enable features
        - name: DD_LOGS_ENABLED
          value: "true"
        - name: DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL
          value: "true"
        - name: DD_APM_ENABLED
          value: "true"
        - name: DD_PROCESS_AGENT_ENABLED
          value: "true"
        # Cluster name
        - name: DD_CLUSTER_NAME
          value: "prod-cluster"
        # Tags
        - name: DD_TAGS
          value: "env:prod region:us-west-2"
        
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        
        volumeMounts:
        # Docker socket for container metrics
        - name: dockersocket
          mountPath: /var/run/docker.sock
          readOnly: true
        # Host /proc for process monitoring
        - name: procdir
          mountPath: /host/proc
          readOnly: true
        # Host /sys for system metrics
        - name: cgroups
          mountPath: /host/sys/fs/cgroup
          readOnly: true
        # Container logs
        - name: pointerdir
          mountPath: /opt/datadog-agent/run
        - name: logpodpath
          mountPath: /var/log/pods
          readOnly: true
        - name: logcontainerpath
          mountPath: /var/lib/docker/containers
          readOnly: true
      
      volumes:
      - name: dockersocket
        hostPath:
          path: /var/run/docker.sock
      - name: procdir
        hostPath:
          path: /proc
      - name: cgroups
        hostPath:
          path: /sys/fs/cgroup
      - name: pointerdir
        hostPath:
          path: /opt/datadog-agent/run
      - name: logpodpath
        hostPath:
          path: /var/log/pods
      - name: logcontainerpath
        hostPath:
          path: /var/lib/docker/containers
      
      tolerations:
      # Run on master nodes too
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
```

---

## 🔑 2. RBAC Configuration

### ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: datadog-agent
  namespace: datadog
automountServiceAccountToken: true
```

### ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: datadog-agent
rules:
# Metrics and metadata
- apiGroups: [""]
  resources:
  - nodes
  - nodes/metrics
  - nodes/stats
  - nodes/proxy
  - pods
  - services
  - endpoints
  - persistentvolumes
  - persistentvolumeclaims
  - replicationcontrollers
  - limitranges
  - resourcequotas
  verbs: ["get", "list", "watch"]

# Metrics Server
- apiGroups: ["metrics.k8s.io"]
  resources:
  - pods
  - nodes
  verbs: ["get", "list"]

# Events
- apiGroups: [""]
  resources:
  - events
  verbs: ["get", "list", "watch", "create"]

# Workloads
- apiGroups: ["apps"]
  resources:
  - deployments
  - replicasets
  - daemonsets
  - statefulsets
  verbs: ["get", "list", "watch"]

# Batch jobs
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["get", "list", "watch"]

# Networking
- apiGroups: ["networking.k8s.io"]
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]

# HPA
- apiGroups: ["autoscaling"]
  resources:
  - horizontalpodautoscalers
  verbs: ["get", "list", "watch"]

# ConfigMaps (for leader election)
- apiGroups: [""]
  resources:
  - configmaps
  resourceNames:
  - datadog-leader-election
  verbs: ["get", "update"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["create"]

# Leases (for leader election in K8s 1.14+)
- apiGroups: ["coordination.k8s.io"]
  resources:
  - leases
  verbs: ["get", "list", "watch", "create", "update"]
```

### ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: datadog-agent
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: datadog-agent
subjects:
- kind: ServiceAccount
  name: datadog-agent
  namespace: datadog
```

---

## 🎯 3. Dynatrace OneAgent

### Operator-Based Deployment

```yaml
# DynaKube CR
apiVersion: dynatrace.com/v1beta1
kind: DynaKube
metadata:
  name: dynakube
  namespace: dynatrace
spec:
  # API endpoint
  apiUrl: https://abc12345.live.dynatrace.com/api
  
  # Tokens (from Secret)
  tokens: dynakube
  
  # OneAgent configuration
  oneAgent:
    classicFullStack:
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      resources:
        requests:
          cpu: 100m
          memory: 512Mi
        limits:
          cpu: 300m
          memory: 1.5Gi
  
  # ActiveGate for routing
  activeGate:
    capabilities:
    - kubernetes-monitoring
    - routing
    replicas: 1
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
      limits:
        cpu: 1000m
        memory: 1.5Gi
```

**Operator creates DaemonSet automatically**

---

## 📊 4. Prometheus Stack

### Prometheus Operator

```yaml
# ServiceMonitor for auto-discovery
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: web-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: web
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Node Exporter (DaemonSet)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        args:
        - '--path.procfs=/host/proc'
        - '--path.sysfs=/host/sys'
        - '--path.rootfs=/host/root'
        - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
        ports:
        - containerPort: 9100
          hostPort: 9100
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /host/root
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
```

---

## ⚙️ 5. Agent Configuration Best Practices

### Resource Limits

```yaml
resources:
  requests:
    cpu: 200m      # Minimum guaranteed
    memory: 256Mi
  limits:
    cpu: 500m      # Maximum allowed
    memory: 512Mi  # OOMKilled if exceeded

# Too low → Agent can't function
# Too high → Wastes resources (runs on ALL nodes!)
```

### Tolerations

```yaml
tolerations:
# Run on master nodes
- key: node-role.kubernetes.io/master
  effect: NoSchedule

# Run on tainted nodes
- key: node.kubernetes.io/not-ready
  effect: NoExecute
  tolerationSeconds: 300

# Run on nodes being drained
- key: node.kubernetes.io/unreachable
  effect: NoExecute
  tolerationSeconds: 300
```

### Node Selection

```yaml
# Only run on specific nodes
nodeSelector:
  monitoring: enabled

# Or use affinity
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: node-role.kubernetes.io/worker
          operator: Exists
```

---

## 🔒 6. Security Considerations

### ✅ DO

**1. Use Secrets for credentials**
```yaml
env:
- name: DD_API_KEY
  valueFrom:
    secretKeyRef:
      name: datadog-secret
      key: api-key
```

**2. Read-only mounts when possible**
```yaml
volumeMounts:
- name: dockersocket
  mountPath: /var/run/docker.sock
  readOnly: true  # ← Read-only
```

**3. Least-privilege RBAC**
```yaml
# Only permissions actually needed
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]  # Not "delete", "update"
```

**4. Network policies**
```yaml
# Allow agent to reach API server and backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: datadog-agent
spec:
  podSelector:
    matchLabels:
      app: datadog-agent
  egress:
  - to:
    - namespaceSelector: {}  # K8s API
  - ports:
    - protocol: TCP
      port: 443  # Datadog backend
```

---

### ❌ DON'T

1. **Hardcode API keys** in YAML
2. **Give excessive RBAC** permissions
3. **Run as privileged** unless absolutely needed
4. **No resource limits** → Agent can OOM Node
5. **Mount entire root filesystem** unless needed

---

## 🎓 Key Takeaways

1. **DaemonSet:** Standard pattern for monitoring agents
2. **HostPath volumes:** Access Node filesystem for metrics/logs
3. **RBAC:** Required for API access (list/watch)
4. **ServiceAccount:** Identity for agent
5. **Tolerations:** Run on master/tainted nodes
6. **Resource limits:** Essential (runs on ALL nodes)
7. **Secrets:** Never hardcode credentials

---

## 🚀 Tiếp Theo

👉 [10.6. Events & Audit Logs](./06-events-audit-logs.md)

---

[⬅️ 10.4. Service Discovery](./04-service-discovery-monitoring.md) | [⬆️ Phần 10](./README.md) | [🏠 Mục Lục](../README.md)

