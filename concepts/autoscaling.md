# Autoscaling in Kubernetes

> Kubernetes can automatically adjust compute resources at the pod level and the cluster level to match workload demand.

---

## Overview

Kubernetes supports multiple levels of autoscaling:

```
  ┌───────────────────────────────────────────────────────┐
  │              Kubernetes Autoscaling Stack               │
  │                                                         │
  │  ┌──────────────────────────────────────────────────┐  │
  │  │  Cluster Autoscaler                              │  │
  │  │  Adds/removes NODES from the cluster             │  │
  │  └───────────────────────┬──────────────────────────┘  │
  │                          │                              │
  │  ┌───────────────────────┴──────────────────────────┐  │
  │  │  Horizontal Pod Autoscaler (HPA)                 │  │
  │  │  Adds/removes POD REPLICAS                       │  │
  │  ├──────────────────────────────────────────────────┤  │
  │  │  Vertical Pod Autoscaler (VPA)                   │  │
  │  │  Adjusts pod RESOURCE REQUESTS/LIMITS            │  │
  │  ├──────────────────────────────────────────────────┤  │
  │  │  KEDA                                            │  │
  │  │  Event-driven autoscaling (queues, streams, etc) │  │
  │  └──────────────────────────────────────────────────┘  │
  │                                                         │
  └───────────────────────────────────────────────────────┘
```

---

## Horizontal Pod Autoscaler (HPA)

The HPA automatically scales the **number of pod replicas** in a Deployment, ReplicaSet, or StatefulSet based on observed metrics.

### How It Works

```
                       ┌──────────────┐
                       │ metrics-     │
                       │ server       │
                       │ (or custom   │
                       │  metrics API)│
                       └──────┬───────┘
                              │ metrics
                              ▼
  ┌──────────────┐     ┌──────────────┐
  │              │     │     HPA      │
  │  Deployment  │◄────┤  Controller  │
  │  (replicas)  │     │              │
  │              │     │ Checks every │
  └──┬──┬──┬──┬──┘     │ 15s (default)│
     │  │  │  │        └──────────────┘
     ▼  ▼  ▼  ▼
   ┌──┐┌──┐┌──┐┌──┐
   │P1││P2││P3││P4│  <-- replicas scale up/down
   └──┘└──┘└──┘└──┘
```

### Scaling Algorithm

The HPA uses this formula:

```
  desiredReplicas = ceil( currentReplicas * (currentMetricValue / targetMetricValue) )
```

Example:
- Current replicas: 3
- Current CPU usage: 90%
- Target CPU: 50%
- Desired = ceil(3 * 90/50) = ceil(5.4) = **6 replicas**

### Supported Metrics

| Metric Type | Source | Example |
|-------------|--------|---------|
| **Resource** | metrics-server | CPU utilization, memory utilization |
| **Pods** | Custom metrics API | Requests per second per pod |
| **Object** | Custom metrics API | Queue depth of a specific object |
| **External** | External metrics API | Cloud monitoring metric (e.g., SQS queue length) |

### Key Configuration

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### Important Details

- **metrics-server** must be installed for CPU/memory metrics.
- Default check interval: every **15 seconds**.
- Default cooldown: **3 minutes** for scale-up, **5 minutes** for scale-down.
- Pods **must have resource requests** defined for CPU/memory-based scaling.
- HPA and VPA should generally **not be used together** on the same metric.

---

## Vertical Pod Autoscaler (VPA)

The VPA automatically adjusts **resource requests and limits** for containers in a pod.

### How It Works

```
  ┌──────────────┐     ┌──────────────────────┐
  │  VPA         │     │   Pod (before VPA)    │
  │  Controller  │     │                       │
  │              │     │   requests:           │
  │  Monitors    │     │     cpu: 100m         │
  │  actual      ├────►│     memory: 128Mi     │
  │  resource    │     │                       │
  │  usage over  │     └───────────────────────┘
  │  time        │                │
  │              │                │ VPA updates
  └──────────────┘                ▼
                       ┌───────────────────────┐
                       │   Pod (after VPA)      │
                       │                       │
                       │   requests:           │
                       │     cpu: 250m         │
                       │     memory: 512Mi     │
                       │                       │
                       └───────────────────────┘
```

