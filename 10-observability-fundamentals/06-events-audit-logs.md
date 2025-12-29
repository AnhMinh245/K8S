# 10.6. Events & Audit Logs

> K8s Events và Audit Logs cho observability

---

## 🎯 K8s Events

**Events = K8s records of significant occurrences**

```
Examples:
  • Pod scheduled
  • Image pulled
  • Container started
  • Container failed
  • OOMKilled
  • Node NotReady
  • Volume mount failed
```

---

## 📋 1. Event Types

### Normal Events

```
✅ Pod created
✅ Pod scheduled
✅ Image pulled successfully
✅ Container started
✅ Volume mounted
✅ Service endpoint updated
```

### Warning Events

```
⚠️ Image pull failed
⚠️ Container crashed (CrashLoopBackOff)
⚠️ OOMKilled (out of memory)
⚠️ Failed scheduling (insufficient resources)
⚠️ Liveness probe failed
⚠️ Volume mount failed
⚠️ Node NotReady
```

---

## 🔍 2. Viewing Events

### kubectl get events

```bash
# All events in namespace
kubectl get events

# Sort by timestamp
kubectl get events --sort-by='.lastTimestamp'

# Watch events live
kubectl get events --watch

# Events for specific object
kubectl describe pod web-abc123

# Events in all namespaces
kubectl get events --all-namespaces
```

### Example Output

```
LAST SEEN   TYPE      REASON              OBJECT               MESSAGE
2m          Normal    Scheduled           pod/web-abc123       Successfully assigned default/web-abc123 to node-1
2m          Normal    Pulling             pod/web-abc123       Pulling image "nginx:1.21"
2m          Normal    Pulled              pod/web-abc123       Successfully pulled image
2m          Normal    Created             pod/web-abc123       Created container nginx
2m          Normal    Started             pod/web-abc123       Started container nginx
1m          Warning   BackOff             pod/web-abc123       Back-off restarting failed container
30s         Warning   FailedMount         pod/web-abc123       Unable to mount volume: timeout expired
```

---

## 📊 3. Event Structure

```yaml
apiVersion: v1
kind: Event
metadata:
  name: web-abc123.17a1b2c3d4e5f6g7
  namespace: default
involvedObject:
  apiVersion: v1
  kind: Pod
  name: web-abc123
  namespace: default
  uid: 1234-5678-9abc-def0
reason: FailedScheduling
message: "0/3 nodes are available: 3 Insufficient cpu."
source:
  component: default-scheduler
firstTimestamp: "2024-01-15T10:30:00Z"
lastTimestamp: "2024-01-15T10:35:00Z"
count: 5
type: Warning
```

---

## 🔔 4. Monitoring Events

### Datadog Event Collection

**Datadog Cluster Agent collects events:**

```yaml
env:
- name: DD_COLLECT_KUBERNETES_EVENTS
  value: "true"
- name: DD_LEADER_ELECTION
  value: "true"  # Only leader collects events
```

**Events appear in Datadog:**
```
Event stream shows:
  • Pod OOMKilled → Alert
  • Image pull failed → Warning
  • Node NotReady → Critical
  • Correlate with metrics/logs
```

**RBAC needed:**
```yaml
rules:
- apiGroups: [""]
  resources:
  - events
  verbs: ["get", "list", "watch", "create"]
```

---

### Alert on Events

**Example: Alert on OOMKilled:**

```yaml
# Datadog monitor
Query: events("sources:kubernetes reason:OOMKilled")
Threshold: count > 0 in last 5 minutes
Alert: "@pagerduty @slack-alerts"
Message: |
  Pod OOMKilled: {{pod_name.name}}
  Namespace: {{kube_namespace.name}}
  Node: {{host.name}}
  
  Action: Increase memory limits
  Runbook: https://runbook.com/oomkilled
```

---

### Event-Based Dashboards

```
Dashboard: "K8s Health"

Widget 1: Event timeline
  • Timeline of Warning/Error events
  • Color-coded by severity

Widget 2: Top reasons
  • Top 10 event reasons (CrashLoopBackOff, OOMKilled, etc.)
  • Bar chart

Widget 3: Events by namespace
  • Heatmap showing event frequency per namespace

Widget 4: Pod restart count
  • Correlate events with restart metrics
```

---

## 📝 5. Audit Logs

**K8s Audit Logs = Record of API server requests**

### What's Logged

```
Every API call:
  • Who: User, ServiceAccount, or system component
  • What: Action (create, update, delete, get)
  • When: Timestamp
  • Where: Which resource (pod, service, etc.)
  • Result: Success or failure
```

