# Lab 03: Inspect Worker Nodes

## Objective

Examine worker nodes in a Kubernetes cluster by listing them, describing their conditions and capacity, understanding allocatable resources, and inspecting the kubelet.

## Prerequisites

- A running multi-node Kubernetes cluster (from [Lab 01](01-cluster-setup.md)).
- `kubectl` installed and configured.
- Read the concept file: [Worker Nodes](../concepts/worker-nodes.md)

---

## Step 1: List All Nodes

```bash
kubectl get nodes
```

Expected output:

```
NAME                     STATUS   ROLES           AGE   VERSION
kcna-lab-control-plane   Ready    control-plane   15m   v1.31.0
kcna-lab-worker          Ready    <none>          14m   v1.31.0
kcna-lab-worker2         Ready    <none>          14m   v1.31.0
```

Worker nodes have no role label by default (shown as `<none>`), while the control plane node shows `control-plane`.

### Get extended information

```bash
kubectl get nodes -o wide
```

Note the **INTERNAL-IP**, **OS-IMAGE**, **KERNEL-VERSION**, and **CONTAINER-RUNTIME** columns.

---

## Step 2: Describe a Worker Node

```bash
kubectl describe node kcna-lab-worker
```

This produces a large output. Work through the following important sections.

### Node Conditions

Look for the `Conditions:` section:

```
Conditions:
  Type                 Status  Reason                       Message
  ----                 ------  ------                       -------
  MemoryPressure       False   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    KubeletReady                 kubelet is posting ready status
```

- **Ready=True** means the node is healthy and can accept pods.
- If any pressure condition is True, the node is under stress and may evict pods.

### Capacity vs. Allocatable

Look for the `Capacity:` and `Allocatable:` sections:

```
Capacity:
  cpu:                4
  ephemeral-storage:  61202244Ki
  memory:             8145260Ki
  pods:               110
Allocatable:
  cpu:                4
  ephemeral-storage:  61202244Ki
  memory:             8042860Ki
  pods:               110
```

- **Capacity** is the total resources on the node.
- **Allocatable** is what is available for pods (capacity minus resources reserved for the system and kubelet).
- The difference between capacity and allocatable memory is reserved for system daemons.

### Non-terminated Pods

Look for the `Non-terminated Pods:` section to see which pods are running on this node:

```
Non-terminated Pods:          (2 in total)
  Namespace    Name              CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------    ----              ------------  ----------  ---------------  -------------
  kube-system  kindnet-xxxxx     100m (2%)     100m (2%)   50Mi (0%)        50Mi (0%)
  kube-system  kube-proxy-xxxxx  0 (0%)        0 (0%)      0 (0%)           0 (0%)
```

### Allocated Resources

Look for the `Allocated resources:` section:

```
Allocated resources:
  Resource           Requests    Limits
  --------           --------    ------
  cpu                100m (2%)   100m (2%)
  memory             50Mi (0%)   50Mi (0%)
  ephemeral-storage  0 (0%)      0 (0%)
```

This shows total resources requested and limited by all pods on this node.

---

## Step 3: Check Node Conditions Programmatically

### Use JSONPath to extract specific conditions

```bash
kubectl get node kcna-lab-worker -o jsonpath='{.status.conditions[*].type}'
```

Expected output:

```
MemoryPressure DiskPressure PIDPressure Ready
```

### Get the Ready status of all nodes

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'
```

Expected output:

```
kcna-lab-control-plane	True
kcna-lab-worker	True
kcna-lab-worker2	True
```

---

## Step 4: View Node Labels

```bash
kubectl get nodes --show-labels
```

This shows all labels applied to each node. Common labels include:
- `kubernetes.io/hostname` — the node's hostname.
- `kubernetes.io/os` — the operating system (linux).
- `kubernetes.io/arch` — the CPU architecture (amd64, arm64).
- `node-role.kubernetes.io/control-plane` — marks the control plane node.

### Filter nodes by label

```bash
kubectl get nodes -l 'node-role.kubernetes.io/control-plane'
```

This returns only the control plane node.

---

## Step 5: Examine the Kubelet

The kubelet is the agent running on every node. In a kind cluster, it runs as a process inside the Docker container.

### Check kubelet status

```bash
docker exec kcna-lab-worker systemctl status kubelet
```

Expected output (truncated):

```
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded
     Active: active (running) since ...
```

### View kubelet configuration

```bash
docker exec kcna-lab-worker cat /var/lib/kubelet/config.yaml
```

Key settings to look for:
- `clusterDNS` — the DNS server IP used by pods.
- `clusterDomain` — the domain suffix (usually `cluster.local`).
- `staticPodPath` — the directory where static pod manifests are read from.
- `maxPods` — maximum number of pods this node can run.

### View recent kubelet logs

```bash
docker exec kcna-lab-worker journalctl -u kubelet --no-pager --lines=20
```

---

## Step 6: Examine kube-proxy on the Worker Node

kube-proxy runs on every node as a DaemonSet and handles service networking.

```bash
kubectl get daemonset kube-proxy -n kube-system
```

Expected output:

```
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   3         3         3       3             3           kubernetes.io/os=linux   15m
```

### View kube-proxy logs on a specific node

```bash
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=10
```

---

## Step 7: Compare Worker Node and Control Plane Node

Describe the control plane node and compare it to the worker node:

```bash
kubectl describe node kcna-lab-control-plane
```

Differences to note:
- The control plane node has a **taint**: `node-role.kubernetes.io/control-plane:NoSchedule`. This prevents regular pods from being scheduled on it.
- The control plane node runs the static pods (etcd, kube-apiserver, etc.) in addition to kube-proxy and the CNI plugin.
- Worker nodes only run kube-proxy, the CNI plugin, and your application pods.

---

## Cleanup

No resources to clean up in this lab.

---

## Key Takeaways

1. `kubectl get nodes -o wide` shows IPs, OS, kernel, and container runtime information.
2. `kubectl describe node` reveals conditions, capacity, allocatable resources, running pods, and allocated resources.
3. **Capacity** is total resources; **Allocatable** is what is available for pods after system reservations.
4. Node conditions (Ready, MemoryPressure, DiskPressure, PIDPressure) indicate node health.
5. The **kubelet** is the primary agent on each node. It registers the node, manages pod lifecycle, and reports status to the API server.
6. **kube-proxy** runs on every node as a DaemonSet and manages iptables/IPVS rules for service networking.
7. Control plane nodes have taints to prevent regular workloads from being scheduled on them.

---

## Next Steps

- Proceed to [Lab 04: kubectl Basics](04-kubectl-basics.md) to practice essential kubectl commands.
- Read about [Kubernetes API](../concepts/kubernetes-api.md) concepts.