### VPA Modes

| Mode | Behavior |
|------|----------|
| **Off** | VPA only provides recommendations; does not change pods. |
| **Initial** | VPA sets resource requests at pod creation only. |
| **Auto** | VPA updates resource requests on running pods (may cause restarts). |

### Key Points
- VPA **evicts and recreates pods** to apply new resource values (cannot update in-place currently).
- VPA is useful when you do not know the right resource requests for a workload.
- VPA and HPA on **CPU/memory together** can cause conflicts. Use HPA for scaling replicas and VPA for right-sizing, but on different metrics.

---

## Cluster Autoscaler

The Cluster Autoscaler automatically adjusts the **number of nodes** in a cluster.

### How It Works

```
  Scenario 1: Scale UP                Scenario 2: Scale DOWN
  ─────────────────────                ──────────────────────

  Pod pending (unschedulable)          Node underutilized
  No node has enough resources         All pods can fit on other nodes

  ┌──────┐ ┌──────┐                    ┌──────┐ ┌──────┐ ┌──────┐
  │Node 1│ │Node 2│ Pod?               │Node 1│ │Node 2│ │Node 3│
  │[full]│ │[full]│ (pending)          │[busy]│ │[busy]│ │[idle]│
  └──────┘ └──────┘                    └──────┘ └──────┘ └──┬───┘
                                                             │
        Cluster Autoscaler                  Cluster Autoscaler│
        detects pending pod                 detects idle node  │
              │                                    │           │
              ▼                                    ▼           │
  ┌──────┐ ┌──────┐ ┌──────┐          ┌──────┐ ┌──────┐      │
  │Node 1│ │Node 2│ │Node 3│          │Node 1│ │Node 2│  removed
  │[full]│ │[full]│ │[new] │          │[busy]│ │[busy]│
  └──────┘ └──────┘ └──────┘          └──────┘ └──────┘
                      Pod!
```

### Scale-Up Trigger
- A pod is **Pending** because no node has sufficient resources.
- Cluster Autoscaler requests a new node from the cloud provider.
- Once the node is ready, the scheduler places the pod.

### Scale-Down Trigger
- A node is **underutilized** (below threshold, typically 50%).
- All pods on the node can be rescheduled to other nodes.
- The node is drained and removed.

### Key Points
- Works with **cloud providers** (AWS, GCP, Azure) and their node groups / instance groups.
- Does **not** work with bare-metal clusters (use other tools like Karpenter for AWS).
- Respects **PodDisruptionBudgets** when draining nodes.
- Default scale-down delay: **10 minutes** of underutilization.

---

## KEDA (Kubernetes Event-Driven Autoscaling)

KEDA extends Kubernetes autoscaling to support **event-driven** scaling based on external event sources.

### How It Works

```
  ┌─────────────┐     ┌─────────────┐     ┌──────────────────┐
  │ Event Source │     │    KEDA     │     │   Deployment     │
  │             │     │             │     │                  │
  │ - Kafka     │────►│ ScaledObject│────►│  Replicas: 0→N  │
  │ - RabbitMQ  │     │             │     │                  │
  │ - AWS SQS   │     │ Reads event │     │  Scales based on │
  │ - Redis     │     │ source and  │     │  event count     │
  │ - Cron      │     │ scales pods │     │                  │
  │ - HTTP      │     │             │     │                  │
  │ - Prometheus│     │             │     │                  │
  └─────────────┘     └─────────────┘     └──────────────────┘
```

### Key Features

