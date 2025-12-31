# 4.5. Jobs & CronJobs - Batch Workloads

> Run-to-completion tasks và scheduled jobs

---

## 🎯 Mục Tiêu Học

Sau khi học xong phần này, bạn sẽ:
- ✅ Hiểu **Jobs và CronJobs** là gì
- ✅ Biết **khi nào dùng** Jobs vs long-running services
- ✅ Tạo **one-time tasks** với Jobs
- ✅ Schedule **recurring tasks** với CronJobs
- ✅ Handle **job failures và retries**
- ✅ **Parallel jobs** và batch processing

---

## 📦 Job Là Gì?

### Định Nghĩa

**Job** = Controller creates Pods để chạy task đến completion, sau đó terminates.

### Giải Thích Bằng Ví Dụ

**Long-running Services vs Jobs:**

```
🏢 CÔNG TY

DEPLOYMENT = Nhân viên full-time:
├── Làm việc 24/7 (always running)
├── Web server, API server
├── Restart if crashed
└── Never "completes" - always running

Use cases: Web apps, APIs, databases

JOB = Nhân viên project-based:
├── Làm task cụ thể
├── Hoàn thành → Done, về nhà
├── Database migration, backup
├── Report generation
└── "Completes" when done

Use cases: Batch processing, one-time tasks

CRONJOB = Scheduled tasks:
├── Mỗi ngày 2AM chạy backup
├── Mỗi tuần gửi báo cáo
├── Cron schedule: "0 2 * * *"
└── Tự động tạo Job theo schedule

Use cases: Backups, cleanup, reports
```

---

## 🤔 TẠI SAO Cần Jobs?

### Problem với Deployment cho Tasks

```yaml
# Using Deployment cho one-time task
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-migration
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: migrate
        image: migrations:v1
        command: ["./migrate"]

# Problems:
❌ Task completes (exit code 0) → Pod restarts!
   (Deployment thinks it crashed)
   → Infinite restarts
   → Database migrated multiple times! (BAD!)

❌ No completion tracking
   → Can't tell if task succeeded

❌ Can't set retry limit
   → Keeps retrying forever if fails

❌ restartPolicy: Never doesn't work với Deployment
   → Deployment requires restartPolicy: Always
```

### Solution: Job

```yaml
# Using Job
apiVersion: batch/v1
kind: Job
metadata:
  name: database-migration
spec:
  template:
    spec:
      restartPolicy: Never  # Don't restart on completion
      containers:
      - name: migrate
        image: migrations:v1
        command: ["./migrate"]

# Benefits:
✓ Task completes (exit 0) → Job done! No restart
✓ Completion tracked (kubectl get jobs shows "1/1")
✓ Retry limit (backoffLimit: 3)
✓ Can run multiple Pods (parallelism)
✓ History preserved (completed Pods kept)
```

---

## 📝 Job YAML

### Basic Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
spec:
  # Template cho Pod
  template:
    spec:
      # MUST be Never or OnFailure (not Always!)
      restartPolicy: Never
      containers:
      - name: backup
        image: backup-tool:v1
        command:
        - /bin/sh
        - -c
        - |
          echo "Starting backup..."
          pg_dump dbname > /backup/dump.sql
          echo "Backup complete!"
```

### Job với Parameters

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
spec:
  # Parallelism: Số Pods chạy đồng thời
  parallelism: 3
  
  # Completions: Tổng số lần cần complete
  completions: 10
  
  # Backoff limit: Max retries khi fail
  backoffLimit: 4
  
  # Timeout: Max thời gian Job chạy
  activeDeadlineSeconds: 600  # 10 minutes
  
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: processor
        image: data-processor:v1
        command: ["./process"]
```

---

## 🎮 Hands-On: Working với Jobs

### Create Simple Job

```yaml
# simple-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: hello
        image: busybox
        command: ['sh', '-c', 'echo "Hello from Job!"; sleep 10; echo "Done!"']
```

