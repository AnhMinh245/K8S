# 4.5. Job & CronJob - Batch Processing

> Job và CronJob cho các tác vụ chạy một lần hoặc theo lịch

---

## 🎯 Job Là Gì?

**Job** = Controller chạy Pod đến khi **hoàn thành task** rồi dừng

```
Normal Deployment:
  Pod fails → Restart forever
  Goal: Keep running

Job:
  Pod completes (exit code 0) → Job done ✅
  Pod fails → Retry (up to limit)
  Goal: Complete task successfully
```

---

## 🏢 Ví Dụ Thực Tế

**Deployment = Nhân viên full-time**
```
Làm việc liên tục:
  • 8 giờ/ngày
  • 5 ngày/tuần
  • Nghỉ → Tuyển người thay
  → Long-running
```

**Job = Nhân viên hợp đồng ngắn hạn**
```
Nhiệm vụ cụ thể:
  • Import 1000 records vào database
  • Xong → Về
  • Thất bại → Thử lại
  → Run-to-completion
```

---

## 📝 Job YAML

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-import
spec:
  completions: 1          # Number of successful completions needed
  parallelism: 1          # Max Pods running in parallel
  backoffLimit: 3         # Max retries on failure
  template:
    spec:
      containers:
      - name: importer
        image: myapp/data-importer:v1
        command:
        - /bin/sh
        - -c
        - |
          echo "Importing data..."
          python import_data.py
          echo "Import complete!"
      restartPolicy: Never  # Never or OnFailure (not Always!)
```

---

## 🔄 Job Lifecycle

### Successful Job

```
1. Job created
   kubectl apply -f job.yaml

2. Job Controller creates Pod

3. Pod runs → Task completes → Exit 0

4. Job status: Completed (1/1)

5. Pod status: Completed (not restarted)

Result: Job done ✅
```

### Failed Job (with retries)

```
1. Job created (backoffLimit: 3)

2. Pod runs → Task fails → Exit 1

3. Pod status: Error

4. Job Controller: Retry (1/3)
   Creates new Pod

5. New Pod fails again → Retry (2/3)

6. Fails third time → Retry (3/3)

7. Job status: Failed (exceeded backoffLimit)

Result: Job failed after 3 attempts ❌
```

---

## ⚙️ Job Configurations

### 1. Completions

```yaml
spec:
  completions: 5  # Need 5 successful completions
  parallelism: 2  # Run 2 Pods at a time
```

**Execution:**
```
Start: Pod 1, Pod 2 (parallelism=2)
Pod 1 completes → Start Pod 3
Pod 2 completes → Start Pod 4
Pod 3 completes → Start Pod 5
Pod 4 completes → Done (4/5)
Pod 5 completes → Done (5/5) ✅
```

---

### 2. Parallelism

```yaml
# Sequential
spec:
  completions: 10
  parallelism: 1  # One at a time
  
# Parallel
spec:
  completions: 10
  parallelism: 5  # 5 Pods at once (faster)
```

**Use case:**
```
Process 1000 files:
  Sequential (parallelism=1): 1000 minutes
  Parallel (parallelism=10): ~100 minutes
```

---

### 3. Backoff Limit

```yaml
spec:
  backoffLimit: 6  # Retry up to 6 times (default: 6)
```

**Backoff delay:**
```
Retry 1: Immediate
Retry 2: 10 seconds
Retry 3: 20 seconds
Retry 4: 40 seconds
...
Max: 6 minutes
```

---

### 4. Active Deadline

```yaml
spec:
  activeDeadlineSeconds: 3600  # Job must complete in 1 hour
```

**Behavior:**
```
Job starts → 1 hour timer
After 1 hour → Job terminated (even if not done)
Status: Failed (DeadlineExceeded)

Use case: Prevent infinite hanging
```

---

### 5. Restart Policy

```yaml
spec:
  template:
    spec:
      restartPolicy: Never  # or OnFailure

# Never: Failed Pod → Job creates new Pod
# OnFailure: Failed container → Restart in same Pod
```

---

## 🎯 Job Use Cases

### 1. Data Import/Export

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-import
spec:
  template:
    spec:
      containers:
      - name: importer
        image: postgres:13
        command:
        - psql
        - -h
        - database-host
        - -U
        - user
        - -f
        - /scripts/import.sql
      restartPolicy: Never
```

---

### 2. Database Migration

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:v2.0
        command:
        - python
        - manage.py
        - migrate
      restartPolicy: OnFailure
```

---

### 3. Batch Processing

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: image-processing
spec:
  completions: 100    # Process 100 images
  parallelism: 10     # 10 workers
  template:
    spec:
      containers:
      - name: processor
        image: myapp/image-processor
        command:
        - /process-batch.sh
      restartPolicy: Never
```

---

### 4. Backup

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup
spec:
  template:
    spec:
      containers:
      - name: backup
        image: postgres:13
        command:
        - pg_dump
        - -h
        - database
        - -U
        - user
        - -f
        - /backup/db-backup.sql
        volumeMounts:
        - name: backup
          mountPath: /backup
      volumes:
      - name: backup
        persistentVolumeClaim:
          claimName: backup-pvc
      restartPolicy: Never
```

---

## 📅 CronJob

**CronJob** = Job chạy theo lịch định kỳ (như crontab Linux)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"  # 2 AM every day
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:v1
            command:
            - /backup.sh
          restartPolicy: OnFailure
```

---

## ⏰ Cron Schedule Syntax

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday=0)
│ │ │ │ │
* * * * *
```

### Examples

```yaml
# Every hour
schedule: "0 * * * *"

# Every day at 2 AM
schedule: "0 2 * * *"

