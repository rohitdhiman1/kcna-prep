# Scheduling

**The kube-scheduler assigns Pods to nodes using a two-phase process (filtering + scoring). You can influence scheduling with node selectors, node affinity, taints/tolerations, and resource requests/limits. Resource settings also determine Pod QoS classes.**

---

## Table of Contents

- [How kube-scheduler Works](#how-kube-scheduler-works)
- [Scheduling Decision Flow Diagram](#scheduling-decision-flow-diagram)
- [Node Selectors](#node-selectors)
- [Node Affinity](#node-affinity)
- [Taints and Tolerations](#taints-and-tolerations)
- [Resource Requests and Limits](#resource-requests-and-limits)
- [Quality of Service (QoS) Classes](#quality-of-service-qos-classes)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## How kube-scheduler Works

The scheduler watches for newly created Pods that have no node assigned. For each Pod, it finds the best node using a two-phase process:

### Phase 1: Filtering (Predicates)

Eliminates nodes that **cannot** run the Pod. A node is filtered out if:

- It does not have enough CPU or memory (based on resource requests)
- It does not match the Pod's `nodeSelector` labels
- It does not satisfy the Pod's `nodeAffinity` rules (requiredDuringScheduling)
- It has a taint that the Pod does not tolerate
- The Pod requests a specific port that is already in use on the node
- The Pod's volume constraints cannot be met on the node

### Phase 2: Scoring (Priorities)

Ranks the remaining nodes to find the **best** fit. Scoring considers:

- **LeastRequestedPriority**: Prefers nodes with more available resources
- **BalancedResourceAllocation**: Prefers nodes where CPU and memory usage are balanced
- **NodeAffinityPriority**: Prefers nodes that match preferredDuringScheduling rules
- **InterPodAffinity**: Prefers nodes based on Pod affinity/anti-affinity preferences
- **ImageLocality**: Prefers nodes that already have the container image cached

The node with the highest total score wins.

---

## Scheduling Decision Flow Diagram

```
  Pod Created (no node assigned)
       |
       v
  +=============================================+
  |           KUBE-SCHEDULER                     |
  |                                              |
  |  PHASE 1: FILTERING                         |
  |  ==================                         |
  |                                              |
  |  All Nodes: [Node-1] [Node-2] [Node-3]      |
  |             [Node-4] [Node-5]                |
  |                                              |
  |  Filter: Sufficient resources?               |
  |    Node-3 removed (not enough memory)        |
  |                                              |
  |  Filter: Matches nodeSelector/affinity?      |
  |    Node-5 removed (missing label)            |
  |                                              |
  |  Filter: Tolerates taints?                   |
  |    Node-4 removed (has NoSchedule taint)     |
  |                                              |
  |  Remaining: [Node-1] [Node-2]                |
  |                                              |
  |  PHASE 2: SCORING                           |
  |  ================                           |
  |                                              |
  |  Score each remaining node:                  |
  |    Node-1: resource=70 + affinity=20 = 90    |
  |    Node-2: resource=50 + affinity=30 = 80    |
  |                                              |
  |  Winner: Node-1 (highest score: 90)          |
  +=============================================+
       |
       v
  Pod bound to Node-1
       |
       v
  kubelet on Node-1 pulls image, starts container
```

---

## Node Selectors

The simplest way to constrain a Pod to specific nodes. The Pod will only be scheduled on nodes with **all** matching labels.

### Label a Node

```bash
kubectl label nodes worker-1 disk=ssd
kubectl label nodes worker-2 disk=hdd
```

### Use nodeSelector in Pod Spec

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fast-app
spec:
  nodeSelector:
    disk: ssd                    # Only schedule on nodes labeled disk=ssd
  containers:
  - name: app
    image: my-app:v2
```

### Limitations of nodeSelector

- Only supports exact equality matching (`key=value`)
- Cannot express "OR" logic (e.g., ssd OR nvme)
- Cannot express preferences ("prefer ssd but allow hdd")
- For more expressive rules, use **Node Affinity**

---

## Node Affinity

Node affinity is a more powerful version of `nodeSelector`. It supports set-based operators and can express both **required** and **preferred** constraints.

### Required vs Preferred

```
  requiredDuringSchedulingIgnoredDuringExecution
  ================================================
  - Pod MUST be scheduled on a matching node
  - If no node matches, Pod stays in Pending
  - "IgnoredDuringExecution" means: if labels change
    after Pod is running, it is NOT evicted

  preferredDuringSchedulingIgnoredDuringExecution
  ================================================
  - Scheduler TRIES to place Pod on matching node
  - If no node matches, Pod is scheduled elsewhere
  - "weight" (1-100) controls how strongly the
    preference is considered in scoring
```

### Node Affinity YAML Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
            - us-east-1b
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: disk
            operator: In
            values:
            - ssd
      - weight: 20
        preference:
          matchExpressions:
          - key: gpu
            operator: Exists
  containers:
  - name: app
    image: my-app:v2
```

### Operators

| Operator       | Description                               | Example                        |
|----------------|-------------------------------------------|--------------------------------|
| `In`           | Label value is in the list                | `zone In [us-east-1a, 1b]`   |
| `NotIn`        | Label value is not in the list            | `env NotIn [production]`      |
| `Exists`       | Label key exists (any value)              | `gpu Exists`                  |
| `DoesNotExist` | Label key does not exist                  | `spot DoesNotExist`           |
| `Gt`           | Label value is greater than (numeric)     | `cores Gt 4`                  |
| `Lt`           | Label value is less than (numeric)        | `memory Lt 32`                |

---

## Taints and Tolerations

Taints are applied to **nodes** to repel Pods. Tolerations are applied to **Pods** to allow them onto tainted nodes.

```
  Taints and Tolerations
  =======================

  Think of it as:
  - Taint on Node = "Keep out!" sign on a room
  - Toleration on Pod = "I have permission to enter"

  +----------+                    +----------+
  |  Node     |                    |  Node     |
  | (tainted) |                    | (tainted) |
  |           |                    |           |
  | "No dogs  |                    | "No dogs  |
  |  allowed" |                    |  allowed" |
  +-----+----+                    +-----+----+
        |                               |
   +----+-----+                    +----+-----+
   | Pod (dog) |  REJECTED         | Pod (cat) |  ALLOWED
   | (no       |  xxxxxxxx         | (has      |  --------
   | toleration)|                   | toleration)|
   +-----------+                   +-----------+
```

### Applying a Taint

```bash
# Taint a node
kubectl taint nodes node1 key=value:NoSchedule

# Remove a taint (note the trailing -)
kubectl taint nodes node1 key=value:NoSchedule-
```

### Taint Effects

| Effect              | Description                                                      |
|---------------------|------------------------------------------------------------------|
| `NoSchedule`        | New Pods without toleration will NOT be scheduled on this node  |
| `PreferNoSchedule`  | Scheduler TRIES to avoid this node but will use it if necessary |
| `NoExecute`         | Existing Pods without toleration are EVICTED, new Pods rejected |

### Taint/Toleration YAML Example

```yaml
# Node taint (applied via kubectl)
# kubectl taint nodes node1 dedicated=gpu:NoSchedule

# Pod with toleration
apiVersion: v1
kind: Pod
metadata:
  name: gpu-app
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  containers:
  - name: app
    image: gpu-app:v1
```

### Toleration Operators

| Operator   | Description                                          |
|------------|------------------------------------------------------|
| `Equal`    | Key, value, AND effect must all match the taint     |
| `Exists`   | Key and effect must match (value is ignored)        |

### Special Cases

```yaml
# Tolerate ALL taints with a specific key
tolerations:
- key: "dedicated"
  operator: "Exists"           # Matches any value for this key

# Tolerate ALL taints on the node (use with caution)
tolerations:
- operator: "Exists"           # Empty key with Exists matches everything
```

### NoExecute and tolerationSeconds

When a node gets a `NoExecute` taint, existing Pods without a matching toleration are evicted immediately. You can delay eviction with `tolerationSeconds`:

```yaml
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300       # Stay on node for 5 minutes, then evict
```

### Built-in Taints

Kubernetes automatically adds taints to nodes:

| Taint                                    | When Applied                              |
|------------------------------------------|-------------------------------------------|
| `node.kubernetes.io/not-ready`          | Node is not ready                         |
| `node.kubernetes.io/unreachable`        | Node is unreachable                       |
| `node.kubernetes.io/memory-pressure`    | Node has memory pressure                  |
| `node.kubernetes.io/disk-pressure`      | Node has disk pressure                    |
| `node.kubernetes.io/pid-pressure`       | Node has too many processes               |
| `node.kubernetes.io/unschedulable`      | Node is cordoned                          |

---

## Resource Requests and Limits

### Requests

- **Requests** = guaranteed minimum resources for the container
- Used by the **scheduler** to find a node with enough available resources
- The node must have enough **allocatable** capacity to satisfy the request

### Limits

- **Limits** = maximum resources the container can use
- Enforced by the **kubelet** at runtime
- CPU: container is **throttled** if it exceeds the limit (compressible)
- Memory: container is **OOM-killed** if it exceeds the limit (incompressible)

```yaml
containers:
- name: app
  image: my-app:v2
  resources:
    requests:
      cpu: "250m"       # Guaranteed 0.25 CPU cores
      memory: "256Mi"   # Guaranteed 256 MiB
    limits:
      cpu: "500m"       # Max 0.5 CPU cores (throttled beyond)
      memory: "512Mi"   # Max 512 MiB (OOM-killed beyond)
```

### How Requests Affect Scheduling

```
  Node capacity: 4 CPU, 8Gi memory
  Already allocated: 2.5 CPU, 5Gi memory
  Available for scheduling: 1.5 CPU, 3Gi memory

  New Pod requests: 1 CPU, 2Gi memory
  +---> Fits! Pod is scheduled here.

  New Pod requests: 2 CPU, 4Gi memory
  +---> Does NOT fit. Node filtered out.
```

---

## Quality of Service (QoS) Classes

Kubernetes assigns a QoS class to each Pod based on its resource settings. QoS classes determine **eviction priority** when a node runs out of resources.

### Three QoS Classes

```
  QoS Classes (from highest to lowest priority)
  ===============================================

  +-------------------+----------------------------------------+
  |    Guaranteed      |  requests == limits for ALL containers |
  |    (highest        |  Both CPU and memory must be set       |
  |     priority)      |  Last to be evicted                    |
  +-------------------+----------------------------------------+
          |
          v
  +-------------------+----------------------------------------+
  |    Burstable       |  At least one container has a request  |
  |    (medium         |  or limit, but they are not all equal  |
  |     priority)      |  Evicted after BestEffort              |
  +-------------------+----------------------------------------+
          |
          v
  +-------------------+----------------------------------------+
  |    BestEffort      |  No requests or limits set for any     |
  |    (lowest         |  container in the Pod                  |
  |     priority)      |  First to be evicted                   |
  +-------------------+----------------------------------------+
```

### Examples

```yaml
# Guaranteed QoS -- requests == limits for ALL containers
containers:
- name: app
  resources:
    requests:
      cpu: "500m"
      memory: "256Mi"
    limits:
      cpu: "500m"          # Same as request
      memory: "256Mi"      # Same as request

# Burstable QoS -- requests != limits, or only some set
containers:
- name: app
  resources:
    requests:
      cpu: "250m"
      memory: "128Mi"
    limits:
      cpu: "500m"          # Different from request
      memory: "256Mi"      # Different from request

# BestEffort QoS -- no requests or limits at all
containers:
- name: app
  image: my-app:v2
  # No resources section
```

### Eviction Order Under Node Pressure

```
  Node running low on memory:
  ============================

  Step 1: Evict BestEffort Pods first
          (no resource guarantees, expendable)
              |
              v
  Step 2: Evict Burstable Pods exceeding their requests
          (using more than they asked for)
              |
              v
  Step 3: Evict Guaranteed Pods last (only if absolutely necessary)
          (these have firm resource guarantees)
```

---

## What to Remember for the Exam

1. **The scheduler uses two phases**: Filtering (eliminate unsuitable nodes) and Scoring (rank remaining nodes). The highest-scoring node wins.

2. **nodeSelector** is the simplest way to constrain Pods to nodes. It uses exact label matching only.

3. **Node affinity** has two types: `requiredDuringSchedulingIgnoredDuringExecution` (hard constraint, Pod stays Pending if no match) and `preferredDuringSchedulingIgnoredDuringExecution` (soft constraint, scheduler tries but will schedule elsewhere).

4. **"IgnoredDuringExecution"** means if node labels change after the Pod is running, the Pod is NOT evicted. It only affects scheduling decisions.

5. **Taints go on nodes, tolerations go on Pods**. A toleration does not guarantee scheduling on the tainted node -- it merely allows it. Use nodeSelector or affinity to attract Pods.

6. **Three taint effects**: `NoSchedule` (hard block), `PreferNoSchedule` (soft block), `NoExecute` (evicts existing Pods too).

7. **Resource requests** are used by the scheduler for placement. **Resource limits** are enforced at runtime by the kubelet.

8. **CPU is compressible** (throttled at limit). **Memory is incompressible** (OOM-killed at limit).

9. **Three QoS classes**: Guaranteed (requests == limits for all containers), Burstable (at least one request or limit set, but not all equal), BestEffort (no requests or limits).

10. **Eviction order**: BestEffort first, then Burstable, then Guaranteed last. Set appropriate resource requests and limits to control eviction priority.

11. **Taints and tolerations** are commonly used for: dedicated nodes (GPU, special hardware), master node protection, and node maintenance (NoExecute to drain).

12. **Control plane nodes** are tainted by default with `node-role.kubernetes.io/control-plane:NoSchedule` to prevent user workloads from running on them.
