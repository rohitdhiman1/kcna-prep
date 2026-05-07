# Lab 11: Scheduling

## Objective

Use nodeSelector to schedule pods on specific nodes, apply taints to nodes and add tolerations to pods, configure resource requests and limits, and observe scheduling decisions through pod events.

## Prerequisites

- A running multi-node Kubernetes cluster (from [Lab 01](01-cluster-setup.md)).
- `kubectl` installed and configured.
- Read the concept file: [Kubernetes Architecture](../concepts/kubernetes-architecture.md)

---

## Step 1: Review Current Node Labels

```bash
kubectl get nodes --show-labels
```

Note the default labels on each node (e.g., `kubernetes.io/hostname`, `kubernetes.io/os`).

---

## Step 2: Use nodeSelector to Pin a Pod to a Specific Node

### Label a worker node

```bash
kubectl label node kcna-lab-worker disktype=ssd
```

### Verify the label

```bash
kubectl get node kcna-lab-worker --show-labels | grep disktype
```

### Create a pod with nodeSelector

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: app
    image: nginx:1.27
EOF
```

### Verify the pod was scheduled on the labeled node

```bash
kubectl get pod ssd-pod -o wide
```

Expected output:

```
NAME      READY   STATUS    RESTARTS   AGE   IP           NODE               ...
ssd-pod   1/1     Running   0          10s   10.244.1.5   kcna-lab-worker    ...
```

The pod is running on `kcna-lab-worker` because that is the only node with `disktype=ssd`.

### Check the scheduling events

```bash
kubectl describe pod ssd-pod
```

Look at the **Events** section at the bottom:

```
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  15s   default-scheduler  Successfully assigned default/ssd-pod to kcna-lab-worker
  Normal  Pulling    14s   kubelet            Pulling image "nginx:1.27"
  Normal  Pulled     10s   kubelet            Successfully pulled image "nginx:1.27"
  Normal  Created    10s   kubelet            Created container app
  Normal  Started    10s   kubelet            Started container app
```

The `Scheduled` event confirms the scheduler's decision.

---

## Step 3: Observe Unschedulable Pod (No Matching Node)

Create a pod with a nodeSelector that no node satisfies:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    accelerator: gpu
  containers:
  - name: app
    image: nginx:1.27
EOF
```

### Check the pod status

```bash
kubectl get pod gpu-pod
```

Expected output:

```
NAME      READY   STATUS    RESTARTS   AGE
gpu-pod   0/1     Pending   0          10s
```

The pod stays in `Pending` because no node has the label `accelerator=gpu`.

### Check the events for the reason

```bash
kubectl describe pod gpu-pod
```

Look at the **Events** section:

```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  15s   default-scheduler  0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.
```

This clearly explains why the pod cannot be scheduled.

---

## Step 4: Taints and Tolerations

Taints are applied to nodes to repel pods. Tolerations are added to pods to allow them onto tainted nodes.

### Taint a worker node

```bash
kubectl taint nodes kcna-lab-worker2 environment=restricted:NoSchedule
```

This means: no pod can be scheduled on `kcna-lab-worker2` unless it tolerates this taint.

### Verify the taint

```bash
kubectl describe node kcna-lab-worker2 | grep Taints
```

Expected output:

```
Taints:             environment=restricted:NoSchedule
```

### Create a pod without toleration

```bash
kubectl run no-tolerate --image=nginx:1.27
```

```bash
kubectl get pod no-tolerate -o wide
```

The pod will be scheduled on `kcna-lab-worker` (or the control plane, if it has no taint), but NOT on `kcna-lab-worker2`.

### Create a pod with toleration

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tolerate-pod
spec:
  tolerations:
  - key: "environment"
    operator: "Equal"
    value: "restricted"
    effect: "NoSchedule"
  containers:
  - name: app
    image: nginx:1.27