```bash
# Create Job
kubectl apply -f simple-job.yaml

# Watch Job
kubectl get jobs -w

# Output:
# NAME        COMPLETIONS   DURATION   AGE
# hello-job   0/1           2s         2s
# hello-job   1/1           12s        12s  ← Completed!

# Check Pods
kubectl get pods -l job-name=hello-job

# Output:
# NAME              READY   STATUS      RESTARTS   AGE
# hello-job-abc12   0/1     Completed   0          30s

# Status = Completed (not Running!)

# Check logs
kubectl logs hello-job-abc12

# Output:
# Hello from Job!
# Done!
```

---

### Job với Failure và Retry

```yaml
# failing-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: failing-job
spec:
  backoffLimit: 3  # Retry max 3 times
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: failer
        image: busybox
        command: ['sh', '-c', 'echo "Attempting..."; exit 1']  # Always fails!
```

```bash
# Create Job
kubectl apply -f failing-job.yaml

# Watch Pods
kubectl get pods -w -l job-name=failing-job

# Output:
# NAME                 READY   STATUS   RESTARTS   AGE
# failing-job-abc12    0/1     Error    0          5s
# failing-job-def34    0/1     Error    0          10s
# failing-job-ghi56    0/1     Error    0          20s
# failing-job-jkl78    0/1     Error    0          40s

# 4 Pods created (1 initial + 3 retries), all failed

# Check Job status
kubectl get jobs

# Output:
# NAME           COMPLETIONS   DURATION   AGE
# failing-job    0/1           80s        80s

# Job never completes (all retries failed)

# Describe Job
kubectl describe job failing-job

# Events:
# Warning  BackoffLimitExceeded  Job has reached the specified backoff limit
```

---

### Parallel Jobs

**Non-Parallel:** One Pod runs to completion

**Parallel với fixed completions:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 10      # Need 10 successful completions
  parallelism: 3       # Run 3 Pods at a time
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox
        command: ['sh', '-c', 'echo "Processing..."; sleep 5; echo "Done!"']
```

```bash
# Create Job
kubectl apply -f parallel-job.yaml

# Watch Pods (3 at a time!)
kubectl get pods -w -l job-name=parallel-job

# Output:
# NAME                 READY   STATUS    AGE
# parallel-job-abc12   1/1     Running   2s   ← Batch 1 (3 Pods)
# parallel-job-def34   1/1     Running   2s
# parallel-job-ghi56   1/1     Running   2s
# parallel-job-abc12   0/1     Completed 7s
# parallel-job-jkl78   0/1     Running   1s   ← New Pod started
# ... continues until 10 completions

# Check Job progress
kubectl get jobs parallel-job -w

# Output:
# NAME           COMPLETIONS   DURATION   AGE
# parallel-job   0/10          5s         5s
# parallel-job   3/10          10s        10s
# parallel-job   6/10          15s        15s
# parallel-job   9/10          20s        20s
# parallel-job   10/10         25s        25s  ← Done!
```

**Work Queue Pattern:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: work-queue
spec:
  completions: null   # Unknown number
  parallelism: 5      # 5 workers
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: work-queue-consumer:v1
        # Workers pull tasks from queue
        # Exit when queue empty
```

---

### Delete Jobs

```bash
# Delete Job (keeps Pods by default)
kubectl delete job hello-job

# Job deleted, but Pods remain!
kubectl get pods -l job-name=hello-job
# hello-job-abc12   0/1     Completed   0    5m

# Delete Job AND Pods
kubectl delete job hello-job --cascade=foreground

# Or use TTL (Time To Live After Finished)
# (See TTLSecondsAfterFinished below)
```

---

## ⚙️ Job Configuration Options

### restartPolicy

