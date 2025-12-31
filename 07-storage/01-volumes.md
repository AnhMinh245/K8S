# 7.1. Volumes - Storage Cơ Bản

> Giải quyết vấn đề ephemeral storage trong containers

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **TẠI SAO cần Volumes** trong Kubernetes
- ✅ Phân biệt **các loại Volumes** và use cases
- ✅ Implement **shared storage** giữa containers
- ✅ Sử dụng **emptyDir, hostPath, cloud volumes**
- ✅ **Best practices** cho volume management

---

## 🔥 Vấn Đề: Ephemeral Container Storage

### Câu Chuyện Thực Tế

**Scenario: Ứng dụng Upload Files**

```
📱 USER ACTION
User uploads avatar image
  ↓
🖥️ BACKEND POD
Saves image to /uploads/avatar.jpg
  ↓
✅ SUCCESS
Image displayed successfully

--------- 5 PHÚT SAU ---------

💥 POD CRASHES (OOM)
Kubernetes restarts Pod
  ↓
📂 NEW CONTAINER
Fresh filesystem, /uploads/ trống!
  ↓
❌ IMAGE LOST
User's avatar gone! 😱
```

**Root Cause:**

```
Container filesystem is ephemeral:
├── Container writes file → Stored in container layer
├── Container terminates → Container layer deleted
└── New container starts → Fresh, empty filesystem

PROBLEM:
❌ Data loss khi restart
❌ No persistence between Pod restarts
❌ No sharing between containers
❌ No separation of data và application
```

### Solution: Kubernetes Volumes

```
Volume = External storage mounted vào container

Container writes to Volume (not container filesystem)
  ↓
Container restarts
  ↓
New container mounts same Volume
  ↓
✅ Data persists!

BENEFITS:
✓ Data survives Pod restarts
✓ Share data giữa containers trong Pod
✓ Separation of concerns (data vs app)
✓ Multiple volume types for different needs
```

---

## 📦 Hiểu Volume Workflow

### Basic Concept

```
┌─────────────────────────────────────────────┐
│              POD                            │
├─────────────────────────────────────────────┤
│                                             │
│  ┌──────────────┐      ┌──────────────┐   │
│  │ Container A  │      │ Container B  │   │
│  │              │      │              │   │
│  │ /app/logs ←──┼──────┼──→ /logs     │   │
│  └──────────────┘      └──────────────┘   │
│         ↓                      ↓            │
│    ┌────────────────────────────────┐      │
│    │   Volume: "shared-logs"        │      │
│    │   Type: emptyDir               │      │
│    │   Actual Path: /var/lib/...   │      │
│    └────────────────────────────────┘      │
│                                             │
└─────────────────────────────────────────────┘
```

**Key Points:**
- Volume định nghĩa ở Pod level (không phải container level)
- Multiple containers có thể mount cùng Volume
- Container sees mountPath (e.g., /app/logs)
- Kubernetes handles underlying storage

---

## 📝 Volume Types

### 1. emptyDir - Temporary Shared Storage

**Definition:** Temporary directory created when Pod starts, deleted when Pod terminates.

**Use Cases:**
- Cache data (không cần persist lâu dài)
- Scratch space cho computation
- Shared storage giữa containers trong Pod
- Sidecar pattern (logging, monitoring)

**YAML Example:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-logs
spec:
  containers:
  # Main application container
  - name: webapp
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
    # Nginx writes logs to /var/log/nginx/access.log
  
  # Sidecar: Log shipping container
  - name: log-shipper
    image: fluent-bit:latest
    volumeMounts:
    - name: logs-volume
      mountPath: /logs
      readOnly: true
    # Reads logs from /logs/access.log (same as nginx's /var/log/nginx)
    # Ships to central logging (Elasticsearch)
  
  volumes:
  - name: logs-volume
    emptyDir: {}  # Temporary storage, Pod-scoped
```

**Workflow:**

```
1. Pod starts
   ↓
2. K8s creates emptyDir on Node
   Location: /var/lib/kubelet/pods/<pod-id>/volumes/...
   ↓
3. Both containers mount emptyDir
   nginx → /var/log/nginx
   fluent-bit → /logs
   ↓
4. nginx writes logs
   /var/log/nginx/access.log
   ↓
5. fluent-bit reads logs
   /logs/access.log (same file!)
   Ships to Elasticsearch
   ↓
6. Pod deleted
   ↓
7. emptyDir deleted
   Logs lost (OK, already shipped!)
```

**emptyDir với Memory Backend:**

```yaml
volumes:
- name: cache-volume
  emptyDir:
    medium: Memory  # tmpfs (RAM)
    sizeLimit: 1Gi
