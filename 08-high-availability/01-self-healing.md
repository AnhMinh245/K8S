# 8.1. Self-Healing - Tự Phục Hồi

> Kubernetes automatically detects and fixes issues

---

## 🎯 Self-Healing Scenarios

### 1. Container Crash

```
Container exits with error code
  ↓
kubelet detects exit
  ↓
Restart container (based on restartPolicy)
  ↓
Container running again ✅
```

**RestartPolicy:**
- `Always` (default): Always restart
- `OnFailure`: Restart if exit code != 0
- `Never`: Don't restart

---

### 2. Failed Health Check

```
Liveness probe fails 3 times
  ↓
kubelet restarts container
  ↓
Container healthy again ✅
```

---

### 3. Pod Deleted

```
Deployment: replicas=3
Current: 2 Pods (1 deleted)
  ↓
ReplicaSet Controller detects
  ↓
Creates new Pod
  ↓
Back to 3 Pods ✅
```

---

### 4. Node Down

```
Node 2 goes down
  ↓
Node Controller marks NotReady
  ↓
After 5 minutes, evict Pods
  ↓
Scheduler assigns to other Nodes
  ↓
Pods running on healthy Nodes ✅
```

---

## 🔄 Control Loop

**Every controller runs:**

```
loop forever {
  desired = get_from_etcd()
  current = observe()
  
  if current != desired {
    fix_difference()
  }
  
  sleep(interval)
}
```

---

## 🎓 Key Takeaways

1. **Automatic recovery:** No manual intervention
2. **Container crash:** Restart automatically
3. **Health checks:** Liveness probe → restart
4. **Pod deleted:** ReplicaSet creates new
5. **Node down:** Pods migrated
6. **Control loop:** Continuously reconciles state

---

[➡️ 8.2. Health Checks](./02-health-checks.md) | [🏠 Mục Lục](../README.md)