### Audit Log Example

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "abc123-def456",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/default/pods",
  "verb": "create",
  "user": {
    "username": "john@company.com",
    "groups": ["developers"]
  },
  "sourceIPs": ["10.0.0.5"],
  "objectRef": {
    "resource": "pods",
    "namespace": "default",
    "name": "web-abc123"
  },
  "responseStatus": {
    "code": 201
  },
  "requestReceivedTimestamp": "2024-01-15T10:30:00.123456Z",
  "stageTimestamp": "2024-01-15T10:30:00.234567Z"
}
```

---

## ⚙️ 6. Audit Log Configuration

**API Server flags:**

```yaml
# kube-apiserver config
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=10
--audit-log-maxsize=100
```

**Audit Policy Example:**

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log metadata for all requests
- level: Metadata

# Don't log read-only requests
- level: None
  verbs: ["get", "list", "watch"]

# Log request and response for Secrets
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# Log metadata for Pod changes
- level: Metadata
  resources:
  - group: ""
    resources: ["pods"]
  verbs: ["create", "update", "delete"]
```

---

## 📊 7. Audit Logs for Observability

### Security Auditing

```
Questions audit logs answer:
  • Who deleted this Pod?
  • Who changed the ConfigMap?
  • Failed login attempts?
  • Unauthorized API access?
  • Secret access patterns?
```

**Query in log aggregation tool:**
```
# Who deleted production Pods?
verb:delete
resource:pods
namespace:production

# Secret access audit
resource:secrets
verb:get
user:*
```

---

### Compliance

```
Compliance requirements (SOC2, PCI-DSS, HIPAA):
  • Track all changes to production
  • Audit privileged access
  • Retain logs for X months
  • Alert on suspicious activity

Audit logs provide:
  ✅ Who did what, when
  ✅ Change history
  ✅ Access logs
  ✅ Failure audit trail
```

---

### Troubleshooting

```
Debugging scenarios:
  • "Pod disappeared" → Check audit logs for delete event
  • "Config changed mysteriously" → Find who updated ConfigMap
  • "Permission denied" → Check RBAC decision logs
```

---

## 🔍 8. Event & Audit Log Best Practices

### ✅ DO

**1. Forward events to monitoring backend**
```yaml
# Datadog, Dynatrace, ELK
DD_COLLECT_KUBERNETES_EVENTS: "true"
```

**2. Alert on critical events**
```
• OOMKilled
• CrashLoopBackOff
• ImagePullBackOff
• Node NotReady
• FailedScheduling
```

**3. Retain audit logs**
```yaml
--audit-log-maxage=90  # 90 days retention
```

**4. Ship audit logs off-cluster**
```
Centralized logging:
  • Datadog
  • Elasticsearch
  • Splunk
  • AWS CloudWatch
```

**5. Configure audit policy**
```yaml
# Balance detail vs volume
- level: Metadata  # Most requests
- level: RequestResponse  # Sensitive resources only
```

---

### ❌ DON'T

1. **Ignore Warning events** → Lead to outages
2. **No audit logs** → Security blind spot
3. **Log everything at RequestResponse level** → Huge volume
4. **Store audit logs only on API server** → Lost if server fails
5. **No alerting on events** → Miss critical issues

---

## 🎓 Key Takeaways

1. **Events:** K8s records significant occurrences (scheduled, failed, etc.)
2. **Monitoring events:** Datadog/Dynatrace collect and alert on events
3. **Event types:** Normal (info) and Warning (issues)
4. **Audit logs:** Record all API server requests (who, what, when)
5. **Audit use cases:** Security, compliance, troubleshooting
6. **Best practice:** Forward events and audit logs to central backend
7. **Alerting:** Critical events (OOMKilled, CrashLoop) should alert

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. K8s Events là gì? Khác gì với audit logs?
2. Làm sao xem Events cho một Pod?
3. Event types nào nên alert?
4. Audit logs record thông tin gì?
5. Tại sao cần forward events/audit logs off-cluster?
6. Audit log levels khác nhau như thế nào?

---

## 🎉 Chúc Mừng!

Bạn đã hoàn thành **Phần 10: Observability Fundamentals**!

Bây giờ bạn hiểu:
- ✅ Metrics architecture (Metrics Server, cAdvisor)
- ✅ Logging patterns (stdout/stderr, log collectors)
- ✅ Labels & Annotations strategy
- ✅ Service discovery mechanisms
- ✅ Deploy monitoring agents (DaemonSet, RBAC)
- ✅ Events và Audit logs

**Kiến thức này là nền tảng để:**
- Deploy Datadog, Dynatrace, Prometheus
- Troubleshoot monitoring issues
- Design observability-friendly applications
- Implement security auditing

---

## 🚀 Bước Tiếp Theo

**Practice:**
1. Deploy metrics-server
2. Deploy log collector (Fluentd/Filebeat)
3. Label Pods consistently
4. Setup monitoring agent (Datadog/Prometheus)
5. Create dashboards và alerts

**Advanced topics:**
- Service mesh observability (Istio)
- Distributed tracing (Jaeger, Zipkin)
- eBPF-based monitoring (Cilium, Pixie)
- Cost monitoring (Kubecost)

---

[⬅️ 10.5. Deploying Agents](./05-deploying-monitoring-agents.md) | [⬆️ Phần 10](./README.md) | [🏠 Mục Lục Chính](../README.md)

