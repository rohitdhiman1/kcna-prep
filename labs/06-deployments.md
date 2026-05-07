# Lab 06: Deployments

## Objective

Create a Deployment, scale it up and down, perform a rolling update by changing the container image, roll back to a previous revision, and inspect rollout status and history.

## Prerequisites

- A running Kubernetes cluster (from [Lab 01](01-cluster-setup.md)).
- `kubectl` installed and configured.
- Read the concept file: [Deployments](../concepts/deployments.md)

---

## Step 1: Create a Deployment

```bash
kubectl create deployment web-app --image=nginx:1.26 --replicas=3
```

### Verify the deployment

```bash
kubectl get deployment web-app
```

Expected output:

```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
web-app   3/3     3            3           20s
```

### Check the ReplicaSet created by the deployment

```bash
kubectl get replicaset
```

Expected output:

```
NAME                 DESIRED   CURRENT   READY   AGE
web-app-6b7f4c5d9f   3         3         3       25s
```

A Deployment creates and manages a ReplicaSet, which in turn manages the pods.

### Check the pods

```bash
kubectl get pods -l app=web-app
```

Expected output:

```
NAME                       READY   STATUS    RESTARTS   AGE
web-app-6b7f4c5d9f-abc12   1/1     Running   0          30s
web-app-6b7f4c5d9f-def34   1/1     Running   0          30s
web-app-6b7f4c5d9f-ghi56   1/1     Running   0          30s
```

---

## Step 2: Scale the Deployment

### Scale up to 5 replicas

```bash
kubectl scale deployment web-app --replicas=5
```

### Verify

```bash
kubectl get deployment web-app
kubectl get pods -l app=web-app
```

You should now see 5 pods running.

### Scale down to 2 replicas

```bash
kubectl scale deployment web-app --replicas=2
```

```bash
kubectl get pods -l app=web-app
```

Three pods will terminate, leaving 2 running.

---

## Step 3: Perform a Rolling Update

Update the nginx image from 1.26 to 1.27:

```bash
kubectl set image deployment/web-app nginx=nginx:1.27
```

### Watch the rollout

```bash
kubectl rollout status deployment/web-app
```

Expected output:

```
Waiting for deployment "web-app" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "web-app" rollout to finish: 1 old replicas are pending termination...
deployment "web-app" successfully rolled out
```

### Observe what happened to ReplicaSets

```bash
kubectl get replicaset
```

Expected output:

```
NAME                 DESIRED   CURRENT   READY   AGE
web-app-6b7f4c5d9f   0         0         0       5m    # old (nginx:1.26)
web-app-7c8d5e6f0a   2         2         2       30s   # new (nginx:1.27)
```

The old ReplicaSet is scaled to 0 but kept around for rollback. The new ReplicaSet has the updated pods.

### Verify the image was updated

```bash
kubectl describe deployment web-app | grep Image
```

Expected output:

```
    Image:        nginx:1.27
```

---

## Step 4: Check Rollout History

```bash
kubectl rollout history deployment/web-app
```

Expected output:

```
deployment.apps/web-app
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

### Add a change cause annotation for better tracking

Perform another update with an annotation:

```bash
kubectl set image deployment/web-app nginx=nginx:1.25 --record=false
kubectl annotate deployment/web-app kubernetes.io/change-cause="Downgrade to nginx 1.25 for testing"
```

Check history again:

```bash
kubectl rollout history deployment/web-app
```

Expected output:

```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         Downgrade to nginx 1.25 for testing
```

### View details of a specific revision

```bash
kubectl rollout history deployment/web-app --revision=2
```

This shows the pod template used in revision 2, including the image.

---

## Step 5: Rollback to a Previous Revision

### Rollback to the immediately previous revision

```bash
kubectl rollout undo deployment/web-app
```

Expected output:

```
deployment.apps/web-app rolled back
```

### Verify the rollback

```bash
kubectl rollout status deployment/web-app
kubectl describe deployment web-app | grep Image
```

The image should now be `nginx:1.27` (revision 2).

### Rollback to a specific revision

```bash
kubectl rollout undo deployment/web-app --to-revision=1
```

```bash
kubectl describe deployment web-app | grep Image
```

The image should now be `nginx:1.26` (the original revision 1).

---

## Step 6: Examine Rolling Update Strategy

```bash
kubectl get deployment web-app -o yaml | grep -A 10 strategy
```

Expected output:

```yaml
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
```

- **maxSurge** — maximum number of pods above the desired count during an update (25% of 2 replicas = 1 extra pod).
- **maxUnavailable** — maximum number of pods that can be unavailable during the update.

### Modify the strategy

```bash
kubectl patch deployment web-app -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
```

This ensures zero downtime by never allowing unavailable pods during the update.

---

## Step 7: Watch a Rollout in Detail

Scale up to see the rolling update more clearly:

```bash
kubectl scale deployment web-app --replicas=5
```

Now trigger an update and watch pods:

```bash
kubectl set image deployment/web-app nginx=nginx:1.27
kubectl get pods -l app=web-app --watch
```

You will see pods terminating and new pods being created in a rolling fashion. Press `Ctrl+C` to stop watching.

---

## Cleanup

```bash
kubectl delete deployment web-app
```

---

## Key Takeaways

1. A **Deployment** manages pods through **ReplicaSets**. You rarely interact with ReplicaSets directly.
2. `kubectl scale` adjusts the number of replicas.
3. `kubectl set image` triggers a **rolling update**. New pods are created before old pods are removed.
4. Old ReplicaSets are preserved (scaled to 0) to enable rollbacks.
5. `kubectl rollout undo` rolls back to a previous revision. Use `--to-revision=N` for a specific revision.
6. `kubectl rollout history` shows all revisions. Annotate deployments with change causes for better tracking.
7. The rolling update strategy (`maxSurge` and `maxUnavailable`) controls how aggressively updates are applied.
8. `kubectl rollout status` blocks until the rollout completes, useful in CI/CD pipelines.

---

## Next Steps

- Proceed to [Lab 07: Services](07-services.md) to expose your deployments to network traffic.
- Read about [Services](../concepts/services.md) concepts.