- **Scale to zero**: Unlike HPA, KEDA can scale deployments down to **zero replicas** when there are no events. This saves resources.
- **50+ scalers**: Supports Kafka, RabbitMQ, AWS SQS, Azure Queue, Redis, Prometheus, Cron, HTTP, and many more.
- **Works alongside HPA**: KEDA creates and manages HPA resources under the hood.
- **ScaledObject**: The custom resource that defines what to scale and what event source to watch.

### KEDA vs HPA

| Feature | HPA | KEDA |
|---------|-----|------|
| Scale to zero | No (minimum 1 replica) | Yes |
| Metric sources | CPU, memory, custom metrics | 50+ event sources |
| Event-driven | Not natively | Yes, primary purpose |
| Setup complexity | Simple | Requires KEDA installation |
| Use case | Steady traffic, resource-based | Event-driven workloads, batch processing |

### Example Use Cases
- Scale workers based on messages in a **Kafka topic** or **RabbitMQ queue**.
- Scale a web app based on **HTTP request rate**.
- Scale batch jobs based on items in an **AWS SQS queue**.
- Run a CronJob-like workload with **cron-based scaling**.

---

## How metrics-server Feeds HPA

The **metrics-server** is the default source of resource metrics for HPA.

```
  ┌─────────┐   ┌─────────┐   ┌─────────┐
  │  Node 1 │   │  Node 2 │   │  Node 3 │
  │         │   │         │   │         │
  │ kubelet │   │ kubelet │   │ kubelet │
  │ (cAdvisor)  │ (cAdvisor)  │ (cAdvisor)
  └────┬────┘   └────┬────┘   └────┬────┘
       │             │             │
       └──────┬──────┴──────┬──────┘
              │             │
              ▼             │
       ┌──────────────┐     │
       │ metrics-     │◄────┘
       │ server       │
       │              │
       │ Aggregates   │
       │ CPU & memory │
       │ from all     │
       │ kubelets     │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │ Metrics API  │
       │ (metrics.    │
       │  k8s.io)     │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │     HPA      │
       │  Controller  │
       │              │
       │  Queries     │
       │  metrics API │
       │  every 15s   │
       └──────────────┘
```

Flow:
1. **kubelet** on each node collects resource metrics from containers via **cAdvisor** (built in).
2. **metrics-server** scrapes all kubelets and aggregates the data.
3. metrics-server exposes data via the **Metrics API** (`metrics.k8s.io`).
4. **HPA controller** queries the Metrics API every 15 seconds (configurable).
5. HPA calculates desired replicas and updates the Deployment.

Note: metrics-server stores only **latest values** (no history). For historical metrics, use Prometheus.

---

## Summary Table

| Autoscaler | What It Scales | Based On | Key Detail |
|------------|---------------|----------|------------|
| **HPA** | Pod replicas (horizontal) | CPU, memory, custom metrics | Needs metrics-server; min 1 replica |
| **VPA** | Pod resource requests/limits | Historical resource usage | Evicts pods to apply changes |
| **Cluster Autoscaler** | Cluster nodes | Pending pods / underutilized nodes | Cloud provider integration required |
| **KEDA** | Pod replicas (event-driven) | External event sources (50+) | Can scale to zero; extends HPA |

---

## What to Remember for the Exam

1. **HPA** scales the number of **pod replicas** based on metrics. Most common: CPU utilization. Needs **metrics-server** installed.
2. **VPA** adjusts **resource requests and limits** for pods. Useful for right-sizing. Evicts pods to apply changes.
3. **Cluster Autoscaler** adds/removes **nodes**. Triggers on pending pods (scale up) or underutilized nodes (scale down).
4. **KEDA** provides **event-driven** autoscaling. Can scale to **zero replicas**. Supports 50+ event sources.
5. HPA requires pods to have **resource requests** defined.
6. **metrics-server** collects CPU and memory from kubelets and exposes via Metrics API.
7. HPA and VPA should not both target the same metric (e.g., both on CPU).
8. Cluster Autoscaler requires **cloud provider** integration; it does not apply to bare-metal.
9. KEDA creates HPA resources internally but extends them with event-driven capabilities.