```

Use case: In-memory cache (ultra fast, but limited size)

**Hands-On:**

```bash
# Create Pod với emptyDir
cat > emptydir-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: test-emptydir
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "while true; do date >> /data/log.txt; sleep 5; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  
  - name: reader
    image: busybox
    command: ["sh", "-c", "tail -f /data/log.txt"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  
  volumes:
  - name: shared-data
    emptyDir: {}
EOF

kubectl apply -f emptydir-pod.yaml

# Check writer logs (writing timestamps)
kubectl logs test-emptydir writer

# Check reader logs (reading same file!)
kubectl logs test-emptydir reader

# Exec vào writer container
kubectl exec -it test-emptydir -c writer -- ls -la /data

# Exec vào reader container
kubectl exec -it test-emptydir -c reader -- cat /data/log.txt

# Delete Pod
kubectl delete pod test-emptydir

# Re-create same Pod
kubectl apply -f emptydir-pod.yaml

# Check logs - New Pod = Fresh emptyDir (old data lost)
kubectl logs test-emptydir reader
# Empty! Proves emptyDir is ephemeral
```

---

### 2. hostPath - Mount Node Filesystem

**Definition:** Mounts file/directory từ Node's filesystem vào Pod.

**⚠️ WARNING:** 
- Pod on Node A → Sees Node A's filesystem
- Pod rescheduled to Node B → Sees Node B's filesystem (different data!)
- Security risk: Pod có thể access Node filesystem

**Use Cases:**
- DaemonSet accessing Node logs (e.g., /var/log)
- Monitoring agents accessing Node metrics
- Development/testing (mount code từ host)

**YAML Example:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpath
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: host-logs
      mountPath: /host-logs
      readOnly: true
  
  volumes:
  - name: host-logs
    hostPath:
      path: /var/log  # Node's /var/log
      type: Directory  # Must exist
```

**hostPath Types:**

```yaml
volumes:
- name: my-volume
  hostPath:
    path: /path/on/node
    type: <type>

# Types:
# - Directory: Must exist directory
# - DirectoryOrCreate: Create if doesn't exist
# - File: Must exist file
# - FileOrCreate: Create if doesn't exist
# - Socket: Must exist Unix socket
# - CharDevice: Must exist character device
# - BlockDevice: Must exist block device
```

**Real-World Example: DaemonSet Logging Agent**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: logging-agent
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: logging-agent
  template:
    metadata:
      labels:
        app: logging-agent
    spec:
      containers:
      - name: fluentd
        image: fluentd:latest
        volumeMounts:
        # Read Node logs
        - name: varlog
          mountPath: /var/log
          readOnly: true
        # Read container logs
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      
      volumes:
      # Access Node's /var/log
      - name: varlog
        hostPath:
          path: /var/log
          type: Directory
      
      # Access container logs
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
          type: Directory
```

**Hands-On:**

```bash
# Create Pod với hostPath
cat > hostpath-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpath
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls -la /host-logs && tail -f /dev/null"]
    volumeMounts:
    - name: host-logs
      mountPath: /host-logs
      readOnly: true
  
  volumes:
  - name: host-logs
    hostPath:
      path: /var/log
      type: Directory
EOF

kubectl apply -f hostpath-pod.yaml

# Exec vào Pod, check Node's /var/log
kubectl exec -it test-hostpath -- ls -la /host-logs

# Pod thấy được Node's logs!
kubectl exec -it test-hostpath -- tail -5 /host-logs/syslog

# Cleanup
kubectl delete pod test-hostpath
```

---

### 3. ConfigMap & Secret Volumes

**Mount ConfigMap as files:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:  # Optional: select specific keys
      - key: database.conf
        path: db.conf
      - key: cache.conf
        path: cache.conf
```

**Result:**

```bash
# Files created trong container:
/etc/config/db.conf
/etc/config/cache.conf
```

**Mount Secret as files:**

```yaml
volumes:
- name: secret-volume
  secret:
    secretName: app-secrets
    defaultMode: 0400  # Read-only by owner
```

---

### 4. Cloud Provider Volumes

#### AWS EBS (Elastic Block Store)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-ebs
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data-volume
      mountPath: /data
  
  volumes:
  - name: data-volume
    awsElasticBlockStore:
      volumeID: vol-0123456789abcdef0  # Pre-created EBS volume
      fsType: ext4
```

**Limitations:**
- Volume phải exist trước
- EBS volume và Pod phải cùng AZ (Availability Zone)
- ReadWriteOnce (single Pod only)

#### GCE Persistent Disk

```yaml
volumes:
- name: data-volume
  gcePersistentDisk:
    pdName: my-disk  # Pre-created disk
    fsType: ext4
```

#### Azure Disk

```yaml
volumes:
- name: data-volume
  azureDisk:
    diskName: my-disk
    diskURI: /subscriptions/.../my-disk
```

**⚠️ Problem với Cloud Volumes:**
- Manual provisioning required
- Hard to manage at scale
- Provider-specific syntax

**→ Solution: Use PersistentVolume & PersistentVolumeClaim (Phần 7.2)**

---

### 5. NFS (Network File System)

```yaml
volumes:
- name: nfs-volume
  nfs:
    server: nfs-server.example.com
    path: /exports/data
    readOnly: false
```

**Use Cases:**
- Shared storage across multiple Pods
- ReadWriteMany scenarios
- Legacy applications requiring NFS

**Example: Shared WordPress uploads**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: wordpress
spec:
  containers:
  - name: wordpress
    image: wordpress:latest
    volumeMounts:
    - name: wordpress-data
      mountPath: /var/www/html/wp-content/uploads
  
  volumes:
  - name: wordpress-data
    nfs:
      server: nfs-server.default.svc.cluster.local
      path: /exports/wordpress
```

Multiple WordPress Pods có thể mount cùng NFS → Shared uploads folder

---

## 🎮 Hands-On Lab: Multi-Container với Shared Volume

### Lab: Application + Log Shipper

**Objective:** Webapp writes logs → Log shipper reads và ships to central logging

```yaml
# lab-shared-logs.yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-logging
  labels:
    app: webapp
spec:
  containers:
  # Application container
  - name: webapp
    image: nginx:alpine
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  
  # Log shipper sidecar
  - name: log-shipper
    image: busybox
    command: 
    - sh
    - -c
    - |
      echo "Log Shipper Started"
      while true; do
        if [ -f /logs/access.log ]; then
          echo "=== Last 5 access logs ==="
          tail -5 /logs/access.log
          echo "Shipping to Elasticsearch..."
          sleep 10
        else
          echo "Waiting for access.log..."
          sleep 2
        fi
      done
    volumeMounts:
    - name: logs
      mountPath: /logs
      readOnly: true
  
  volumes:
  - name: logs
    emptyDir: {}
```

**Run Lab:**

```bash
# 1. Deploy Pod
kubectl apply -f lab-shared-logs.yaml

# 2. Check Pods running
kubectl get pods -w

# 3. Generate traffic (trong terminal khác)
kubectl port-forward webapp-with-logging 8080:80 &

# Generate requests
for i in {1..10}; do
  curl http://localhost:8080
  sleep 1
done

# 4. Check webapp logs (nginx access logs)
kubectl logs webapp-with-logging -c webapp

# 5. Check log-shipper logs (reads same logs!)
kubectl logs webapp-with-logging -c log-shipper -f

# Output sẽ show:
# === Last 5 access logs ===
# 127.0.0.1 - - [date] "GET / HTTP/1.1" 200 ...
# Shipping to Elasticsearch...

# 6. Exec vào webapp container
kubectl exec -it webapp-with-logging -c webapp -- ls -la /var/log/nginx

# 7. Exec vào log-shipper container
kubectl exec -it webapp-with-logging -c log-shipper -- ls -la /logs

# Cùng files! (shared volume)

# 8. Cleanup
kubectl delete pod webapp-with-logging
```

---

## 🔍 Troubleshooting Volumes

### Issue 1: Volume Mount Failed

```bash
$ kubectl get pods
NAME    READY   STATUS                 RESTARTS   AGE
mypod   0/1     ContainerCreating      0          2m

$ kubectl describe pod mypod

# Events:
# Warning  FailedMount  kubelet  MountVolume.SetUp failed: hostPath type check failed

# Root Cause: hostPath path doesn't exist

# Fix:
volumes:
- name: my-volume
  hostPath:
    path: /path/to/directory
    type: DirectoryOrCreate  # Auto-create if not exists
```

### Issue 2: Permission Denied

```bash
$ kubectl logs mypod
# Error: Permission denied writing to /data

# Check volume mount:
$ kubectl exec mypod -- ls -ld /data
# drwxr-xr-x  2 root root

# App runs as non-root user, can't write!

# Fix: Set securityContext
spec:
  securityContext:
    fsGroup: 1000  # Files owned by group 1000
  containers:
  - name: app
    securityContext:
      runAsUser: 1000
```

### Issue 3: Volume Not Shared Between Containers

```bash
# Container A writes, Container B can't see files

# Check: Are both mounting SAME volume?
spec:
  containers:
  - name: container-a
    volumeMounts:
    - name: shared-data  # ← Same name
      mountPath: /data-a
  
  - name: container-b
    volumeMounts:
    - name: shared-data  # ← Same name
      mountPath: /data-b
  
  volumes:
  - name: shared-data  # ← Defined once
    emptyDir: {}
```

---

## 💡 Best Practices

### ✅ DO

**1. Use emptyDir for temporary data**
```yaml
# Cache, scratch space
volumes:
- name: cache
  emptyDir:
    sizeLimit: 1Gi  # Limit size
```

**2. Use hostPath only for DaemonSets**
```yaml
# Accessing Node filesystem
kind: DaemonSet  # Not Deployment!
spec:
  volumes:
  - name: node-logs
    hostPath:
      path: /var/log
      type: Directory
```

**3. Set size limits**
```yaml
volumes:
- name: temp
  emptyDir:
    sizeLimit: 500Mi  # Prevent disk fill-up
```

**4. Use readOnly when possible**
```yaml
volumeMounts:
- name: config
  mountPath: /etc/config
  readOnly: true  # Prevent accidental writes
```

### ❌ DON'T

**1. Don't use hostPath for stateful data**
```yaml
# ❌ BAD: Database data on hostPath
volumes:
- name: db-data
  hostPath:
    path: /data/postgres
# Pod moves to different Node → Data lost!

# ✅ GOOD: Use PersistentVolume (Phần 7.2)
```

**2. Don't hardcode cloud volume IDs**
```yaml
# ❌ BAD:
volumes:
- name: data
  awsElasticBlockStore:
    volumeID: vol-0123456789  # Hardcoded!

# ✅ GOOD: Use StorageClass + PVC (Phần 7.3)
```

**3. Don't mount sensitive paths từ Node**
```yaml
# ❌ DANGEROUS:
volumes:
- name: root
  hostPath:
    path: /  # Host's root filesystem!
# Security risk! Pod có full Node access
```

---

## 🎯 Key Takeaways

**1. Volume Types Summary:**

| Type | Lifetime | Sharing | Use Case |
|------|----------|---------|----------|
| **emptyDir** | Pod | Within Pod | Cache, temp data, sidecar |
| **hostPath** | Node | Node-specific | DaemonSet, Node access |
| **ConfigMap** | Cluster | Read-only config | App configuration |
| **Secret** | Cluster | Read-only secrets | Credentials |
| **Cloud Volumes** | Persistent | Depends | Legacy (use PVC instead) |
| **NFS** | Persistent | Multiple Pods | Shared storage |

**2. emptyDir = Temporary**
- Created when Pod starts
- Deleted when Pod terminates
- Shared within Pod

**3. hostPath = Node-Specific**
- Access Node filesystem
- Use carefully (security risk)
- DaemonSets only

**4. For Production Storage:**
- Don't use Volumes directly
- Use PersistentVolume + PersistentVolumeClaim (Phần 7.2)
- Use StorageClass for dynamic provisioning (Phần 7.3)

**5. Sidecar Pattern:**
- Main app + Helper container
- Share data via emptyDir
- Common for logging, monitoring, proxies

---

## 📚 Commands Cheat Sheet

```bash
# Create Pod với volume
kubectl apply -f pod-with-volume.yaml

# Check volume mounts
kubectl describe pod <pod-name>
# Look for: Mounts, Volumes sections

# Exec và check mount
kubectl exec <pod-name> -- df -h
kubectl exec <pod-name> -- ls -la /mountpath

# Check volume usage
kubectl exec <pod-name> -- du -sh /mountpath

# Multi-container: specify container
kubectl exec <pod-name> -c <container-name> -- ls /data

# Logs từ specific container
kubectl logs <pod-name> -c <container-name>

# Port forward để test
kubectl port-forward <pod-name> 8080:80
```

---

## 🚀 Tiếp Theo

Volumes học xong! Next: PersistentVolumes cho production storage!

**Next:** [7.2. PersistentVolume & PersistentVolumeClaim →](./02-persistent-volumes.md)

Ở phần tiếp theo, bạn sẽ học:
- Abstraction layer cho storage
- PV (admin) vs PVC (developer)
- Storage lifecycle management
- Production-ready persistent storage

---

[⬅️ Phần 7: Storage](./README.md) | [🏠 Mục Lục](../README.md) | [➡️ 7.2. Persistent Volumes](./02-persistent-volumes.md)