```yaml
# Option 1: Never (recommended)
spec:
  template:
    spec:
      restartPolicy: Never

# Pod fails → Job creates new Pod
# Each attempt = New Pod

# Option 2: OnFailure
spec:
  template:
    spec:
      restartPolicy: OnFailure

# Pod fails → Restart container in SAME Pod
# All attempts in same Pod
```

**When to use what:**

```
restartPolicy: Never
✓ Clean retries (new Pod each time)
✓ Easy to debug (separate logs per attempt)
✓ Good for idempotent tasks

restartPolicy: OnFailure
✓ Faster retries (no Pod creation overhead)
✓ Less resource usage
✓ Good for transient failures
```

---

### TTL After Finished

**Automatic cleanup:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: cleanup-job
spec:
  # Delete Job and Pods 60s after completion
  ttlSecondsAfterFinished: 60
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: task
        image: busybox
        command: ['sh', '-c', 'echo "Done"']
```

```bash
# Job completes at t=0
# At t=60s → Job and Pods automatically deleted!

# Check after 60s
kubectl get jobs
# No resources found (auto-deleted!)
```

---

### activeDeadlineSeconds

**Timeout for Job:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: timeout-job
spec:
  activeDeadlineSeconds: 60  # Max 60s
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: sleeper
        image: busybox
        command: ['sh', '-c', 'sleep 120']  # Runs 120s (exceeds limit!)
```

```bash
# Job starts
# After 60s → Job killed (DeadlineExceeded)

kubectl describe job timeout-job
# Reason: DeadlineExceeded
# Message: Job was active longer than specified deadline
```

---

## 📅 CronJob

### Định Nghĩa

**CronJob** = Tạo Jobs theo schedule (cron format).

### Giải Thích

```
CronJob = Scheduled Tasks Manager

Every day at 2 AM: Run backup
Every Monday at 9 AM: Send weekly report
Every 15 minutes: Cleanup temp files

Uses standard cron syntax:
* * * * *
│ │ │ │ │
│ │ │ │ └─ Day of week (0-7, 0=Sunday)
│ │ │ └─── Month (1-12)
│ │ └───── Day of month (1-31)
│ └─────── Hour (0-23)
└───────── Minute (0-59)
```

---

### CronJob YAML

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-cronjob
spec:
  # Schedule (cron format)
  schedule: "0 2 * * *"  # Every day at 2 AM
  
  # Job template
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: backup
            image: backup-tool:v1
            command: ["./backup.sh"]
```

### Common Cron Schedules

```yaml
# Every minute
schedule: "* * * * *"

# Every 5 minutes
schedule: "*/5 * * * *"

# Every hour
schedule: "0 * * * *"

# Every day at 3 AM
schedule: "0 3 * * *"

# Every Monday at 9 AM
schedule: "0 9 * * 1"

# First day of month at midnight
schedule: "0 0 1 * *"

# Every weekday (Mon-Fri) at 8 AM
schedule: "0 8 * * 1-5"
```

---

### CronJob Hands-On

```yaml
# hello-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *"  # Every minute
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: hello
            image: busybox
            command:
            - /bin/sh
            - -c
            - date; echo "Hello from CronJob!"
```

```bash
# Create CronJob
kubectl apply -f hello-cronjob.yaml

# Check CronJob
kubectl get cronjobs
# or
kubectl get cj

# Output:
# NAME         SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# hello-cron   */1 * * * *   False     0        <none>          10s

# Wait a minute, then check Jobs
kubectl get jobs

# Output (new Job every minute!):
# NAME                  COMPLETIONS   DURATION   AGE
# hello-cron-27894123   1/1           5s         55s
# hello-cron-27894124   1/1           4s         4s  ← New Job!

# Check Pods
kubectl get pods

# Output:
# NAME                        READY   STATUS      AGE
# hello-cron-27894123-abc12   0/1     Completed   1m
# hello-cron-27894124-def34   0/1     Completed   10s