EOF
```

```bash
kubectl get pod tolerate-pod -o wide
```

This pod CAN be scheduled on `kcna-lab-worker2` because it tolerates the taint. However, having a toleration does not force the pod onto that node -- the scheduler may still place it elsewhere.

### Check scheduling events

```bash
kubectl describe pod tolerate-pod | grep -A 3 "Events:"
```

---

## Step 5: Taint Effects

There are three taint effects:

- **NoSchedule** — pods without the toleration will not be scheduled on the node.
- **PreferNoSchedule** — the scheduler tries to avoid the node but may use it if no other option exists.
- **NoExecute** — pods without the toleration are evicted from the node if already running.

### Demonstrate NoExecute

First, create a pod on worker2:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: evict-test
spec:
  nodeName: kcna-lab-worker2
  containers:
  - name: app
    image: nginx:1.27
EOF
```

Verify it is running:

```bash
kubectl get pod evict-test -o wide
```

Now apply a NoExecute taint:

```bash
kubectl taint nodes kcna-lab-worker2 urgent=true:NoExecute
```

Check the pod:

```bash
kubectl get pod evict-test
```

The pod should be evicted and terminated because it does not tolerate the `urgent=true:NoExecute` taint.

### Remove the NoExecute taint

```bash
kubectl taint nodes kcna-lab-worker2 urgent=true:NoExecute-
```

The minus sign (`-`) at the end removes the taint.

---

## Step 6: Resource Requests and Scheduling

The scheduler considers resource requests when placing pods. A pod will only be scheduled on a node with enough allocatable resources.

### Check available resources on nodes

```bash
kubectl describe node kcna-lab-worker | grep -A 6 "Allocated resources"
```

### Create a pod with high resource requests

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: big-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests:
        cpu: "100"
        memory: "100Gi"
EOF
```

### Check why it is pending

```bash
kubectl get pod big-pod
```

Expected output:

```
NAME      READY   STATUS    RESTARTS   AGE
big-pod   0/1     Pending   0          5s
```

```bash
kubectl describe pod big-pod
```

Look at the Events:

```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  10s   default-scheduler  0/3 nodes are available: 3 Insufficient cpu, 3 Insufficient memory.
```

No node has 100 CPUs and 100Gi of memory, so the pod remains Pending.

---

## Step 7: View Scheduler Decisions with Events

The Events section of `kubectl describe pod` is the primary way to understand scheduling:

```bash
kubectl get events --field-selector reason=FailedScheduling
```

This lists all recent failed scheduling events across the namespace.

```bash
kubectl get events --sort-by='.lastTimestamp' --field-selector involvedObject.kind=Pod
```

This shows all pod-related events sorted by time.

---

## Cleanup

```bash
kubectl delete pod ssd-pod gpu-pod no-tolerate tolerate-pod evict-test big-pod --ignore-not-found
kubectl label node kcna-lab-worker disktype-
kubectl taint nodes kcna-lab-worker2 environment=restricted:NoSchedule-
```

---

## Key Takeaways

1. **nodeSelector** is the simplest way to constrain pods to specific nodes using labels.
2. If no node matches the nodeSelector, the pod stays in `Pending` with a `FailedScheduling` event.
3. **Taints** repel pods from nodes. **Tolerations** allow pods to be scheduled on tainted nodes.
4. Three taint effects: `NoSchedule` (prevent scheduling), `PreferNoSchedule` (soft preference), `NoExecute` (evict existing pods).
5. **Resource requests** affect scheduling. The scheduler only places a pod on a node with enough allocatable resources.
6. Resource **limits** do not affect scheduling, only requests do.
7. The **Events** section of `kubectl describe pod` is the best tool for debugging scheduling issues.
8. Use `kubectl get events` to see cluster-wide scheduling activity.

---

## Next Steps

- Proceed to [Lab 12: Containers](12-containers.md) to work with container images.
- Read about [Container Runtimes](../concepts/container-runtimes.md) concepts.
