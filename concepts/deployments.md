# Deployments

**A Deployment is a higher-level controller that manages ReplicaSets and provides declarative updates, rolling updates, and rollbacks for your Pods.**

---

## Table of Contents

- [What Is a Deployment?](#what-is-a-deployment)
- [Deployment Hierarchy Diagram](#deployment-hierarchy-diagram)
- [Deployment Spec](#deployment-spec)
- [Rolling Update Strategy](#rolling-update-strategy)
- [Rollback](#rollback)
- [Scaling](#scaling)
- [Deployment Status and Conditions](#deployment-status-and-conditions)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## What Is a Deployment?

A Deployment provides:
- **Declarative updates** for Pods via ReplicaSets
- **Rolling updates** — gradually replace old Pods with new ones
- **Rollback** — revert to a previous version if something goes wrong
- **Scaling** — increase or decrease the number of replicas
- **Self-healing** — automatically replaces failed Pods

You almost never create ReplicaSets or Pods directly. Instead, you create a **Deployment**, and it manages everything below it.

---

## Deployment Hierarchy Diagram

```
+================================================================+
|                        DEPLOYMENT                              |
|  name: web-app                                                 |
|  replicas: 3                                                   |
|  strategy: RollingUpdate                                       |
|                                                                |
|  Manages                                                       |
|    |                                                           |
|    v                                                           |
|  +----------------------------------------------------------+  |
|  |                    REPLICASET                             |  |
|  |  name: web-app-7d9f8b6c5                                 |  |
|  |  replicas: 3                                              |  |
|  |  selector: app=web-app, pod-template-hash=7d9f8b6c5      |  |
|  |                                                           |  |
|  |  Manages                                                  |  |
|  |    |          |          |                                |  |
|  |    v          v          v                                |  |
|  |  +------+  +------+  +------+                            |  |
|  |  | Pod  |  | Pod  |  | Pod  |                            |  |
|  |  | web- |  | web- |  | web- |                            |  |
|  |  | app- |  | app- |  | app- |                            |  |
|  |  | xxxx |  | yyyy |  | zzzz |                            |  |
|  |  +------+  +------+  +------+                            |  |
|  +----------------------------------------------------------+  |
+================================================================+
```

### Ownership Chain

```
  Deployment
      |  owns (via ownerReferences)
      v
  ReplicaSet
      |  owns (via ownerReferences)
      v
  Pod, Pod, Pod
```

When you delete a Deployment, it cascades: the ReplicaSet is deleted, and then the Pods are deleted.

---

## Deployment Spec

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3                    # Desired number of Pods
  selector:                      # How to find Pods this Deployment manages
    matchLabels:
      app: web-app
  strategy:                      # How to perform updates
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1          # Max Pods that can be unavailable during update
      maxSurge: 1                # Max extra Pods above desired count
  template:                      # Pod template — what each Pod looks like
    metadata:
      labels:
        app: web-app             # MUST match selector.matchLabels
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

### Key Fields Explained

```
  spec.replicas           How many Pods to run
  spec.selector           Label selector to identify managed Pods
  spec.strategy           RollingUpdate (default) or Recreate
  spec.template           Pod template (metadata + spec)
  spec.template.metadata  Labels MUST match spec.selector
  spec.template.spec      The actual Pod spec (containers, volumes, etc.)
```

> **Important**: The `selector.matchLabels` must match `template.metadata.labels`. If they don't, the Deployment will be rejected.

---

## Rolling Update Strategy

When you update a Deployment (e.g., change the container image), Kubernetes performs a **rolling update** by default.

### How It Works

```
  BEFORE UPDATE: Deployment running nginx:1.24, replicas=3

  ReplicaSet v1 (nginx:1.24)
  +------+  +------+  +------+
  | Pod  |  | Pod  |  | Pod  |
  | v1   |  | v1   |  | v1   |
  +------+  +------+  +------+


  DURING ROLLING UPDATE: Changing to nginx:1.25

  Step 1: New ReplicaSet created, scale up 1, scale down 1
  ReplicaSet v1 (nginx:1.24)         ReplicaSet v2 (nginx:1.25)
  +------+  +------+                 +------+
  | Pod  |  | Pod  |                 | Pod  |
  | v1   |  | v1   |                 | v2   |
  +------+  +------+                 +------+

  Step 2: Continue rolling
  ReplicaSet v1 (nginx:1.24)         ReplicaSet v2 (nginx:1.25)
  +------+                           +------+  +------+
  | Pod  |                           | Pod  |  | Pod  |
  | v1   |                           | v2   |  | v2   |
  +------+                           +------+  +------+

  Step 3: Complete
  ReplicaSet v1 (scaled to 0)       ReplicaSet v2 (nginx:1.25)
  (kept for rollback)                +------+  +------+  +------+
                                     | Pod  |  | Pod  |  | Pod  |
                                     | v2   |  | v2   |  | v2   |
                                     +------+  +------+  +------+


  AFTER UPDATE: Old ReplicaSet kept (0 replicas) for rollback
```

### Strategy Parameters

| Parameter        | Default | Meaning                                          |
|------------------|---------|--------------------------------------------------|
| `maxUnavailable` | 25%     | Max Pods that can be down during the update       |
| `maxSurge`       | 25%     | Max extra Pods above desired count during update  |

With replicas=4:
- `maxUnavailable: 1` means at least 3 Pods must be available during the update
- `maxSurge: 1` means at most 5 Pods can exist at any time during the update

### Recreate Strategy

An alternative strategy that kills all old Pods before creating new ones:

```
  BEFORE:
  +------+  +------+  +------+
  | v1   |  | v1   |  | v1   |
  +------+  +------+  +------+

  DURING (downtime!):
  (no pods running)

  AFTER:
  +------+  +------+  +------+
  | v2   |  | v2   |  | v2   |
  +------+  +------+  +------+
```

- Causes downtime
- Useful when you cannot run two versions simultaneously (e.g., database schema changes)

---

## Rollback

Kubernetes keeps old ReplicaSets (scaled to 0) so you can rollback to a previous version.

### Rollout Commands

```bash
# Check rollout status
kubectl rollout status deployment/web-app

# View rollout history
kubectl rollout history deployment/web-app

# View details of a specific revision
kubectl rollout history deployment/web-app --revision=2

# Rollback to the previous version
kubectl rollout undo deployment/web-app

# Rollback to a specific revision
kubectl rollout undo deployment/web-app --to-revision=2

# Pause a rollout (for canary-style testing)
kubectl rollout pause deployment/web-app

# Resume a paused rollout
kubectl rollout resume deployment/web-app
```

### Rollback Flow

```
  Current State:
  RS v1 (0 replicas)    RS v2 (3 replicas)  <-- current
  (nginx:1.24)           (nginx:1.25)

  kubectl rollout undo deployment/web-app

  After Rollback:
  RS v1 (3 replicas)  <-- rolled back to this
  (nginx:1.24)
  RS v2 (0 replicas)
  (nginx:1.25)
```

### Revision History Limit

By default, Kubernetes keeps the last **10 ReplicaSets** for rollback. Controlled by `spec.revisionHistoryLimit`.

---

## Scaling

### Manual Scaling

```bash
# Scale to 5 replicas
kubectl scale deployment/web-app --replicas=5

# Scale to 0 (stop all pods)
kubectl scale deployment/web-app --replicas=0
```

### What Happens When You Scale

```
  kubectl scale deployment/web-app --replicas=5

  Before:                         After:
  +------+  +------+  +------+   +------+  +------+  +------+  +------+  +------+
  | Pod  |  | Pod  |  | Pod  |   | Pod  |  | Pod  |  | Pod  |  | Pod  |  | Pod  |
  +------+  +------+  +------+   +------+  +------+  +------+  +------+  +------+
  (3 replicas)                    (5 replicas)
```

Scaling does NOT create a new ReplicaSet — it updates the replica count on the existing one.

### Horizontal Pod Autoscaler (HPA)

For automatic scaling based on metrics:

```bash
kubectl autoscale deployment/web-app --min=2 --max=10 --cpu-percent=80
```

This creates an HPA that scales between 2-10 replicas based on CPU utilization.

---

## Deployment Status and Conditions

### Checking Status

```bash
# Quick status
kubectl get deployment web-app

NAME      READY   UP-TO-DATE   AVAILABLE   AGE
web-app   3/3     3            3           5m

# READY:      current/desired replicas that are ready
# UP-TO-DATE: replicas matching the current template
# AVAILABLE:  replicas available to serve traffic
```

### Deployment Conditions

| Condition     | Meaning                                          |
|---------------|--------------------------------------------------|
| Available     | Minimum required replicas are available           |
| Progressing   | Deployment is creating/updating/scaling replicas  |
| ReplicaFailure| Deployment cannot create a Pod (e.g., bad image) |

---

## What to Remember for the Exam

1. **Deployment manages ReplicaSets**, which manage Pods. You almost never create ReplicaSets directly.

2. **Rolling update is the default strategy**. It gradually replaces old Pods with new ones, ensuring availability during updates.

3. **maxUnavailable** and **maxSurge** control the rolling update pace. Defaults are 25% each.

4. **Recreate strategy** kills all old Pods before creating new ones — causes downtime.

5. **Old ReplicaSets are kept** (scaled to 0) for rollback capability. Default history limit is 10.

6. **Rollback commands**: `kubectl rollout undo deployment/name` to go back, `kubectl rollout history` to see revisions.

7. **Scaling** changes the replica count on the current ReplicaSet. It does NOT trigger a new rollout.

8. **selector.matchLabels must match template.metadata.labels**. If they don't match, the Deployment will be rejected.

9. **Changing the Pod template** (image, env vars, resources, etc.) triggers a new rollout with a new ReplicaSet.

10. **Deployments are the standard way** to run stateless applications in Kubernetes. For stateful apps, use StatefulSets.
