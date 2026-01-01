# 8.2. Health Checks - Probes

> Đảm bảo Pods luôn healthy và sẵn sàng nhận traffic

---

## 📖 Mục Lục

1. [Health Checks là gì?](#-health-checks-là-gì)
2. [3 Types of Probes](#-3-types-of-probes)
3. [Liveness Probe Deep Dive](#-liveness-probe-deep-dive)
4. [Readiness Probe Deep Dive](#-readiness-probe-deep-dive)
5. [Startup Probe Deep Dive](#-startup-probe-deep-dive)
6. [Probe Methods](#-probe-methods)
7. [Configuration Parameters](#-configuration-parameters)
8. [Hands-on Labs](#-hands-on-labs)
9. [Troubleshooting](#-troubleshooting)
10. [Best Practices](#-best-practices)

---

## 🤔 Health Checks là gì?

### Định nghĩa

**Health Checks (Probes)** là cơ chế để Kubernetes:
- 🔍 **Monitor:** Liên tục kiểm tra trạng thái container
- 🚦 **Control:** Quyết định routing traffic (readiness)
- 🔧 **Heal:** Tự động restart khi unhealthy (liveness)
- ⏰ **Timing:** Xử lý slow-starting apps (startup)

### Vấn đề nếu không có Health Checks

**❌ Scenario: Database connection pool timeout**

```
00:00 - Pod starts
00:05 - Container running (Status: Running)
00:05 - Service sends traffic to Pod
00:05 - App still connecting to database... ❌
00:10 - Users get errors: "Connection timeout"
00:30 - App finally connected to DB ✅
00:30 - Now ready for traffic!

Problem: 25 seconds of failed requests! 🔥
```

**✅ With Readiness Probe:**

```
00:00 - Pod starts
00:05 - Container running (Status: Running)
00:05 - Readiness probe: GET /ready → 503 (not ready) ❌
00:05 - Pod NOT added to Service endpoints
00:05 - NO traffic sent to this Pod ✅
00:30 - App connected to DB
00:30 - Readiness probe: GET /ready → 200 (ready) ✅
00:30 - Pod added to Service endpoints
00:31 - Traffic can now be sent!

Result: Zero failed requests! 🎉
```

---

## 🎯 3 Types of Probes

### Quick Comparison

| Probe | Question | Failure Action | Use Case |
|-------|----------|----------------|----------|
| **Liveness** | Is alive? | **Restart container** | Detect deadlock, hung process |
| **Readiness** | Ready for traffic? | **Remove from Service** | Wait for dependencies (DB, cache) |
| **Startup** | Has started? | **Restart container** | Slow-starting apps (Java, legacy) |

### Visual Flow

```
┌─────────────────────────────────────────────────────────┐
│              PROBE LIFECYCLE                            │
└─────────────────────────────────────────────────────────┘

Container starts
    ↓
STARTUP PROBE (if configured)
    ├─ Running until success
    ├─ Blocks liveness & readiness
    ├─ Success → Enable liveness & readiness
    └─ Failure → Restart container
    ↓
LIVENESS PROBE
    ├─ Running forever
    ├─ Success → Container alive ✅
    └─ Failure (3x) → Restart container 🔄
    ↓
READINESS PROBE
    ├─ Running forever
    ├─ Success → Add to Service endpoints 🚦
    └─ Failure → Remove from Service (no restart!)
```

### Decision Tree

```
Need health checks?
    ↓
Is app slow to start (>30s)?
    ├─ YES → Use STARTUP probe
    └─ NO  → Skip startup
    ↓
Can app hang/deadlock?
    ├─ YES → Use LIVENESS probe
    └─ NO  → Maybe skip liveness
    ↓
Does app need warm-up (DB, cache)?
    ├─ YES → MUST use READINESS probe ✅
    └─ NO  → Still use readiness (best practice)
```

---

## 💓 Liveness Probe Deep Dive

### Purpose

**Detect:** Container process running but app is stuck (deadlock, infinite loop, out of memory)

**Action:** Restart container

### Real-world Example

**Node.js app with memory leak:**

```javascript
// app.js (buggy code)
let leakyArray = [];

setInterval(() => {
  leakyArray.push(new Array(1000000));  // ← Memory leak!
}, 1000);

app.get('/api/data', (req, res) => {
  // Eventually runs out of memory
  // Process hangs, no response
});
```

**Timeline WITHOUT liveness probe:**

```
00:00 - App starts
10:00 - Memory at 80% (still responding)
20:00 - Memory at 95% (slow responses)
25:00 - Memory at 99% (app hangs, no response)
25:00 - Process still running! (ps aux shows node process)
25:00 - Users get timeouts ❌
∞     - App never recovers (stuck forever!)
```

**Timeline WITH liveness probe:**

```
00:00 - App starts
10:00 - Memory at 80%
        └── Liveness: GET /health → 200 OK ✅
20:00 - Memory at 95%
        └── Liveness: GET /health → Timeout (5s) ❌ (1/3)
20:10 - Liveness: GET /health → Timeout ❌ (2/3)
20:20 - Liveness: GET /health → Timeout ❌ (3/3)
20:20 - kubelet: "3 failures, restarting!" 🔄
20:25 - Container restarted (fresh memory)
20:30 - Liveness: GET /health → 200 OK ✅
        
Total stuck time: 5 minutes (vs infinite!)
```

### Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: app
    image: my-app:1.0
    livenessProbe:
      httpGet:
        path: /healthz        # ← Health endpoint
        port: 8080
        httpHeaders:
        - name: X-Probe-Type
          value: liveness
      initialDelaySeconds: 30  # ← Wait 30s (app startup)
      periodSeconds: 10        # ← Check every 10s
      timeoutSeconds: 5        # ← 5s timeout
      failureThreshold: 3      # ← 3 failures → restart
      successThreshold: 1      # ← 1 success → healthy
```

### Health Endpoint Implementation

**Good implementation:**

```go
// Go example
func healthHandler(w http.ResponseWriter, r *http.Request) {
    // Quick checks only!
    
    // 1. Check critical goroutines
    if deadlockDetected() {
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    }
    
    // 2. Check memory
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    if m.Alloc > maxMemory {
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    }
    
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
}
```

**❌ Bad implementation:**

```python
# Python example (DON'T DO THIS!)
@app.route('/health')
def health():
    # ❌ Checking database (slow, can fail)
    db.execute("SELECT 1")
    
    # ❌ Checking external API (slow, can timeout)
    requests.get("https://external-api.com/status")
    
    # ❌ Complex business logic (slow)
    process_heavy_data()
    
    return "OK"

# Problem: Health check too slow → timeout → restart loop! 🔥
```

---

## 🚦 Readiness Probe Deep Dive

### Purpose

**Detect:** Container running but app not ready for traffic (loading data, connecting to DB, warming up cache)

**Action:** Remove from Service endpoints (NOT restart!)

### Real-world Example

**E-commerce app loading product catalog:**

```python
# app.py
import time
from flask import Flask, jsonify

app = Flask(__name__)
products = []
ready = False

@app.route('/api/products')
def get_products():
    if not ready:
        return jsonify({"error": "Not ready"}), 503
    return jsonify({"products": products})

@app.route('/health')
def health():
    return "OK", 200  # ← Always OK (liveness)

@app.route('/ready')
def readiness():
    if ready:
        return "Ready", 200
    else:
        return "Not ready", 503  # ← Return 503 until ready!

# Background: Load products from DB
def load_products():
    global products, ready
    time.sleep(30)  # ← 30s to load from database
    products = fetch_from_database()
    ready = True  # ← Now ready!

if __name__ == '__main__':
    threading.Thread(target=load_products).start()
    app.run(host='0.0.0.0', port=8080)
```

**Timeline:**

```
00:00 - Pod starts
00:01 - Container running
00:01 - Liveness: GET /health → 200 OK ✅
00:01 - Readiness: GET /ready → 503 Not ready ❌
00:01 - Pod Status: Running, but Ready: 0/1
00:01 - Service endpoints: Pod NOT included ✅
00:10 - Readiness: GET /ready → 503 (still loading...)
00:20 - Readiness: GET /ready → 503 (still loading...)
00:30 - Products loaded! ready = True
00:31 - Readiness: GET /ready → 200 Ready ✅
00:31 - Pod Status: Running, Ready: 1/1 ✅
00:31 - Service endpoints: Pod added! 🎉
00:32 - Traffic starts flowing

Result: Zero failed requests during startup!
```

### Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: app
    image: my-app:1.0
    readinessProbe:
      httpGet:
        path: /ready         # ← Different from liveness!
        port: 8080
      initialDelaySeconds: 5   # ← Shorter than liveness
      periodSeconds: 5         # ← Check more frequently
      timeoutSeconds: 3
      failureThreshold: 3      # ← Remove after 3 failures
      successThreshold: 1      # ← Add after 1 success
```

### Liveness vs Readiness Comparison

```yaml
# Pod với cả 2 probes
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: app
    image: my-app:1.0
    
    # LIVENESS: "Is app alive?" (restart if not)
    livenessProbe:
      httpGet:
        path: /healthz       # ← /healthz (liveness)
        port: 8080
      initialDelaySeconds: 30  # ← Longer delay (wait for startup)
      periodSeconds: 10        # ← Less frequent
      failureThreshold: 3      # ← Restart after 3 failures
    
    # READINESS: "Ready for traffic?" (remove from Service if not)
    readinessProbe:
      httpGet:
        path: /ready         # ← /ready (different endpoint!)
        port: 8080
      initialDelaySeconds: 5   # ← Shorter delay
      periodSeconds: 5         # ← More frequent
      failureThreshold: 3      # ← Remove from Service after 3 failures
```

**Key differences:**

| Aspect | Liveness | Readiness |
|--------|----------|-----------|
| **Endpoint** | `/healthz` | `/ready` (different!) |
| **Checks** | Process alive? Deadlock? | Dependencies ready? (DB, cache) |
| **Initial delay** | Longer (30s) | Shorter (5s) |
| **Period** | Less frequent (10s) | More frequent (5s) |
| **Failure action** | **Restart container** 🔄 | **Remove from Service** 🚫 |
| **Use case** | Detect hung process | Wait for dependencies |

---

## ⏰ Startup Probe Deep Dive

### Purpose

**Problem:** Legacy/slow apps take 2-5 minutes to start. Liveness probe kills them before they finish starting!

**Solution:** Startup probe disables liveness until app is ready.

### Real-world Example

**Java Spring Boot app (slow startup):**

```
00:00 - Container starts
00:00 - JVM initializing...
00:30 - Loading Spring context...
01:00 - Connecting to database...
01:30 - Loading caches...
02:00 - App finally ready! ✅

Without startup probe:
00:30 - Liveness: GET /health → Timeout ❌ (1/3)
00:40 - Liveness: GET /health → Timeout ❌ (2/3)
00:50 - Liveness: GET /health → Timeout ❌ (3/3)
00:50 - kubelet: "Restarting!" 🔄
00:50 - Infinite restart loop! 🔥

With startup probe:
00:00 - Startup probe running (liveness disabled!)
00:10 - Startup: GET /health → Timeout (1/30)
00:20 - Startup: GET /health → Timeout (2/30)
...
02:00 - Startup: GET /health → 200 OK ✅
02:00 - Startup probe: SUCCESS! Enable liveness!
02:00 - Now liveness probe takes over
```

### Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: java-app
spec:
  containers:
  - name: app
    image: my-spring-boot-app:1.0
    
    # STARTUP: For slow-starting apps
    startupProbe:
      httpGet:
        path: /health
        port: 8080
      failureThreshold: 30   # ← 30 attempts
      periodSeconds: 10      # ← Every 10s
      # Total timeout: 30 * 10 = 300s (5 minutes)
    
    # LIVENESS: Only enabled after startup succeeds
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      periodSeconds: 10
      failureThreshold: 3
    
    # READINESS: For traffic routing
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
      failureThreshold: 3
```

### Timeline with all 3 probes

```
┌─────────────────────────────────────────────────────────┐
│         COMPLETE PROBE TIMELINE                         │
└─────────────────────────────────────────────────────────┘

00:00 - Container starts
        ├── Liveness: DISABLED (waiting for startup)
        ├── Readiness: DISABLED (waiting for startup)
        └── Startup: ENABLED ✅
        
00:10 - Startup probe attempt 1/30: Timeout ❌
00:20 - Startup probe attempt 2/30: Timeout ❌
00:30 - Startup probe attempt 3/30: Timeout ❌
...
02:00 - Startup probe attempt 12/30: 200 OK ✅
        └── Startup: SUCCESS! Disable startup probe
        
02:00 - Enable liveness & readiness probes
        ├── Liveness: ENABLED ✅
        └── Readiness: ENABLED ✅
        
02:00 - Liveness: GET /health → 200 OK ✅
02:00 - Readiness: GET /ready → 503 (loading data) ❌
02:05 - Readiness: GET /ready → 503 ❌
02:10 - Readiness: GET /ready → 200 OK ✅
02:10 - Pod added to Service endpoints! 🎉
        
02:10 - Normal operation
        ├── Liveness: Check every 10s
        └── Readiness: Check every 5s
```

---

## 🔧 Probe Methods

### 1. HTTP GET (Phổ Biến Nhất)

**Khi nào dùng:**
- ✅ Web apps, APIs
- ✅ Apps with HTTP endpoints
- ✅ Linh hoạt nhất (custom logic)

**Configuration:**

```yaml
httpGet:
  path: /health           # ← Endpoint path
  port: 8080              # ← Port number or name
  host: localhost         # ← Host (default: Pod IP)
  scheme: HTTP            # ← HTTP or HTTPS
  httpHeaders:
  - name: X-Custom-Header
    value: HealthCheck
```

**Success criteria:**
- ✅ HTTP status code: 200-399
- ❌ Bất kỳ code nào khác (400+, 500+)
- ❌ Timeout

**Example endpoint:**

```javascript
// Node.js Express
app.get('/health', (req, res) => {
  // Quick checks
  if (isHealthy()) {
    res.status(200).send('OK');
  } else {
    res.status(503).send('Unhealthy');
  }
});

app.get('/ready', (req, res) => {
  // Check dependencies
  if (db.isConnected() && cache.isReady()) {
    res.status(200).send('Ready');
  } else {
    res.status(503).send('Not ready');
  }
});
```

### 2. TCP Socket

**When to use:**
- ✅ Databases (MySQL, PostgreSQL, Redis)
- ✅ Message queues (RabbitMQ, Kafka)
- ✅ Bất kỳ TCP service nào
- ✅ No HTTP endpoint available

**Configuration:**

```yaml
tcpSocket:
  port: 3306      # ← MySQL port
  host: localhost # ← Optional (default: Pod IP)
```

**Success criteria:**
- ✅ TCP connection established
- ❌ Connection refused
- ❌ Timeout

**Example: MySQL database**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    livenessProbe:
      tcpSocket:
        port: 3306
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      tcpSocket:
        port: 3306
      initialDelaySeconds: 5
      periodSeconds: 5
```

### 3. Exec Command

**When to use:**
- ✅ Custom health checks
- ✅ File-based checks
- ✅ Non-networked apps
- ✅ Legacy apps

**Configuration:**

```yaml
exec:
  command:
  - cat
  - /tmp/healthy
```

**Success criteria:**
- ✅ Exit code: 0
- ❌ Non-zero exit code
- ❌ Timeout

**Example: PostgreSQL**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
  - name: postgres
    image: postgres:15
    livenessProbe:
      exec:
        command:
        - /bin/sh
        - -c
        - pg_isready -U postgres
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      exec:
        command:
        - /bin/sh
        - -c
        - pg_isready -U postgres && psql -U postgres -c 'SELECT 1'
      initialDelaySeconds: 5
      periodSeconds: 5
```

**Example: Redis**

```yaml
readinessProbe:
  exec:
    command:
    - redis-cli
    - ping
  periodSeconds: 5
```

### Method Comparison

| Method | Performance | Flexibility | Use Case |
|--------|-------------|-------------|----------|
| **HTTP GET** | Medium | High (custom logic) | Web apps, APIs |
| **TCP Socket** | Fast | Low (just connection) | Databases, TCP services |
| **Exec** | Slow | High (any command) | Legacy apps, custom checks |

---

## ⚙️ Configuration Parameters

### Giải Thích Tất Cả Parameters

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  
  # TIMING PARAMETERS
  
  initialDelaySeconds: 30
  # ↑ Chờ bao lâu sau khi container start mới bắt đầu probe
  # Use case: Cho app thời gian khởi động
  # Default: 0 (start immediately)
  # Typical: 10-30s
  
  periodSeconds: 10
  # ↑ Tần suất kiểm tra (check every N seconds)
  # Default: 10s
  # Typical: 5-10s (readiness), 10-30s (liveness)
  
  timeoutSeconds: 5
  # ↑ Timeout của mỗi probe request
  # Default: 1s
  # Typical: 1-5s
  
  # THRESHOLD PARAMETERS
  
  successThreshold: 1
  # ↑ Số lần success liên tiếp để coi là healthy
  # Default: 1
  # Typical: 1 (liveness/startup), 1-2 (readiness)
  # NOTE: MUST be 1 for liveness/startup!
  
  failureThreshold: 3
  # ↑ Số lần failure liên tiếp để coi là unhealthy
  # Default: 3
  # Typical: 3-5
  # Restart after: initialDelaySeconds + ((failureThreshold - 1) * periodSeconds)
```

### Timing Calculations

**Example configuration:**

```yaml
livenessProbe:
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

**Timeline:**

```
00:00 - Container starts
00:30 - First probe (after initialDelaySeconds)
00:30 - Probe timeout after 5s → Failure (1/3)
00:40 - Second probe (after periodSeconds)
00:40 - Probe timeout → Failure (2/3)
00:50 - Third probe
00:50 - Probe timeout → Failure (3/3)
00:50 - Container restarted! 🔄

Time to restart: 30 + ((3-1) * 10) = 50 seconds
Formula: initialDelaySeconds + ((failureThreshold - 1) * periodSeconds)
```

### Recommendations by App Type

**Fast-starting apps (< 10s):**

```yaml
startupProbe: null  # Not needed

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

**Slow-starting apps (30s - 2min):**

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 20  # 20 * 10 = 200s (3min 20s)
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
  failureThreshold: 3
```

**Very slow apps (Java, legacy, 2-5min):**

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30  # 30 * 10 = 300s (5 minutes)
  periodSeconds: 10
  timeoutSeconds: 5

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 15     # Less frequent
  failureThreshold: 5   # More tolerant
  timeoutSeconds: 5

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
```

---

## 🧪 Hands-on Labs

### Lab 1: HTTP Liveness Probe

```bash
# Create app với liveness probe
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: liveness-http
spec:
  containers:
  - name: app
    image: k8s.gcr.io/liveness
    args:
    - /server
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 3
      failureThreshold: 2
EOF

# Watch Pod (will restart after ~10s)
kubectl get pod liveness-http -w
```

**Expected:**
```
NAME            READY   STATUS    RESTARTS   AGE
liveness-http   1/1     Running   0          5s
liveness-http   1/1     Running   1          15s  ← Restarted!
liveness-http   1/1     Running   2          25s  ← Restarted again!
```

**Why?** The `k8s.gcr.io/liveness` image returns 200 for first 10s, then returns 500 (simulating unhealthy app).

### Lab 2: Readiness Probe with Service

```bash
# Create Deployment với readiness probe
kubectl apply -f - <<EOF
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
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
EOF

# Check Service endpoints
kubectl get endpoints web
```

**Expected:**
```
NAME   ENDPOINTS                                   AGE
web    10.1.0.5:80,10.1.0.6:80,10.1.0.7:80         1m
```

**Experiment: Make one Pod unready**

```bash
# Get Pod name
POD=$(kubectl get pods -l app=web -o jsonpath='{.items[0].metadata.name}')

# Delete index.html (readiness probe will fail!)
kubectl exec $POD -- rm /usr/share/nginx/html/index.html

# Check endpoints (Pod removed!)
kubectl get endpoints web
```

**Expected:**
```
NAME   ENDPOINTS                     AGE
web    10.1.0.6:80,10.1.0.7:80       2m
# ↑ Only 2 endpoints! (removed unready Pod)
```

**Restore:**
```bash
kubectl exec $POD -- sh -c 'echo "OK" > /usr/share/nginx/html/index.html'
```

### Lab 3: Startup Probe for Slow App

```bash
# Simulate slow-starting app
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: slow-app
spec:
  containers:
  - name: app
    image: alpine
    command:
    - /bin/sh
    - -c
    - |
      # Simulate slow startup
      sleep 60
      echo "Started!"
      # Then run HTTP server
      while true; do
        echo -e "HTTP/1.1 200 OK\n\nOK" | nc -l -p 8080
      done
    
    startupProbe:
      tcpSocket:
        port: 8080
      failureThreshold: 15  # 15 * 5 = 75s
      periodSeconds: 5
    
    livenessProbe:
      tcpSocket:
        port: 8080
      periodSeconds: 10
      failureThreshold: 3
EOF

# Watch Pod (should NOT restart)
kubectl get pod slow-app -w
```

**Expected:**
```
NAME       READY   STATUS    RESTARTS   AGE
slow-app   0/1     Running   0          10s  ← Not ready (startup probe failing)
slow-app   0/1     Running   0          30s  ← Still not ready
slow-app   1/1     Running   0          65s  ← Ready! (startup succeeded)
```

---

## 🔧 Troubleshooting

### Issue 1: CrashLoopBackOff (Liveness probe too aggressive)

**Symptoms:**
```bash
kubectl get pods
# NAME   READY   STATUS             RESTARTS   AGE
# web    0/1     CrashLoopBackOff   10         5m
```

**Debug:**
```bash
kubectl describe pod web | grep -A 10 Liveness
```

**Output:**
```
Liveness:  http-get http://:8080/healthz delay=5s timeout=1s period=5s
  Warning  Unhealthy  2m  kubelet  Liveness probe failed: Get http://10.1.0.5:8080/healthz: dial tcp 10.1.0.5:8080: connect: connection refused
```

**Cause:** App needs 30s to start, but liveness starts after 5s.

**Fix:**
```yaml
livenessProbe:
  initialDelaySeconds: 30  # ← Increase!
  
# OR use startupProbe:
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

### Issue 2: Pod never becomes Ready

**Symptoms:**
```bash
kubectl get pods
# NAME   READY   STATUS    RESTARTS   AGE
# web    0/1     Running   0          5m  ← Never becomes 1/1!
```

**Debug:**
```bash
kubectl describe pod web | grep -A 10 Readiness
```

**Output:**
```
Readiness:  http-get http://:8080/ready delay=5s timeout=1s period=5s
  Warning  Unhealthy  2m  kubelet  Readiness probe failed: Get http://10.1.0.5:8080/ready: dial tcp 10.1.0.5:8080: i/o timeout
```

**Causes:**

**A. Wrong port:**
```yaml
# App listens on 3000, but probe checks 8080!
readinessProbe:
  httpGet:
    port: 8080  # ← Wrong!

# Fix:
readinessProbe:
  httpGet:
    port: 3000  # ← Correct!
```

**B. Endpoint doesn't exist:**
```bash
# Test manually
kubectl exec web -- curl localhost:8080/ready
# 404 Not Found

# Fix: Implement /ready endpoint!
```

**C. App never becomes ready (stuck loading dependencies):**
```bash
# Check logs
kubectl logs web
# Error: Cannot connect to database!

# Fix: Check database connectivity
```

### Issue 3: Service routing to unready Pods

**Symptoms:**
- Users get errors
- Một số requests thành công, một số fail

**Debug:**
```bash
# Check Pod readiness
kubectl get pods -o wide
# NAME    READY   STATUS    RESTARTS   AGE   IP
# web-1   1/1     Running   0          5m    10.1.0.5
# web-2   0/1     Running   0          5m    10.1.0.6  ← Not ready!

# Check Service endpoints
kubectl get endpoints web
# NAME   ENDPOINTS                     AGE
# web    10.1.0.5:80,10.1.0.6:80       5m
#                     ↑ Unready Pod still in endpoints! ❌
```

**Cause:** No readiness probe configured!

**Fix:**
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 80
  periodSeconds: 5
  failureThreshold: 3
```

---

## 💡 Best Practices

### 1. Always configure readiness probe

❌ **Bad:**
```yaml
# No readiness probe
spec:
  containers:
  - name: app
    image: my-app
```

✅ **Good:**
```yaml
spec:
  containers:
  - name: app
    image: my-app
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
```

**Why?** Without readiness, Pods receive traffic immediately, even if not ready.

### 2. Use different endpoints for liveness and readiness

❌ **Bad:**
```yaml
livenessProbe:
  httpGet:
    path: /health
readinessProbe:
  httpGet:
    path: /health  # ← Same endpoint!
```

✅ **Good:**
```yaml
livenessProbe:
  httpGet:
    path: /healthz  # ← Liveness: Process alive?
readinessProbe:
  httpGet:
    path: /ready    # ← Readiness: Dependencies ready?
```

**Why?**
- Liveness: Quick checks (process alive, no deadlock)
- Readiness: Dependency checks (DB, cache, external APIs)

### 3. Liveness checks should be fast and simple

❌ **Bad:**
```python
@app.route('/health')
def health():
    # ❌ Check database (slow, can fail)
    db.query("SELECT COUNT(*) FROM users")
    
    # ❌ Check external API (slow, can timeout)
    requests.get("https://api.example.com/status", timeout=10)
    
    return "OK"
```

✅ **Good:**
```python
@app.route('/healthz')
def liveness():
    # ✅ Quick check: Process alive?
    return "OK", 200

@app.route('/ready')
def readiness():
    # ✅ Check dependencies for readiness
    if db.ping() and cache.ping():
        return "Ready", 200
    else:
        return "Not ready", 503
```

### 4. Use startup probe for slow apps

❌ **Bad:**
```yaml
# Java app takes 2min to start
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 120  # ← Guessing!
  periodSeconds: 10
  failureThreshold: 3
```

✅ **Good:**
```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30  # 30 * 10 = 300s (5min)
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 10      # ← Simple config (startup handles delay)
  failureThreshold: 3
```

### 5. Set appropriate timeouts

❌ **Bad:**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  timeoutSeconds: 30  # ← Too long!
```

✅ **Good:**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  timeoutSeconds: 3  # ← Fast timeout (health check should be quick!)
```

### 6. Don't restart on external dependency failures

❌ **Bad:**
```yaml
# Liveness probe checks database
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - mysql -h db -e 'SELECT 1'  # ← Database down → restart! ❌
```

✅ **Good:**
```yaml
# Liveness: Just check process
livenessProbe:
  httpGet:
    path: /healthz  # ← Returns 200 even if DB down
    port: 8080

# Readiness: Check database
readinessProbe:
  httpGet:
    path: /ready  # ← Returns 503 if DB down (remove from Service)
    port: 8080
```

**Why?** Restarting container won't fix external DB failure!

### 7. Monitor probe metrics

```bash
# Check probe failures
kubectl get events --sort-by='.lastTimestamp' | grep -i unhealthy

# Prometheus metrics (if monitoring installed)
# - probe_success
# - probe_duration_seconds
```

---

## 🎓 Key Takeaways

### Concepts

1. **Liveness Probe:** Restart if unhealthy (deadlock, hung process)
2. **Readiness Probe:** Remove from Service if not ready (loading dependencies)
3. **Startup Probe:** For slow-starting apps (disable liveness until ready)
4. **Methods:** HTTP GET (most common), TCP Socket (databases), Exec (custom)
5. **Different endpoints:** `/healthz` (liveness), `/ready` (readiness)
6. **Fast checks:** Liveness should be quick (< 1s)
7. **Always use readiness:** Especially important for production!

### Decision Matrix

| Scenario | Liveness | Readiness | Startup |
|----------|----------|-----------|---------|
| Fast app (< 10s) | ✅ | ✅ | ❌ |
| Slow app (> 30s) | ✅ | ✅ | ✅ |
| Database | ✅ (TCP) | ✅ (TCP) | ❌ |
| Stateless web app | ✅ | ✅ | ❌ |
| Stateful app (cache) | ✅ | ✅ (check cache) | Maybe |

### Commands

```bash
# Check Pod readiness
kubectl get pods -o wide

# Describe probes
kubectl describe pod <name> | grep -A 10 -i probe

# Check events
kubectl get events --sort-by='.lastTimestamp' | grep -i unhealthy

# Test probe manually
kubectl exec <pod> -- curl localhost:8080/health

# Watch Pod restart
kubectl get pod <name> -w

# Check Service endpoints
kubectl get endpoints <service>
```

---

**Chúc mừng!** Bạn đã thành thạo **Health Checks** trong Kubernetes! 🎉

👉 [**8.3. Scaling**](./03-scaling.md)

---

[⬅️ 8.1. Self-Healing](./01-self-healing.md) | [⬆️ Phần 8](./README.md) | [➡️ 8.3. Scaling](./03-scaling.md) | [🏠 Mục Lục](../README.md)