# Every Monday at 9 AM
schedule: "0 9 * * 1"

# Every 15 minutes
schedule: "*/15 * * * *"

# First day of month at midnight
schedule: "0 0 1 * *"

# Every weekday (Mon-Fri) at 6 PM
schedule: "0 18 * * 1-5"
```

---

## ⚙️ CronJob Configurations

### 1. Concurrency Policy

```yaml
spec:
  concurrencyPolicy: Forbid  # Default: Allow

# Allow: Multiple Jobs can run concurrently
# Forbid: Skip new Job if previous still running
# Replace: Cancel old Job, start new one
```

**Example:**
```
CronJob: Every minute
Job takes 2 minutes to complete

Policy: Allow
  0:00 → Job 1 starts
  1:00 → Job 2 starts (Job 1 still running)
  2:00 → Job 3 starts (Job 1 done, Job 2 running)

Policy: Forbid
  0:00 → Job 1 starts
  1:00 → Skip (Job 1 still running)
  2:00 → Job 2 starts (Job 1 done)

Policy: Replace
  0:00 → Job 1 starts
  1:00 → Job 1 terminated, Job 2 starts
```

---

### 2. Starting Deadline

```yaml
spec:
  startingDeadlineSeconds: 300  # 5 minutes
```

**Meaning:**
```
Scheduled: 2:00 AM
K8s controller unavailable until 2:10 AM

Without deadline:
  → Job still starts at 2:10 AM (10 min late)

With deadline (300s):
  → Job skipped (missed by > 5 minutes)
```

---

### 3. History Limits

```yaml
spec:
  successfulJobsHistoryLimit: 3  # Keep last 3 successful Jobs
  failedJobsHistoryLimit: 1      # Keep last 1 failed Job
```

**Why:**
- Old Job objects accumulate
- Cleanup to save etcd space

---

## 🎯 CronJob Use Cases

### 1. Daily Backup

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-db-backup
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:13
            command: ["/backup.sh"]
          restartPolicy: OnFailure
```

---

### 2. Periodic Cleanup

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: log-cleanup
spec:
  schedule: "0 0 * * 0"  # Midnight every Sunday
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: busybox
            command:
            - /bin/sh
            - -c
            - |
              find /logs -mtime +7 -delete
          restartPolicy: Never
```

---

### 3. Report Generation

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekly-report
spec:
  schedule: "0 9 * * 1"  # Monday 9 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report
            image: myapp/reporter:v1
            command: ["python", "generate_report.py"]
          restartPolicy: OnFailure
```

---

### 4. Data Sync

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: data-sync
spec:
  schedule: "*/30 * * * *"  # Every 30 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: sync
            image: rsync:latest
            command: ["/sync-data.sh"]
          restartPolicy: OnFailure
```

---

## 🔧 Operations

### Job

```bash
# Create
kubectl apply -f job.yaml

# List
kubectl get jobs

# Check status
kubectl describe job data-import

# Logs
kubectl logs job/data-import

# Delete (Pods deleted too)
kubectl delete job data-import
```

### CronJob

```bash
# Create
kubectl apply -f cronjob.yaml

# List
kubectl get cronjobs

# Describe
kubectl describe cronjob daily-backup

# Create Job manually from CronJob (testing)
kubectl create job --from=cronjob/daily-backup test-backup

# Suspend (pause scheduling)
kubectl patch cronjob daily-backup -p '{"spec":{"suspend":true}}'

# Resume
kubectl patch cronjob daily-backup -p '{"spec":{"suspend":false}}'
```

---

## 💡 Best Practices

### ✅ DO

1. **Idempotent tasks**
```
Task can run multiple times safely
  → Same result every time
  → Important for retry logic
```

2. **Set activeDeadlineSeconds**
```yaml
activeDeadlineSeconds: 3600  # Timeout after 1 hour
```

3. **Set resource limits**
```yaml
resources:
  limits:
    cpu: 1
    memory: 1Gi
```

4. **Use OnFailure for expensive startups**
```yaml
restartPolicy: OnFailure  # Restart container, not Pod
```

5. **Clean up completed Jobs**
```yaml
# CronJob
successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 1

# Or use TTL
ttlSecondsAfterFinished: 86400  # Delete after 24h
```

6. **Test with manual Job**
```bash
kubectl create job --from=cronjob/backup test-backup
```

---

### ❌ DON'T

1. **Long-running tasks in Job** → Use Deployment
2. **Always restart policy** → Won't work with Job
3. **No timeout** → Job can hang forever
4. **No resource limits** → Can starve cluster
5. **Concurrent CronJobs without handling** → Race conditions

---

## 🎓 Key Takeaways

1. **Job:** Run-to-completion tasks
2. **CronJob:** Scheduled tasks (like crontab)
3. **Completions:** Number of successful runs needed
4. **Parallelism:** Run multiple Pods concurrently
5. **BackoffLimit:** Retry failed Pods
6. **restartPolicy:** Never or OnFailure (not Always)
7. **Use cases:** Batch processing, backups, migrations, reports

---

## ❓ Câu Hỏi Tự Kiểm Tra

1. Job khác gì với Deployment?
2. completions và parallelism là gì?
3. restartPolicy có thể là gì trong Job?
4. CronJob schedule syntax?
5. concurrencyPolicy trong CronJob?
6. Khi nào dùng Job vs CronJob?

---

**Chúc mừng!** Bạn đã hoàn thành **Phần 4: Workloads** 🎉

👉 [**Phần 5: Networking**](../05-networking/README.md)

---

[⬅️ 4.4. DaemonSet](./04-daemonset.md) | [⬆️ Phần 4](./README.md) | [🏠 Mục Lục Chính](../README.md)

