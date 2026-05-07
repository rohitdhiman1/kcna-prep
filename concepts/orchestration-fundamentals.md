# Orchestration Fundamentals

> Container orchestration automates the deployment, scaling, networking, and management of containerized applications across a cluster of machines.

---

## Why Container Orchestration Exists

Running a single container on a single machine is straightforward. Running hundreds or thousands of containers across dozens of machines is not. Orchestration solves the problems that arise at scale.

### What Happens Without Orchestration

Imagine managing containers manually:

```
  Developer's Nightmare (Manual Container Management)

  Server 1          Server 2          Server 3
  ┌──────────┐      ┌──────────┐      ┌──────────┐
  │ app-v1   │      │ app-v1   │      │ (empty)  │
  │ app-v1   │      │ db       │      │          │
  │ app-v1   │      │ cache    │      │          │
  │ (full!)  │      │ app-v1   │      │          │
  └──────────┘      └──────────┘      └──────────┘

  Problems:
  - Server 1 is overloaded, Server 3 is idle
  - app-v1 crashed on Server 2 — who restarts it?
  - How do you update all instances to app-v2?
  - How does traffic find each app instance?
  - A new server joins — how does it get work?
```

Manual management fails because:

1. **No automatic scheduling** — You must decide which server runs which container.
2. **No self-healing** — If a container crashes at 3 AM, nobody restarts it until morning.
3. **No scaling** — Traffic spikes require manual intervention to add instances.
4. **No service discovery** — Containers don't know how to find each other.
5. **No rolling updates** — Deploying a new version means downtime or risky manual steps.
6. **No load balancing** — Traffic is not distributed evenly across instances.

---

## What Container Orchestration Solves

An orchestrator handles six core responsibilities:

```
  ┌──────────────────────────────────────────────────────────────┐
  │                  CONTAINER ORCHESTRATOR                      │
  │                                                              │
  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
  │  │  Scheduling  │  │   Scaling    │  │   Self-Healing    │  │
  │  │              │  │              │  │                   │  │
  │  │ Place pods   │  │ Scale up/    │  │ Restart crashed   │  │
  │  │ on nodes     │  │ down based   │  │ containers,       │  │
  │  │ based on     │  │ on demand    │  │ replace failed    │  │
  │  │ resources    │  │ or metrics   │  │ nodes             │  │
  │  └──────────────┘  └──────────────┘  └───────────────────┘  │
  │                                                              │
  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
  │  │    Load      │  │   Rolling    │  │    Service        │  │
  │  │  Balancing   │  │   Updates    │  │   Discovery       │  │
  │  │              │  │              │  │                   │  │
  │  │ Distribute   │  │ Update pods  │  │ Pods and services │  │
  │  │ traffic      │  │ gradually,   │  │ find each other   │  │
  │  │ across       │  │ rollback if  │  │ via DNS and       │  │
  │  │ instances    │  │ errors occur │  │ internal IPs      │  │
  │  └──────────────┘  └──────────────┘  └───────────────────┘  │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

### 1. Scheduling

The orchestrator decides **where** each container runs. It evaluates:

- Available CPU and memory on each node
- Affinity/anti-affinity rules (co-locate or spread)
- Taints and tolerations (node restrictions)
- Resource requests and limits declared by the pod

### 2. Scaling

- **Horizontal scaling** — Add or remove pod replicas based on CPU, memory, or custom metrics.
- **Vertical scaling** — Adjust CPU/memory allocated to existing pods.
- **Cluster scaling** — Add or remove nodes from the cluster when needed.

### 3. Self-Healing

- Automatically **restarts** containers that crash.
- **Replaces** pods on nodes that become unhealthy.
- **Kills** containers that fail health checks (liveness probes).
- Does **not route traffic** to pods that aren't ready (readiness probes).

### 4. Load Balancing

- Distributes incoming traffic across all healthy pod replicas.
- Kubernetes Services provide a stable virtual IP (ClusterIP) backed by multiple pods.
- External load balancers can be provisioned automatically (LoadBalancer type).

### 5. Rolling Updates and Rollbacks

- Updates pods incrementally (e.g., replace 25% at a time).
- Monitors new pods for health before continuing.
- If something goes wrong, rolls back to the previous version automatically.

### 6. Service Discovery

- Every Service gets a DNS name (e.g., `my-service.my-namespace.svc.cluster.local`).
- Pods discover other services via DNS — no hardcoded IPs.
- CoreDNS provides the cluster DNS service.

---

## Orchestration in Action

```
  Before Orchestration              After Orchestration
  (Manual)                          (Automated)

  ┌────────┐                        ┌─────────────────────┐
  │        │ "Deploy 5 copies       │    Orchestrator     │
  │  You   │  of app-v2 to         │    ┌─────────────┐  │
  │        │  3 servers, set up     │    │ Scheduler   │  │
  │  😰   │  load balancing,       │    │ Controller  │  │
  │        │  monitor health,       │    │ DNS         │  │
  │        │  handle failures..."   │    └──────┬──────┘  │
  └────────┘                        │           │         │
                                    │     ┌─────┴─────┐   │
                                    │     ▼     ▼     ▼   │
                                    │   Node1 Node2 Node3 │
                                    │   [app] [app] [app]  │
                                    │   [app]       [app]  │
                                    └─────────────────────┘

  You say:                          Orchestrator does:
  "I want 5 replicas"              Schedule across nodes
  "Use image app:v2"               Rolling update from v1
  "Expose on port 80"              Load balance + DNS entry
  (that's it)                       Self-heal if any crash
```

---

## Comparison: Kubernetes vs Docker Swarm vs Nomad

| Feature | Kubernetes | Docker Swarm | HashiCorp Nomad |
|---------|-----------|-------------|----------------|
| **Maintained by** | CNCF (community) | Docker Inc (Mirantis) | HashiCorp |
| **Architecture** | Control plane + workers | Manager + worker nodes | Server + client agents |
| **Complexity** | High (most features) | Low (simpler) | Medium |
| **Scaling** | Excellent (tested at 5,000+ nodes) | Good (smaller scale) | Excellent |
| **Ecosystem** | Massive (CNCF landscape) | Limited | Integrates with HashiCorp stack |
| **Workloads** | Containers (pods) | Containers (tasks) | Containers, VMs, binaries |
| **Networking** | CNI plugins (Calico, Cilium) | Built-in overlay | CNI or built-in |
| **Service mesh** | Istio, Linkerd, etc. | Not built-in | Consul Connect |
| **Industry adoption** | De facto standard | Declining | Growing (niche) |
| **KCNA focus** | Primary focus | Awareness only | Awareness only |

### Key Takeaway

**Kubernetes is the industry standard** for container orchestration. Docker Swarm is simpler but lacks the ecosystem. Nomad is flexible (runs non-container workloads too) but has a smaller community. For the KCNA exam, Kubernetes is all you need to know deeply — the others are for awareness.

---

## The Declarative Model

Orchestrators use a **declarative** approach: you describe the **desired state**, and the system works to make it real.

```
  Declarative Model

  ┌─────────────┐     ┌──────────────────┐     ┌─────────────┐
  │  You write  │     │  Control Loop    │     │  Cluster    │
  │  YAML:      │────►│                  │────►│  State:     │
  │             │     │  Desired State   │     │             │
  │ replicas: 3 │     │  vs              │     │  3 pods     │
  │ image: v2   │     │  Current State   │     │  running    │
  │             │     │  = Reconcile     │     │  image v2   │
  └─────────────┘     └──────────────────┘     └─────────────┘

  If a pod crashes (current state = 2 pods),
  the controller detects the drift and creates a new pod (back to 3).
```

This is fundamentally different from the **imperative** approach ("run this container on this server"), which requires you to manage every step manually.

---

## Key Exam Points

- Container orchestration automates scheduling, scaling, self-healing, load balancing, rolling updates, and service discovery.
- Without orchestration, managing containers at scale is error-prone and unsustainable.
- Kubernetes uses a **declarative model**: you define desired state, the system reconciles.
- **Kubernetes is the industry standard**. Docker Swarm and Nomad exist but are not the focus of the KCNA.
- The orchestrator continuously monitors and reconciles — this is the **control loop** pattern.
- Orchestration happens at the **cluster level**, not the individual machine level.

---

## What to Remember for the Exam

1. **Six responsibilities of orchestration**: scheduling, scaling, self-healing, load balancing, rolling updates, service discovery — be able to name all six.
2. **Declarative vs imperative**: Kubernetes is declarative. You describe what you want, not how to do it.
3. **Kubernetes vs alternatives**: Know that Docker Swarm and Nomad exist. Kubernetes dominates. Nomad can run non-container workloads (a differentiator).
4. **Self-healing**: Kubernetes restarts crashed containers, replaces pods on dead nodes, and stops routing traffic to unhealthy pods. This is automatic, not manual.
5. **Control loop**: The core pattern — observe current state, compare to desired state, take action to reconcile.
