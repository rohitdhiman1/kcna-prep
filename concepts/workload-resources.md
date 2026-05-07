# Workload Resources

**Beyond Deployments and ReplicaSets, Kubernetes offers specialized workload controllers: Jobs for one-off tasks, CronJobs for scheduled tasks, DaemonSets for per-node Pods, and StatefulSets for stateful applications with stable identities.**

---

## Table of Contents

- [Workload Types Overview](#workload-types-overview)
- [Jobs](#jobs)
- [CronJobs](#cronjobs)
- [DaemonSets](#daemonsets)
- [StatefulSets](#statefulsets)
- [Comparison Table](#comparison-table)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## Workload Types Overview

```
  Kubernetes Workload Controllers
  ================================

  Stateless, long-running:
    Deployment -------> ReplicaSet -------> Pods
    (rolling updates,   (maintains desired   (your containers)
     rollbacks)          replica count)

  One-off tasks:
    Job ----------------------------------> Pods
    (run to completion, retries, parallelism)

  Scheduled tasks:
    CronJob ----------> Job --------------> Pods
    (cron schedule)     (created per run)

  One per node:
    DaemonSet -----------------------------> Pods (one per node)
    (log collectors, monitoring agents)

  Stateful, long-running:
    StatefulSet ---------------------------> Pods (ordered, stable identity)
    (databases, distributed systems)        + PersistentVolumeClaims
```

---

## Jobs

A Job creates one or more Pods and ensures they run **to completion**. Unlike Deployments, Jobs are for finite tasks that should terminate.

### Use Cases

- Database migrations
- Batch processing (data import/export)
- Sending emails
- Running reports
- One-time initialization tasks

### Job YAML Spec

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-import
spec:
  completions: 1              # How many Pods must complete successfully
  parallelism: 1              # How many Pods can run at the same time
  backoffLimit: 4             # How many retries before marking Job as failed
  activeDeadlineSeconds: 300  # Max time (seconds) before Job is terminated
  ttlSecondsAfterFinished: 100 # Auto-delete Job 100s after it finishes
  template:
    spec:
      containers:
      - name: import
        image: data-tools:v1
        command: ["python", "import.py"]
      restartPolicy: OnFailure  # Must be OnFailure or Never (not Always)
```

### Completions and Parallelism

```
  completions: 5, parallelism: 2
  ================================

  Time --->

  t0:  [Pod-1 running]  [Pod-2 running]  [Pod-3 waiting]  [Pod-4 waiting]  [Pod-5 waiting]
  t1:  [Pod-1 done]     [Pod-2 running]  [Pod-3 running]  [Pod-4 waiting]  [Pod-5 waiting]
  t2:  [Pod-1 done]     [Pod-2 done]     [Pod-3 running]  [Pod-4 running]  [Pod-5 waiting]
  t3:  [Pod-1 done]     [Pod-2 done]     [Pod-3 done]     [Pod-4 running]  [Pod-5 running]
  t4:  [Pod-1 done]     [Pod-2 done]     [Pod-3 done]     [Pod-4 done]    [Pod-5 done]

  Job Complete! (5 of 5 completions succeeded)
```

| Parameter             | Default | Description                                     |
|-----------------------|---------|-------------------------------------------------|
| `completions`         | 1       | Number of Pods that must complete successfully  |
| `parallelism`         | 1       | Max Pods running concurrently                   |
| `backoffLimit`        | 6       | Number of retries before Job is marked Failed   |
| `activeDeadlineSeconds` | none  | Maximum runtime for the entire Job              |
| `ttlSecondsAfterFinished` | none | Auto-cleanup delay after Job completes       |

### Restart Policy for Jobs

- **OnFailure**: Container restarts in the same Pod if it fails
- **Never**: A new Pod is created if the container fails (failed Pods are kept for debugging)
- **Always** is NOT allowed for Jobs

---

## CronJobs

A CronJob creates **Jobs** on a repeating schedule, using standard cron syntax.

### CronJob YAML Spec

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"              # At 2:00 AM every day
  concurrencyPolicy: Forbid          # Don't start new if previous still running
  successfulJobsHistoryLimit: 3      # Keep last 3 successful Jobs
  failedJobsHistoryLimit: 1          # Keep last 1 failed Job
  startingDeadlineSeconds: 200       # If missed by 200s, skip this run
  suspend: false                     # Set to true to pause scheduling
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:v1
            command: ["/bin/sh", "-c", "pg_dump mydb > /backup/dump.sql"]
          restartPolicy: OnFailure
```

### Cron Schedule Syntax

```
  +-------------- minute (0-59)
  |  +----------- hour (0-23)
  |  |  +-------- day of month (1-31)
  |  |  |  +----- month (1-12)
  |  |  |  |  +-- day of week (0-6, 0=Sunday)
  |  |  |  |  |
  *  *  *  *  *

  Examples:
  "0 2 * * *"       Every day at 2:00 AM
  "*/15 * * * *"    Every 15 minutes
  "0 0 * * 0"       Every Sunday at midnight
  "0 9 1 * *"       First day of every month at 9:00 AM
  "30 8 * * 1-5"    Weekdays at 8:30 AM
```

### Concurrency Policies

| Policy     | Description                                                   |
|------------|---------------------------------------------------------------|
| `Allow`    | (Default) Multiple Jobs can run concurrently                  |
| `Forbid`   | Skip the new run if previous Job is still running            |
| `Replace`  | Cancel the running Job and start a new one                   |

### CronJob Creates Jobs

```
  CronJob: nightly-backup (schedule: "0 2 * * *")
       |
       | 2:00 AM Day 1
       +---> Job: nightly-backup-28345600
       |          +---> Pod (runs backup, completes)
       |
       | 2:00 AM Day 2
       +---> Job: nightly-backup-28345601
       |          +---> Pod (runs backup, completes)
       |
       | 2:00 AM Day 3
       +---> Job: nightly-backup-28345602
                   +---> Pod (runs backup, completes)
```

---

## DaemonSets

A DaemonSet ensures that **one copy of a Pod runs on every node** (or a subset of nodes) in the cluster. When a new node is added, the DaemonSet automatically schedules a Pod on it. When a node is removed, the Pod is garbage collected.

### Use Cases

- **Log collection** agents (Fluentd, Filebeat)
- **Monitoring** agents (Prometheus Node Exporter, Datadog)
- **Networking** plugins (Calico, Cilium, kube-proxy)
- **Storage** daemons (Ceph, GlusterFS)
- **Security** agents (Falco, Aqua)

### DaemonSet Diagram

```
  DaemonSet: log-collector
  ==========================

  +----------+     +----------+     +----------+
  |  Node 1   |     |  Node 2   |     |  Node 3   |
  |           |     |           |     |           |
  | +-------+ |     | +-------+ |     | +-------+ |
  | | Pod   | |     | | Pod   | |     | | Pod   | |
  | | fluentd| |     | | fluentd| |     | | fluentd| |
  | +-------+ |     | +-------+ |     | +-------+ |
  |           |     |           |     |           |
  | (other   |     | (other   |     | (other   |
  |  pods)    |     |  pods)    |     |  pods)    |
  +----------+     +----------+     +----------+

  New node added:
  +----------+
  |  Node 4   |  <-- DaemonSet automatically schedules
  | +-------+ |      a fluentd Pod here
  | | Pod   | |
  | | fluentd| |
  | +-------+ |
  +----------+
```

### DaemonSet YAML Spec

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  labels:
    app: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
      - name: fluentd
        image: fluentd:v1.16
        resources:
          requests:
            cpu: "100m"
            memory: "200Mi"
          limits:
            cpu: "200m"
            memory: "400Mi"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

### Running on Specific Nodes

Use `nodeSelector` or `nodeAffinity` to limit which nodes run the DaemonSet:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd              # Only run on nodes labeled disk=ssd
```

### DaemonSet vs Deployment

- DaemonSet: exactly **one Pod per node**, no replicas setting
- Deployment: **N replicas** distributed across nodes by scheduler

---

## StatefulSets

A StatefulSet manages stateful applications that need **stable network identities**, **ordered deployment and scaling**, and **persistent storage**.

### Use Cases

- Databases (MySQL, PostgreSQL, MongoDB)
- Distributed systems (Kafka, ZooKeeper, etcd)
- Caches (Redis Cluster, Elasticsearch)
- Any workload requiring stable hostnames or persistent volumes

### Key Properties

```
  StatefulSet Properties
  =======================

  1. STABLE NETWORK IDENTITY
     Pod names are predictable: <statefulset-name>-<ordinal>
     +----------+    +----------+    +----------+
     | mysql-0   |    | mysql-1   |    | mysql-2   |
     +----------+    +----------+    +----------+
     Headless Service DNS:
       mysql-0.mysql-headless.default.svc.cluster.local
       mysql-1.mysql-headless.default.svc.cluster.local
       mysql-2.mysql-headless.default.svc.cluster.local

  2. ORDERED DEPLOYMENT (by default)
     mysql-0 created and ready  --->  mysql-1 created and ready  --->  mysql-2 created
     (Pods created one at a time, in order 0, 1, 2)

  3. ORDERED DELETION (reverse)
     mysql-2 deleted  --->  mysql-1 deleted  --->  mysql-0 deleted
     (Pods deleted in reverse order: 2, 1, 0)

  4. STABLE PERSISTENT STORAGE
     Each Pod gets its own PersistentVolumeClaim (PVC):
     +----------+    +----------+    +----------+
     | mysql-0   |    | mysql-1   |    | mysql-2   |
     +-----+----+    +-----+----+    +-----+----+
           |              |              |
     +-----+----+    +-----+----+    +-----+----+
     | PVC:      |    | PVC:      |    | PVC:      |
     | data-     |    | data-     |    | data-     |
     | mysql-0   |    | mysql-1   |    | mysql-2   |
     +----------+    +----------+    +----------+

     If mysql-1 is deleted and recreated, it gets the SAME PVC back.
```

### StatefulSet YAML Spec

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless     # Required: headless Service name
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:            # Each Pod gets its own PVC
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### Headless Service (Required for StatefulSets)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None              # This makes it a headless Service
  selector:
    app: mysql
  ports:
  - port: 3306
```

A headless Service (clusterIP: None) provides DNS entries for each Pod directly, rather than load-balancing across them.

### Pod Management Policies

| Policy         | Behavior                                                      |
|----------------|---------------------------------------------------------------|
| `OrderedReady` | (Default) Pods are created/deleted one at a time, in order   |
| `Parallel`     | All Pods are created/deleted simultaneously                  |

---

## Comparison Table

| Feature                | Deployment       | Job              | CronJob          | DaemonSet        | StatefulSet      |
|------------------------|------------------|------------------|------------------|------------------|------------------|
| **Purpose**           | Stateless apps   | One-off tasks    | Scheduled tasks  | Per-node agents  | Stateful apps    |
| **Runs**              | Continuously     | To completion    | On schedule      | Continuously     | Continuously     |
| **Replicas**          | N (configurable) | completions      | N/A (creates Jobs)| One per node    | N (configurable) |
| **Pod identity**      | Random names     | Random names     | Random names     | Random names     | Stable ordinal   |
| **Ordering**          | No               | No               | No               | No               | Yes (default)    |
| **Persistent storage**| Shared (if any)  | No (ephemeral)   | No (ephemeral)   | Usually hostPath | Per-Pod PVC      |
| **Rolling updates**   | Yes              | N/A              | N/A              | Yes              | Yes              |
| **Rollback**          | Yes              | N/A              | N/A              | Yes              | No               |
| **Scaling**           | Manual/HPA       | parallelism      | N/A              | Tied to nodes    | Manual           |
| **restartPolicy**     | Always           | OnFailure/Never  | OnFailure/Never  | Always           | Always           |
| **Example workload**  | Web server       | DB migration     | Nightly backup   | Log collector    | Database         |

---

## What to Remember for the Exam

1. **Jobs** run Pods to completion. They are for finite, one-off tasks. The `restartPolicy` must be `OnFailure` or `Never` (not `Always`).

2. **completions** = how many Pods must finish successfully. **parallelism** = how many can run at once. **backoffLimit** = retry count before marking the Job as failed.

3. **CronJobs** create Jobs on a schedule using cron syntax. Know the five fields: minute, hour, day-of-month, month, day-of-week.

4. **CronJob concurrencyPolicy**: `Allow` (default, multiple concurrent), `Forbid` (skip if previous running), `Replace` (cancel previous, start new).

5. **DaemonSets** run exactly one Pod per node. Common use cases: log collectors (Fluentd), monitoring agents (Node Exporter), networking plugins (kube-proxy, Calico).

6. **DaemonSets** do not have a `replicas` field. The number of Pods equals the number of matching nodes.

7. **StatefulSets** provide three guarantees: stable network identity (predictable Pod names), ordered deployment/deletion, and stable persistent storage (per-Pod PVCs).

8. **StatefulSet Pod names** follow the pattern `<statefulset-name>-<ordinal>` (e.g., mysql-0, mysql-1, mysql-2). This is why they need a **headless Service** (clusterIP: None).

9. **StatefulSet PVCs are NOT deleted** when the StatefulSet or Pod is deleted. This preserves data. You must manually delete PVCs.

10. **Deployments are for stateless apps**, **StatefulSets are for stateful apps**. This is the most common distinction asked on the exam.

11. **DaemonSets automatically add Pods** to new nodes and remove them from deleted nodes.