# Logs
kubectl logs hello-cron-27894124-def34
# Output:
# Mon Jan  1 10:05:00 UTC 2024
# Hello from CronJob!
```

---

### CronJob Configuration

**concurrencyPolicy:**

```yaml
spec:
  # What if previous Job still running when next schedule?
  concurrencyPolicy: Allow  # Default (allow concurrent)
  # concurrencyPolicy: Forbid  # Skip if previous still running
  # concurrencyPolicy: Replace  # Kill previous, start new
```

**Example - Forbid:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: slow-job
spec:
  schedule: "*/1 * * * *"  # Every minute
  concurrencyPolicy: Forbid  # Don't allow concurrent
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: slow
            image: busybox
            command: ['sh', '-c', 'sleep 120']  # Runs 2 min
```

```
t=0:   Job 1 starts (runs 2 min)
t=1m:  Job 2 schedule → Skipped! (Job 1 still running)
t=2m:  Job 3 schedule → Skipped! (Job 1 finished, but too late)
t=3m:  Job 4 starts (no conflict)
```

**successfulJobsHistoryLimit:**

```yaml
spec:
  successfulJobsHistoryLimit: 3   # Keep last 3 successful Jobs
  failedJobsHistoryLimit: 1       # Keep last 1 failed Job
```

---

### Suspend CronJob

```bash
# Temporarily stop CronJob from creating Jobs
kubectl patch cronjob hello-cron -p '{"spec":{"suspend":true}}'

# Check
kubectl get cronjob hello-cron

# Output:
# NAME         SCHEDULE      SUSPEND   ACTIVE
# hello-cron   */1 * * * *   True      0  ← Suspended!

# Resume
kubectl patch cronjob hello-cron -p '{"spec":{"suspend":false}}'
```

---

## 🐛 Troubleshooting Jobs & CronJobs

### Issue 1: Job Stuck (Not Completing)

```bash
$ kubectl get jobs
NAME      COMPLETIONS   DURATION   AGE
my-job    0/1           10m        10m

# Job running too long, not completing

# Check Pod
$ kubectl get pods -l job-name=my-job
NAME           READY   STATUS    AGE
my-job-abc12   1/1     Running   10m

# Pod still running, check logs
$ kubectl logs my-job-abc12
# (Check why task not completing)

# If stuck, set activeDeadlineSeconds next time
```

---

### Issue 2: CronJob Not Creating Jobs

```bash
$ kubectl get cronjob
NAME         SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE
my-cron      */5 * * * *   False     0        <none>

# LAST SCHEDULE = <none> → No Jobs created!

# Describe CronJob
$ kubectl describe cronjob my-cron

# Events might show:
# Warning  FailedNeedsStart  ... forbidden: ...

# Possible causes:
1. Invalid cron syntax
2. ServiceAccount missing permissions
3. Resource quotas exceeded
4. CronJob suspended

# Debug:
kubectl get events | grep my-cron
```

---

### Issue 3: Too Many Failed Pods

```bash
$ kubectl get pods | grep Error
my-job-abc12   0/1   Error   0   5m
my-job-def34   0/1   Error   0   5m
my-job-ghi56   0/1   Error   0   5m
... 20 more Error Pods ...

# Many failed Pods accumulating

# Solution: Set TTL
spec:
  ttlSecondsAfterFinished: 300  # Delete 5 min after finish

# Or manual cleanup
kubectl delete pods --field-selector status.phase=Failed
```

---

## 🎓 Kiểm Tra Hiểu Biết

**1. Khi nào dùng Job vs Deployment?**
<details>
<summary>Xem đáp án</summary>

**Job:**
- One-time tasks
- Task có "completion" (done khi exit 0)
- Database migrations, backups, batch processing
- restartPolicy: Never/OnFailure

**Deployment:**
- Long-running services
- Always running (no "completion")
- Web servers, APIs, databases
- restartPolicy: Always

**Rule:** Task có "done" state? → Job. Always running? → Deployment.
</details>

