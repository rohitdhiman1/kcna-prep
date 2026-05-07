# Lab 10: Workload Resources — Jobs, CronJobs, and DaemonSets

## Objective

Create and observe Jobs, CronJobs, and DaemonSets. Understand their behavior, completion semantics, and lifecycle.

## Prerequisites

- A running multi-node Kubernetes cluster (from [Lab 01](01-cluster-setup.md)).
- `kubectl` installed and configured.
- Read the concept file: [Orchestration Fundamentals](../concepts/orchestration-fundamentals.md)

---

## Step 1: Create a Simple Job

A Job creates one or more pods and ensures they run to successful completion.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calc
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.38
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(200)"]
      restartPolicy: Never
  backoffLimit: 4
EOF
```

### Watch the job

```bash
kubectl get jobs --watch
```

Expected progression:

```
NAME      COMPLETIONS   DURATION   AGE
pi-calc   0/1           5s         5s
pi-calc   1/1           15s        15s
```

Press `Ctrl+C` to stop watching.

### View the completed pod

```bash
kubectl get pods -l job-name=pi-calc
```

Expected output:

```
NAME             READY   STATUS      RESTARTS   AGE
pi-calc-abc12    0/1     Completed   0          20s
```

The pod status is `Completed`, not `Running`.

### View the output

```bash
kubectl logs -l job-name=pi-calc
```

You should see pi calculated to 200 decimal places.

---

## Step 2: Job with Multiple Completions

Create a job that runs 5 tasks, 2 at a time:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 5
  parallelism: 2
  template:
    spec:
      containers:
      - name: worker
        image: busybox:1.36
        command: ['sh', '-c', 'echo "Processing task on $(hostname)"; sleep 5']
      restartPolicy: Never
EOF
```

### Watch the pods

```bash
kubectl get pods -l job-name=batch-job --watch
```

You will see 2 pods running at a time, with new pods created as previous ones complete, until all 5 completions are done.

Press `Ctrl+C` to stop watching.

### Check job status

```bash
kubectl get job batch-job
```

Expected output:

```
NAME        COMPLETIONS   DURATION   AGE
batch-job   5/5           30s        35s
```

### View logs from all pods

```bash
kubectl logs -l job-name=batch-job
```

---

## Step 3: Create a CronJob

A CronJob creates Jobs on a schedule, like a cron entry in Linux.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.36
            command: ['sh', '-c', 'echo "Hello from CronJob at $(date)"']
          restartPolicy: OnFailure
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
EOF
```

This CronJob runs every minute.

### Verify the CronJob

```bash
kubectl get cronjobs
```

Expected output:

```
NAME         SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello-cron   */1 * * * *   False     0        <none>          10s
```

### Wait for the first execution

Wait about 60 seconds, then check:

```bash
kubectl get cronjobs
kubectl get jobs
kubectl get pods
```

You should see a Job created by the CronJob, and a Completed pod.

### View the output

```bash
kubectl logs -l job-name=$(kubectl get jobs -o jsonpath='{.items[0].metadata.name}')
```

Expected output:

```
Hello from CronJob at Sat Mar 14 12:01:00 UTC 2026
```

### Wait for multiple executions

After 3 minutes, check jobs:

```bash
kubectl get jobs
```

You should see up to 3 completed jobs (due to `successfulJobsHistoryLimit: 3`). Older jobs are automatically deleted.

---

## Step 4: Create a DaemonSet

A DaemonSet ensures a copy of a pod runs on every node (or a subset of nodes).

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: monitor
        image: busybox:1.36
        command: ['sh', '-c', 'while true; do echo "Node: $(hostname) - $(date)"; sleep 30; done']
EOF
```

The toleration allows this DaemonSet to run on the control plane node too.

### Verify

```bash
kubectl get daemonset node-monitor
```

Expected output:

```
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
node-monitor   3         3         3       3             3           <none>          20s
```

`DESIRED=3` means one pod on each of the 3 nodes.

### Check which node each pod is on

```bash
kubectl get pods -l app=node-monitor -o wide
```

Expected output:

```
NAME                 READY   STATUS    RESTARTS   AGE   IP           NODE
node-monitor-abc12   1/1     Running   0          30s   10.244.0.5   kcna-lab-control-plane
node-monitor-def34   1/1     Running   0          30s   10.244.1.4   kcna-lab-worker
node-monitor-ghi56   1/1     Running   0          30s   10.244.2.5   kcna-lab-worker2
```

One pod per node.

### View logs from a specific pod

```bash
kubectl logs -l app=node-monitor --tail=3
```

---

## Step 5: DaemonSet Without Control Plane Toleration

Create another DaemonSet that only runs on worker nodes (no toleration for the control plane taint):

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: worker-agent
spec:
  selector:
    matchLabels:
      app: worker-agent
  template:
    metadata:
      labels:
        app: worker-agent
    spec:
      containers:
      - name: agent
        image: busybox:1.36
        command: ['sh', '-c', 'echo "Agent running on $(hostname)"; sleep 3600']
EOF
```

### Verify

```bash
kubectl get daemonset worker-agent
```

Expected output:

```
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
worker-agent   2         2         2       2             2           <none>          10s
```

`DESIRED=2` because the control plane node has a taint that this DaemonSet does not tolerate.

```bash
kubectl get pods -l app=worker-agent -o wide
```

Pods only run on the two worker nodes.

---

## Step 6: Inspect Workload Resources

### List all workload types

```bash
kubectl get jobs
kubectl get cronjobs
kubectl get daemonsets
```

### Describe a job for detailed info

```bash
kubectl describe job pi-calc
```

Look at:
- **Completions** — how many successful completions are needed.
- **Parallelism** — how many pods can run at the same time.
- **Pod Statuses** — counts of active, succeeded, and failed pods.
- **Events** — pod creation and completion events.

### Describe the DaemonSet

```bash
kubectl describe daemonset node-monitor
```

Look at:
- **Node-Selector** — which nodes to target.
- **Number of Nodes Scheduled** — nodes running the DaemonSet pod.
- **Tolerations** — what taints the pod tolerates.

---

## Cleanup

```bash
kubectl delete daemonset node-monitor worker-agent
kubectl delete cronjob hello-cron
kubectl delete job pi-calc batch-job
```

---

## Key Takeaways

1. **Jobs** run pods to completion. Use `completions` for the total number of tasks and `parallelism` for concurrency.
2. Job pods remain in `Completed` status after finishing, so you can view their logs.
3. `backoffLimit` controls how many times a failed pod is retried.
4. **CronJobs** create Jobs on a cron schedule. Use `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` to control history retention.
5. **DaemonSets** ensure one pod per node (or per selected nodes). Common use cases: log collectors, monitoring agents, network plugins.
6. DaemonSets respect node taints. Add tolerations if you want pods on tainted nodes (like the control plane).
7. When a new node joins the cluster, the DaemonSet controller automatically creates a pod on it.

---

## Next Steps

- Proceed to [Lab 11: Scheduling](11-scheduling.md) to learn how Kubernetes decides where pods run.
- Read about [Deployment Strategies](../concepts/deployment-strategies.md) concepts.
