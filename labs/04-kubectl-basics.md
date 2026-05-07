# Lab 04: kubectl Basics

## Objective

Practice essential kubectl commands using both imperative and declarative approaches. Learn to use various output formats and master the most common operations: create, get, describe, delete, apply, logs, exec, and port-forward.

## Prerequisites

- A running Kubernetes cluster (from [Lab 01](01-cluster-setup.md)).
- `kubectl` installed and configured.
- Read the concept file: [Kubernetes API](../concepts/kubernetes-api.md)

---

## Step 1: Imperative Commands — Create Resources Directly

Imperative commands tell Kubernetes exactly what to do right now.

### Create a pod imperatively

```bash
kubectl run nginx-imp --image=nginx:1.27
```

Expected output:

```
pod/nginx-imp created
```

### Create a deployment imperatively

```bash
kubectl create deployment web-imp --image=httpd:2.4 --replicas=2
```

Expected output:

```
deployment.apps/web-imp created
```

### Expose the deployment as a service

```bash
kubectl expose deployment web-imp --port=80 --target-port=80 --type=ClusterIP
```

Expected output:

```
service/web-imp exposed
```

---

## Step 2: Declarative Commands — Apply from YAML

Declarative commands describe the desired state and let Kubernetes figure out how to achieve it.

### Generate a YAML manifest (dry-run)

Instead of writing YAML from scratch, generate it:

```bash
kubectl run nginx-decl --image=nginx:1.27 --dry-run=client -o yaml
```

This prints the YAML without creating anything. Save it to a file:

```bash
kubectl run nginx-decl --image=nginx:1.27 --dry-run=client -o yaml > nginx-pod.yaml
```

### Apply the manifest

```bash
kubectl apply -f nginx-pod.yaml
```

Expected output:

```
pod/nginx-decl created
```

### Modify and re-apply

The power of declarative management is idempotent updates. Edit the file (e.g., add a label), then apply again:

```bash
kubectl apply -f nginx-pod.yaml
```

If nothing changed, the output is:

```
pod/nginx-decl unchanged
```

---

## Step 3: Get Resources

### Basic get

```bash
kubectl get pods
```

### Get with wide output

```bash
kubectl get pods -o wide
```

Shows additional columns: node, IP, nominated node, readiness gates.

### Get all resource types

```bash
kubectl get all
```

Lists pods, services, deployments, replicasets in the current namespace.

---

## Step 4: Output Formats

### YAML output

```bash
kubectl get pod nginx-imp -o yaml
```

Shows the complete resource definition including status. Useful for seeing the full spec and understanding defaults Kubernetes added.

### JSON output

```bash
kubectl get pod nginx-imp -o json
```

Same information as YAML, in JSON format. Useful for scripting with tools like `jq`.

### JSONPath — extract specific fields

Get just the pod IP:

```bash
kubectl get pod nginx-imp -o jsonpath='{.status.podIP}'
```

Get the container image:

```bash
kubectl get pod nginx-imp -o jsonpath='{.spec.containers[0].image}'
```

List all pod names and their statuses:

```bash
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

Expected output:

```
nginx-decl	Running
nginx-imp	Running
```

### Custom columns

```bash
kubectl get pods -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP,NODE:.spec.nodeName'
```

Expected output:

```
NAME         STATUS    IP            NODE
nginx-decl   Running   10.244.1.3    kcna-lab-worker
nginx-imp    Running   10.244.2.2    kcna-lab-worker2
```

---

## Step 5: Describe Resources

```bash
kubectl describe pod nginx-imp
```

Key sections in the output:
- **Labels and Annotations** — metadata attached to the resource.
- **Containers** — image, ports, resource requests/limits, mounts.
- **Conditions** — pod status conditions (Initialized, Ready, ContainersReady, PodScheduled).
- **Events** — timeline of what happened (scheduled, pulling image, started container).

The Events section is especially valuable for troubleshooting.

### Describe a deployment

```bash
kubectl describe deployment web-imp
```

Look at the **Replicas** field and the **Events** for scaling events.

---

## Step 6: View Logs

### Get pod logs

```bash
kubectl logs nginx-imp
```

If the pod has not received any traffic, the output may be empty.

### Stream logs in real time

```bash
kubectl logs nginx-imp -f
```

Press `Ctrl+C` to stop streaming.

### View previous container logs (if it restarted)

```bash
kubectl logs nginx-imp --previous
```

This is useful when a container has crashed and restarted.

### View logs with timestamps

```bash
kubectl logs nginx-imp --timestamps
```

---

## Step 7: Execute Commands Inside a Pod

### Open an interactive shell

```bash
kubectl exec -it nginx-imp -- /bin/bash
```

Once inside the container, try:

```bash
hostname
cat /etc/nginx/nginx.conf
curl localhost
exit
```

### Run a single command without interactive mode

```bash
kubectl exec nginx-imp -- cat /etc/os-release
```

### Run a command in a specific container (for multi-container pods)

```bash
kubectl exec nginx-imp -c nginx-imp -- hostname
```

---

## Step 8: Port Forwarding

Forward a local port to a pod's port to access it from your machine:

```bash
kubectl port-forward pod/nginx-imp 8080:80
```

In another terminal, test it:

```bash
curl http://localhost:8080
```

Expected output: the default nginx welcome page HTML.

Press `Ctrl+C` to stop port-forwarding.

### Port-forward to a service

```bash
kubectl port-forward service/web-imp 8081:80
```

This forwards local port 8081 to the service, which load-balances across its pods.

---

## Step 9: Delete Resources

### Delete a single pod

```bash
kubectl delete pod nginx-imp
```

### Delete using a manifest file

```bash
kubectl delete -f nginx-pod.yaml
```

### Delete a deployment (also deletes its pods and replicaset)

```bash
kubectl delete deployment web-imp
```

### Delete a service

```bash
kubectl delete service web-imp
```

### Delete multiple resources at once

```bash
kubectl delete pod,service --all
```

**Caution:** This deletes all pods and services in the current namespace.

---

## Cleanup

```bash
kubectl delete pod nginx-imp --ignore-not-found
kubectl delete pod nginx-decl --ignore-not-found
kubectl delete deployment web-imp --ignore-not-found
kubectl delete service web-imp --ignore-not-found
rm -f nginx-pod.yaml
```

---

## Key Takeaways

1. **Imperative commands** (`kubectl run`, `kubectl create`) are quick but not repeatable. Good for one-off tasks.
2. **Declarative commands** (`kubectl apply -f`) use YAML manifests and are idempotent. Good for version-controlled, repeatable deployments.
3. Use `--dry-run=client -o yaml` to generate YAML templates instead of writing them from scratch.
4. Output formats: `-o yaml`, `-o json`, `-o wide`, `-o jsonpath`, and `-o custom-columns` provide different levels of detail.
5. `kubectl describe` shows events and conditions, which are essential for troubleshooting.
6. `kubectl logs` retrieves container stdout/stderr. Use `-f` for streaming, `--previous` for crashed containers.
7. `kubectl exec -it` opens a shell inside a running container for debugging.
8. `kubectl port-forward` creates a tunnel from your local machine to a pod or service.

---

## Next Steps

- Proceed to [Lab 05: Pods](05-pods.md) to dive deeper into pod creation and configuration.
- Read about [Pods](../concepts/pods.md) concepts.
