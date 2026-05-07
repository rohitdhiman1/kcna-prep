# Lab 13: Inspecting Container Runtimes

## Objective

Identify the container runtime used by your Kubernetes cluster and explore it using `crictl` and `ctr` commands to understand how containers and images are managed at the runtime level.

## Prerequisites

- A running Kubernetes cluster (kind, minikube, or cloud).
- `kubectl` installed and configured.
- SSH access to a cluster node (for `crictl` and `ctr` commands).
- Read the concept file: [Container Runtimes](../concepts/container-runtimes.md)

---

## Step 1: Check the Container Runtime via kubectl

The simplest way to see which container runtime your cluster uses is:

```bash
kubectl get nodes -o wide
```

Expected output:

```
NAME                 STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
kind-control-plane   Ready    control-plane   10d   v1.29.2   172.18.0.2    <none>        Ubuntu 22.04.3 LTS   6.1.0            containerd://1.7.13
```

The `CONTAINER-RUNTIME` column shows the runtime and its version. Common values:
- `containerd://1.x.x` — containerd (most common today)
- `cri-o://1.x.x` — CRI-O
- `docker://24.x.x` — Docker (older clusters, uses dockershim or cri-dockerd)

### Get runtime info in JSON format

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.containerRuntimeVersion}{"\n"}{end}'
```

This extracts only the node name and runtime version.

---

## Step 2: Examine Node Info in Detail

```bash
kubectl describe node <your-node-name>
```

Look for the `System Info` section:

```
System Info:
  ...
  Container Runtime Version:  containerd://1.7.13
  Kubelet Version:             v1.29.2
  Kube-Proxy Version:          v1.29.2
  Operating System:            linux
  Architecture:                amd64
```

This confirms the runtime, kubelet version, and OS details.

---

## Step 3: Access a Cluster Node

To run `crictl` and `ctr`, you need shell access to a node.

### For kind clusters

```bash
docker exec -it kind-control-plane bash
```

### For minikube

```bash
minikube ssh
```

### For cloud clusters

SSH into a worker node using your cloud provider's method (e.g., `gcloud compute ssh`, `aws ssm start-session`).

---

## Step 4: Use crictl to List Containers

`crictl` is the CLI tool for CRI-compatible container runtimes. It works with both containerd and CRI-O.

### List running containers

```bash
crictl ps
```

Expected output:

```
CONTAINER           IMAGE               CREATED             STATE       NAME                      ATTEMPT   POD ID              POD
a1b2c3d4e5f6        registry.k8s.io/..  2 hours ago         Running     etcd                      0         f1e2d3c4b5a6        etcd-kind-control-plane
b2c3d4e5f6a7        registry.k8s.io/..  2 hours ago         Running     kube-apiserver            0         e2d3c4b5a6f1        kube-apiserver-kind-control-plane
...
```

### List all containers (including stopped)

```bash
crictl ps -a
```

### List containers with additional details

```bash
crictl ps --output yaml
```

---

## Step 5: Use crictl to List Images

### List all images on the node

```bash
crictl images
```

Expected output:

```
IMAGE                                      TAG                  IMAGE ID            SIZE
docker.io/kindest/kindnetd                 v20240202-8f1494ea   b0b1f7f3d981e       27.7MB
registry.k8s.io/coredns/coredns            v1.11.1              cbb01a7bd410d       16.6MB
registry.k8s.io/etcd                       3.5.12-0             014faa467e29c       64.5MB
registry.k8s.io/kube-apiserver             v1.29.2              96d4c09573e6a       33.8MB
...
```

### Get image details

```bash
crictl inspecti registry.k8s.io/pause:3.9
```

This shows the full image metadata including labels and configuration.

---

## Step 6: Use crictl to Inspect Pods

### List pods

```bash
crictl pods
```

Expected output:

```
POD ID              CREATED             STATE     NAME                                         NAMESPACE         ATTEMPT   RUNTIME
f1e2d3c4b5a6        2 hours ago         Ready     etcd-kind-control-plane                      kube-system       0         (default)
e2d3c4b5a6f1        2 hours ago         Ready     kube-apiserver-kind-control-plane             kube-system       0         (default)
...
```

### Filter pods by namespace

```bash
crictl pods --namespace kube-system
```

### Inspect a specific pod

```bash
crictl inspectp <pod-id>
```

---

## Step 7: Explore containerd with ctr

If your cluster uses containerd, you can use `ctr` for lower-level operations. Note that `ctr` interacts directly with containerd, bypassing the CRI layer.

### List containerd namespaces

```bash
ctr namespaces list
```

Expected output:

```
NAME    LABELS
k8s.io
moby
```

Kubernetes uses the `k8s.io` namespace in containerd.

### List containers in the k8s.io namespace

```bash
ctr --namespace k8s.io containers list
```

### List images in the k8s.io namespace

```bash
ctr --namespace k8s.io images list
```

This often produces a long list. Filter with grep:

```bash
ctr --namespace k8s.io images list | grep coredns
```

### Check containerd status

```bash
ctr version
```

Expected output:

```
Client:
  Version:  1.7.13
  Revision: ...
  Go version: go1.21.6

Server:
  Version:  1.7.13
  Revision: ...
```

---

## Step 8: Compare crictl and ctr

| Feature | crictl | ctr |
|---------|--------|-----|
| Purpose | CRI client (Kubernetes-aware) | containerd native client |
| Works with CRI-O | Yes | No |
| Shows pods | Yes | No |
| Namespace default | Uses CRI namespace | Must specify `k8s.io` |
| Use case | Troubleshooting K8s containers | Low-level containerd debugging |

### Key insight

For day-to-day Kubernetes troubleshooting, use `crictl`. Use `ctr` only when you need to debug containerd itself.

---

## Step 9: Check Runtime Configuration (Optional)

### View containerd config

```bash
cat /etc/containerd/config.toml
```

Look for:
- `[plugins."io.containerd.grpc.v1.cri"]` — CRI plugin settings
- `sandbox_image` — the pause container image used
- `SystemdCgroup` — whether systemd cgroups are enabled

### Check the CRI socket

```bash
ls -la /var/run/containerd/containerd.sock
```

This is the Unix socket that kubelet uses to communicate with containerd via CRI.

---

## Cleanup

If you accessed a node via `docker exec` or `minikube ssh`, simply exit:

```bash
exit
```

No cluster resources were created in this lab.

---

## Key Takeaways

1. `kubectl get nodes -o wide` reveals the container runtime and version for each node.
2. **crictl** is the standard CRI debugging tool that works with containerd and CRI-O.
3. **ctr** is containerd-specific and operates at a lower level than crictl.
4. Kubernetes containers live in the `k8s.io` namespace within containerd.
5. The container runtime is responsible for pulling images, running containers, and managing their lifecycle.
6. Docker was removed as a supported runtime in Kubernetes v1.24; containerd and CRI-O are now the standard choices.

---

## Next Steps

- Read about [Networking](../concepts/networking.md) to understand how containers communicate.
- Try [Lab 14: Pod Networking](14-networking.md) to explore network connectivity between pods.
