# Lab 02: Explore the Control Plane

## Objective

Inspect the Kubernetes control plane components running in the kube-system namespace, view their logs, and understand what each component does.

## Prerequisites

- A running Kubernetes cluster (from [Lab 01](01-cluster-setup.md) or similar).
- `kubectl` installed and configured.
- Read the concept file: [Control Plane](../concepts/control-plane.md)

---

## Step 1: List All Pods in the kube-system Namespace

```bash
kubectl get pods -n kube-system
```

Expected output:

```
NAME                                             READY   STATUS    RESTARTS   AGE
coredns-6f6b679f8f-abcde                         1/1     Running   0          10m
coredns-6f6b679f8f-fghij                         1/1     Running   0          10m
etcd-kcna-lab-control-plane                      1/1     Running   0          10m
kindnet-xxxxx                                    1/1     Running   0          10m
kube-apiserver-kcna-lab-control-plane            1/1     Running   0          10m
kube-controller-manager-kcna-lab-control-plane   1/1     Running   0          10m
kube-proxy-xxxxx                                 1/1     Running   0          10m
kube-scheduler-kcna-lab-control-plane            1/1     Running   0          10m
```

The key control plane components are:
- **kube-apiserver** — the front door to the cluster, handles all API requests.
- **etcd** — the key-value store holding all cluster state.
- **kube-scheduler** — decides which node runs each new pod.
- **kube-controller-manager** — runs controllers that reconcile desired vs. actual state.

---

## Step 2: Get More Detail with -o wide

```bash
kubectl get pods -n kube-system -o wide
```

This shows which node each pod is running on and its IP address. Notice that control plane components run on the control plane node.

---

## Step 3: Inspect the API Server

### Describe the API server pod

```bash
kubectl describe pod -n kube-system -l component=kube-apiserver
```

Key things to look for in the output:
- **Image** — the container image version.
- **Command/Args** — the flags the API server was started with (e.g., `--etcd-servers`, `--service-cluster-ip-range`).
- **Ports** — the API server listens on port 6443 by default.
- **Events** — any recent events for this pod.

### View API server logs

```bash
kubectl logs -n kube-system -l component=kube-apiserver --tail=30
```

You will see JSON-formatted log lines showing API requests being processed. Look for entries related to authentication, authorization, and request handling.

---

## Step 4: Inspect etcd

### Describe etcd

```bash
kubectl describe pod -n kube-system -l component=etcd
```

Look at the `Command` section to see etcd's startup flags:
- `--data-dir` — where etcd stores its data.
- `--listen-client-urls` — the URL etcd listens on for client connections.
- `--cert-file` and `--key-file` — TLS certificates used by etcd.

### View etcd logs

```bash
kubectl logs -n kube-system -l component=etcd --tail=20
```

You should see entries about leader elections, compaction, and snapshot operations.

---

## Step 5: Inspect the Scheduler

### Describe the scheduler

```bash
kubectl describe pod -n kube-system -l component=kube-scheduler
```

### View scheduler logs

```bash
kubectl logs -n kube-system -l component=kube-scheduler --tail=20
```

Scheduler logs show decisions about which node was chosen for each pod. When you create pods in later labs, come back here to see the scheduling decisions.

---

## Step 6: Inspect the Controller Manager

### Describe the controller manager

```bash
kubectl describe pod -n kube-system -l component=kube-controller-manager
```

Look at the `Args` section to see which controllers are enabled. The controller manager runs many controllers including:
- Deployment controller
- ReplicaSet controller
- Node controller
- Service account controller

### View controller manager logs

```bash
kubectl logs -n kube-system -l component=kube-controller-manager --tail=20
```

---

## Step 7: Check Component Statuses

```bash
kubectl get componentstatuses
```

Expected output (may vary by Kubernetes version):

```
NAME                 STATUS    MESSAGE   ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   ok
```

**Note:** `componentstatuses` (also abbreviated as `cs`) is deprecated in newer Kubernetes versions. The command may still work but could show warnings. An alternative approach is checking each component individually as done in the previous steps.

```bash
kubectl get cs
```

---

## Step 8: View All Resources in kube-system

To see everything running in the kube-system namespace:

```bash
kubectl get all -n kube-system
```

This lists pods, services, daemonsets, deployments, and replicasets in the kube-system namespace.

---

## Step 9: Examine Static Pod Manifests (Optional)

Control plane components in kubeadm-based clusters (which kind uses) are run as **static pods**. Their manifests live on the control plane node. You can peek at them:

```bash
docker exec kcna-lab-control-plane ls /etc/kubernetes/manifests/
```

Expected output:

```
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

View one of the manifests:

```bash
docker exec kcna-lab-control-plane cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

These YAML files are the source of truth for static pods. The kubelet watches this directory and creates pods automatically from these manifests.

---

## Cleanup

No resources to clean up in this lab. The control plane components are part of the cluster itself.

---

## Key Takeaways

1. The four core control plane components are: **kube-apiserver**, **etcd**, **kube-scheduler**, and **kube-controller-manager**.
2. All control plane components run as pods in the **kube-system** namespace.
3. In kubeadm-based clusters, control plane components are **static pods** managed by the kubelet from manifest files in `/etc/kubernetes/manifests/`.
4. `kubectl logs` and `kubectl describe` work on control plane pods just like any other pod.
5. `kubectl get componentstatuses` provides a quick health check (though it is deprecated).
6. CoreDNS and kube-proxy are also part of kube-system but serve different roles (DNS resolution and network proxying).

---

## Next Steps

- Proceed to [Lab 03: Inspect Worker Nodes](03-inspect-worker-nodes.md) to understand the worker node components.
- Read about [Worker Nodes](../concepts/worker-nodes.md) concepts.
