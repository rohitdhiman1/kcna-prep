# Lab 01: Set Up a Local Kubernetes Cluster with kind

## Objective

Create a multi-node local Kubernetes cluster using kind (Kubernetes in Docker), verify it is running with kubectl, and explore the nodes in the cluster.

## Prerequisites

- Docker installed and running.
- `kind` installed ([installation guide](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)).
- `kubectl` installed and configured.
- Read the concept file: [Kubernetes Architecture](../concepts/kubernetes-architecture.md)

---

## Step 1: Create a Multi-Node kind Cluster Configuration

Create a kind configuration file that defines one control plane node and two worker nodes.

```bash
cat <<'EOF' > kind-multi-node.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

This configuration tells kind to create:
- **1 control plane node** that runs the API server, etcd, scheduler, and controller-manager.
- **2 worker nodes** that run your application workloads.

---

## Step 2: Create the Cluster

```bash
kind create cluster --name kcna-lab --config kind-multi-node.yaml
```

Expected output:

```
Creating cluster "kcna-lab" ...
 ✓ Ensuring node image (kindest/node:v1.31.0) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kcna-lab"
```

This may take a few minutes depending on your machine and whether the node image needs to be pulled.

---

## Step 3: Verify the Cluster with kubectl cluster-info

```bash
kubectl cluster-info --context kind-kcna-lab
```

Expected output:

```
Kubernetes control plane is running at https://127.0.0.1:PORT
CoreDNS is running at https://127.0.0.1:PORT/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

This confirms that the API server and CoreDNS are running and accessible.

---

## Step 4: Explore the Nodes

### List all nodes

```bash
kubectl get nodes
```

Expected output:

```
NAME                     STATUS   ROLES           AGE   VERSION
kcna-lab-control-plane   Ready    control-plane   2m    v1.31.0
kcna-lab-worker          Ready    <none>          90s   v1.31.0
kcna-lab-worker2         Ready    <none>          90s   v1.31.0
```

All three nodes should be in **Ready** status.

### Get detailed node information with -o wide

```bash
kubectl get nodes -o wide
```

Expected output:

```
NAME                     STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME
kcna-lab-control-plane   Ready    control-plane   2m    v1.31.0   172.18.0.3    <none>        Debian GNU/Linux 12 (bookworm)   ...              containerd://1.7.x
kcna-lab-worker          Ready    <none>          90s   v1.31.0   172.18.0.2    <none>        Debian GNU/Linux 12 (bookworm)   ...              containerd://1.7.x
kcna-lab-worker2         Ready    <none>          90s   v1.31.0   172.18.0.4    <none>        Debian GNU/Linux 12 (bookworm)   ...              containerd://1.7.x
```

The `-o wide` flag shows additional columns: internal/external IPs, OS image, kernel version, and container runtime.

---

## Step 5: Verify the kubectl Context

```bash
kubectl config current-context
```

Expected output:

```
kind-kcna-lab
```

### List all available contexts

```bash
kubectl config get-contexts
```

This shows all clusters configured in your kubeconfig. The current context will be marked with `*`.

---

## Step 6: Confirm Docker Containers

Since kind runs Kubernetes nodes as Docker containers, you can see them with Docker:

```bash
docker ps --filter "name=kcna-lab"
```

Expected output:

```
CONTAINER ID   IMAGE                  COMMAND                  NAMES
abc123         kindest/node:v1.31.0   "/usr/local/bin/entr…"   kcna-lab-control-plane
def456         kindest/node:v1.31.0   "/usr/local/bin/entr…"   kcna-lab-worker
ghi789         kindest/node:v1.31.0   "/usr/local/bin/entr…"   kcna-lab-worker2
```

Each node in the kind cluster is a Docker container.

---

## Cleanup

When you are done with this lab, you can delete the cluster:

```bash
kind delete cluster --name kcna-lab
```

**Note:** If you plan to continue with the next labs, keep the cluster running.

Remove the config file:

```bash
rm kind-multi-node.yaml
```

---

## Key Takeaways

1. **kind** (Kubernetes in Docker) runs Kubernetes nodes as Docker containers, making it ideal for local development and learning.
2. A multi-node cluster can be created with a simple YAML configuration file.
3. `kubectl cluster-info` confirms the cluster is reachable and shows the API server endpoint.
4. `kubectl get nodes -o wide` provides detailed information about each node including IP addresses, OS, and container runtime.
5. The kubectl context determines which cluster your commands target.

---

## Next Steps

- Proceed to [Lab 02: Explore the Control Plane](02-explore-control-plane.md) to inspect the control plane components running on the cluster.
- Read about [Control Plane](../concepts/control-plane.md) concepts.
