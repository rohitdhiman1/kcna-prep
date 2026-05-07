# Lab 05: Pods

## Objective

Create pods imperatively and declaratively, work with multi-container pods (sidecar pattern), use init containers, inspect logs, exec into containers, and configure resource requests and limits.

## Prerequisites

- A running Kubernetes cluster (from [Lab 01](01-cluster-setup.md)).
- `kubectl` installed and configured.
- Read the concept file: [Pods](../concepts/pods.md)

---

## Step 1: Create a Pod Imperatively

```bash
kubectl run web --image=nginx:1.27 --port=80
```

Verify:

```bash
kubectl get pod web
```

Expected output:

```
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   0          10s
```

---

## Step 2: Create a Pod from YAML

Create a pod manifest:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: web-yaml
  labels:
    app: web
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
EOF
```

Verify:

```bash
kubectl get pod web-yaml -o wide
```

---

## Step 3: Multi-Container Pod with Sidecar

A sidecar container runs alongside the main container in the same pod. They share the same network and can share volumes.

This example runs an nginx container that serves content, and a sidecar that writes the current date into a shared volume every 2 seconds:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: content-writer
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      while true; do
        echo "Page generated at: $(date)" > /html/index.html
        sleep 2
      done
    volumeMounts:
    - name: shared-data
      mountPath: /html
  volumes:
  - name: shared-data
    emptyDir: {}
EOF
```

### Verify both containers are running

```bash
kubectl get pod sidecar-pod
```

Expected output:

```
NAME          READY   STATUS    RESTARTS   AGE
sidecar-pod   2/2     Running   0          15s
```

The `2/2` confirms both containers are running.

### Test the sidecar

```bash
kubectl exec sidecar-pod -c nginx -- curl -s localhost
```

Expected output:

```
Page generated at: Sat Mar 14 12:00:00 UTC 2026
```

Run it again after a few seconds and the timestamp will have changed, proving the sidecar is updating the content.

### View logs for each container

```bash
kubectl logs sidecar-pod -c nginx
kubectl logs sidecar-pod -c content-writer
```

You must specify `-c <container-name>` when a pod has multiple containers.

---

## Step 4: Init Container

Init containers run before the main containers start. They are useful for setup tasks like waiting for a dependency.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
spec:
  initContainers:
  - name: init-wait
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Init container running..."; sleep 5; echo "Init complete"']
  containers:
  - name: app
    image: nginx:1.27
    ports:
    - containerPort: 80
EOF
```

### Watch the pod go through init

```bash
kubectl get pod init-pod --watch
```

Expected progression:

```
NAME       READY   STATUS     RESTARTS   AGE
init-pod   0/1     Init:0/1   0          2s
init-pod   0/1     Init:0/1   0          3s
init-pod   0/1     PodInitializing   0   7s
init-pod   1/1     Running    0          8s
```

Press `Ctrl+C` to stop watching.

### View init container logs

```bash
kubectl logs init-pod -c init-wait
```

Expected output:

```
Init container running...
Init complete
```

---

## Step 5: Check Pod Logs

### Basic logs

```bash
kubectl logs web
```

### Stream logs

```bash
kubectl logs web -f
```

Press `Ctrl+C` to stop.

### Show timestamps

```bash
kubectl logs web --timestamps
```

### Show only the last N lines

```bash
kubectl logs web --tail=5
```

---

## Step 6: Exec into a Pod

### Open an interactive shell

```bash
kubectl exec -it web -- /bin/bash
```

Inside the container:

```bash
hostname
ls /usr/share/nginx/html/
cat /etc/resolv.conf
exit
```

The `/etc/resolv.conf` file shows the cluster DNS configuration, pointing to the CoreDNS service.

### Run a one-off command

```bash
kubectl exec web -- env
```

This prints all environment variables in the container without opening a shell.

---

## Step 7: Resource Requests and Limits

Resource requests guarantee a minimum amount of CPU/memory. Limits set the maximum a container can use.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "250m"
        memory: "128Mi"
EOF
```

### Verify resources are set

```bash
kubectl describe pod resource-pod | grep -A 6 "Limits:"
```

Expected output:

```
    Limits:
      cpu:     250m
      memory:  128Mi
    Requests:
      cpu:        100m
      memory:     64Mi
```

### Check the node to see allocated resources

```bash
kubectl describe node kcna-lab-worker | grep -A 8 "Allocated resources"
```

The resource-pod's requests will appear in the node's allocated resources summary.

### What happens when you exceed limits?

- **CPU:** The container is throttled. It will not be killed.
- **Memory:** The container is OOM-killed and restarted (if the restartPolicy allows it).

---

## Step 8: View Pod Details in YAML

```bash
kubectl get pod resource-pod -o yaml
```

Key fields to examine:
- `spec.containers[].resources` — requests and limits you defined.
- `status.phase` — current phase (Pending, Running, Succeeded, Failed, Unknown).
- `status.conditions` — detailed conditions (PodScheduled, Initialized, ContainersReady, Ready).
- `status.containerStatuses` — state of each container (waiting, running, terminated).

---

## Cleanup

```bash
kubectl delete pod web web-yaml sidecar-pod init-pod resource-pod
```

---

## Key Takeaways

1. A **Pod** is the smallest deployable unit in Kubernetes. It can contain one or more containers.
2. Containers in the same pod share the same network namespace (localhost) and can share volumes.
3. **Sidecar containers** augment the main container (e.g., log shipping, content generation, proxying).
4. **Init containers** run to completion before the main containers start. Use them for setup tasks or dependency checks.
5. Always specify `-c <container-name>` when working with multi-container pods.
6. **Resource requests** influence scheduling (the scheduler finds a node with enough capacity). **Limits** enforce maximum usage.
7. CPU limits cause throttling; memory limits cause OOM kills.
8. Use `--dry-run=client -o yaml` to generate pod YAML templates quickly.

---

## Next Steps

- Proceed to [Lab 06: Deployments](06-deployments.md) to learn how to manage pods at scale.
- Read about [Deployments](../concepts/deployments.md) concepts.