**2. restartPolicy: Never vs OnFailure cho Jobs?**
<details>
<summary>Xem đáp án</summary>

**Never:**
- Fail → Create new Pod
- Each attempt = New Pod
- Pros: Clean retries, separate logs
- Cons: More resource usage

**OnFailure:**
- Fail → Restart container in same Pod
- All attempts in same Pod
- Pros: Faster, less resources
- Cons: Logs mixed, harder to debug

**Recommended:** Never (easier to troubleshoot)
</details>

**3. CronJob schedule `*/10 * * * *` nghĩa là gì?**
<details>
<summary>Xem đáp án</summary>

**Every 10 minutes**

```
*/10 * * * *
 │   │ │ │ │
 │   │ │ │ └─ Day of week: * (every day)
 │   │ │ └─── Month: * (every month)
 │   │ └───── Day: * (every day of month)
 │   └─────── Hour: * (every hour)
 └─────────── Minute: */10 (every 10 minutes)

Jobs run at: 00, 10, 20, 30, 40, 50 minutes past every hour
```
</details>

---

## 💪 Bài Tập Thực Hành

### Bài 1: Database Backup Job

```yaml
# backup-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-backup
spec:
  backoffLimit: 2
  activeDeadlineSeconds: 300  # 5 min timeout
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: backup
        image: postgres:14
        command:
        - /bin/sh
        - -c
        - |
          echo "Starting backup at $(date)"
          sleep 10  # Simulate backup
          echo "Backup complete at $(date)"
        env:
        - name: PGHOST
          value: "postgres-service"
```

```bash
# Run backup Job
kubectl apply -f backup-job.yaml

# Watch
kubectl get jobs -w

# Check logs
kubectl logs -l job-name=db-backup

# Cleanup
kubectl delete job db-backup
```

---

### Bài 2: Scheduled Cleanup CronJob

```yaml
# cleanup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
spec:
  schedule: "0 2 * * *"  # Every day at 2 AM
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: cleanup
            image: busybox
            command:
            - /bin/sh
            - -c
            - |
              echo "Cleanup started at $(date)"
              echo "Removing old logs..."
              echo "Cleanup complete at $(date)"
```

```bash
# Create CronJob
kubectl apply -f cleanup-cronjob.yaml

# For testing, change schedule to run every minute
kubectl patch cronjob cleanup -p '{"spec":{"schedule":"*/1 * * * *"}}'

# Watch Jobs being created
kubectl get jobs -w

# After a few minutes, check history
kubectl get jobs

# Should see max 3 successful Jobs (history limit)

# Cleanup
kubectl delete cronjob cleanup
```

---

## 🎯 Key Takeaways

1. **Job = Run to Completion**
   - One-time tasks
   - Completes then stops
   - restartPolicy: Never/OnFailure

2. **vs Deployment**
   - Deployment: Always running
   - Job: Has completion state
   - Choose based on workload type

3. **Parallel Jobs**
   - completions: Total needed
   - parallelism: Concurrent Pods
   - Work queue pattern

4. **CronJob = Scheduled Jobs**
   - Cron syntax for schedule
   - Creates Jobs automatically
   - concurrencyPolicy controls overlaps

5. **Best Practices**
   - Set backoffLimit (retry limit)
   - Set activeDeadlineSeconds (timeout)
   - Use ttlSecondsAfterFinished (cleanup)
   - Set history limits for CronJobs

---

## 🚀 Hoàn Thành Phần 4!

Congratulations! Bạn đã hoàn thành Phần 4 - Workloads!

**Next:** [Phần 5: Networking →](../05-networking/README.md)

Learn về Services, Ingress, và Pod communication!

---

[⬅️ 4.4. DaemonSet](./04-daemonset.md) | [🏠 Mục Lục](../README.md) | [📂 Phần 4: Workloads](./README.md) | [➡️ Phần 5: Networking](../05-networking/README.md)
