# 8.2. Health Checks - Probes

> Health checks ensure Pods are healthy

---

## 🎯 Three Types of Probes

### 1. Liveness Probe
**Question:** "Is container alive?"

**If fails:** Restart container

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

**Use case:** Detect deadlock, hung process

---

### 2. Readiness Probe
**Question:** "Is container ready for traffic?"

**If fails:** Remove from Service endpoints

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

**Use case:** App loading data, connecting to database

---

### 3. Startup Probe
**Question:** "Has container started?"

**If fails:** Restart container

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30  # 30 * 10s = 5 min timeout
  periodSeconds: 10
```

**Use case:** Slow-starting applications (Java, legacy apps)

---

## 🔧 Probe Methods

### 1. HTTP GET
```yaml
httpGet:
  path: /health
  port: 8080
  httpHeaders:
  - name: Custom-Header
    value: Value

# Success: HTTP 200-399
# Failure: Other codes or timeout
```

### 2. TCP Socket
```yaml
tcpSocket:
  port: 3306

# Success: TCP connection established
# Failure: Connection refused
```

### 3. Exec Command
```yaml
exec:
  command:
  - cat
  - /tmp/healthy

# Success: Exit code 0
# Failure: Non-zero exit code
```

---

## ⚙️ Configuration

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30  # Wait 30s after start
  periodSeconds: 10        # Check every 10s
  timeoutSeconds: 5        # Timeout after 5s
  successThreshold: 1      # 1 success = healthy
  failureThreshold: 3      # 3 failures = restart
```

---

## 🎯 Best Practices

### ✅ DO

1. **Always use readiness probe**
```yaml
readinessProbe:
  httpGet:
    path: /ready
```

2. **Liveness for hung processes**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
```

3. **Startup probe for slow apps**
```yaml
startupProbe:
  failureThreshold: 30
  periodSeconds: 10
```

### ❌ DON'T

1. **Same endpoint for liveness and readiness**
2. **Too aggressive timing** (restart loops)
3. **Heavy health checks** (slow response)

---

## 🎓 Key Takeaways

1. **Liveness:** Restart if unhealthy
2. **Readiness:** Route traffic only if ready
3. **Startup:** For slow-starting apps
4. **Methods:** HTTP, TCP, Exec
5. **Always configure:** Especially readiness
6. **Tune thresholds:** Avoid false positives

---

[⬅️ 8.1. Self-Healing](./01-self-healing.md) | [➡️ 8.3. Scaling](./03-scaling.md) | [🏠 Mục Lục](../README.md)

