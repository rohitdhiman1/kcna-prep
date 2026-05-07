# KCNA Cheat Sheet

> Quick-review reference for last-minute revision. Covers all 5 exam domains.

---

## Domain 1 — Kubernetes Fundamentals (46%)

### Cluster Architecture

```
Control Plane                    Worker Node
┌──────────────────────┐        ┌──────────────────────┐
│ kube-apiserver       │◄──────►│ kubelet              │
│ etcd                 │        │ kube-proxy           │
│ kube-scheduler       │        │ container runtime    │
│ kube-controller-mgr  │        │   (containerd)       │
│ cloud-controller-mgr │        └──────────────────────┘
└──────────────────────┘
```

- **kube-apiserver** — Central hub; all components talk through it; only component that talks to etcd
- **etcd** — Key-value store; single source of truth for all cluster state
- **kube-scheduler** — Assigns Pods to Nodes (filter → score → bind)
- **kube-controller-manager** — Runs reconciliation loops (ReplicaSet, Node, Endpoint controllers)
- **cloud-controller-manager** — Integrates with cloud provider APIs (optional)
- **kubelet** — Node agent; manages Pod lifecycle; reports status
- **kube-proxy** — Programs iptables/IPVS rules for Service routing
- **Container runtime** — Runs containers via CRI (containerd, CRI-O)

### Core Resources

| Resource | Purpose |
|----------|---------|
| Pod | Smallest deployable unit; 1+ containers sharing network/storage |
| ReplicaSet | Maintains desired number of Pod replicas |
| Deployment | Manages ReplicaSets; provides rolling updates and rollbacks |
| Service | Stable network endpoint for a set of Pods |
| Namespace | Logical isolation of resources within a cluster |
| ConfigMap | Non-sensitive configuration data (key-value) |
| Secret | Sensitive data (base64-encoded); tokens, passwords, certs |

### Service Types

| Type | Access | Use Case |
|------|--------|----------|
| ClusterIP | Internal only (default) | Service-to-service communication |
| NodePort | External via `<NodeIP>:<Port>` | Development / testing |
| LoadBalancer | External via cloud LB | Production external access |
| ExternalName | DNS CNAME redirect | Pointing to external services |

### Workload Resources

| Resource | Behavior |
|----------|----------|
| Job | Run to completion, then stop |
| CronJob | Schedule Jobs on a cron schedule |
| DaemonSet | One Pod per node (logging, monitoring agents) |
| StatefulSet | Stable identity, ordered deploy, persistent storage |

### Scheduling

- **Taints** (on nodes) + **Tolerations** (on Pods) = repel Pods without matching tolerations
- **Node Affinity** = attract Pods to nodes based on labels
- **Resource Requests** = guaranteed minimum (used for scheduling)
- **Resource Limits** = maximum allowed (CPU throttled, memory OOM-killed)

### Probes

| Probe | Purpose | On Failure |
|-------|---------|------------|
| Liveness | Is the container stuck? | Restart container |
| Readiness | Is the container ready for traffic? | Remove from Service endpoints |
| Startup | Has the container started? | Block liveness/readiness checks |

### Labels vs Annotations

- **Labels** — Identify and select resources (used by selectors, Services, ReplicaSets)
- **Annotations** — Non-identifying metadata (build info, tool config, contact info)

---

## Domain 2 — Container Orchestration (22%)

### Container Runtimes & Standards

| Standard | Full Name | Defines |
|----------|-----------|---------|
| OCI | Open Container Initiative | Image format + runtime spec |
| CRI | Container Runtime Interface | K8s ↔ runtime communication |
| CNI | Container Network Interface | Pod networking |
| CSI | Container Storage Interface | Storage provisioning |

- **containerd** — Default CRI runtime (CNCF graduated)
- **CRI-O** — Lightweight CRI runtime built for Kubernetes
- **runc** — Low-level OCI runtime (actually runs containers)

### Networking

- Every Pod gets its own IP
- Pod-to-pod communication across nodes works without NAT
- **CNI plugins**: Calico, Cilium, Flannel, Weave
- **CoreDNS**: Cluster DNS; resolves Service names to ClusterIPs
- **Ingress**: L7 HTTP/HTTPS routing; host/path-based rules; needs an Ingress Controller

### Security

- **RBAC**: Role + RoleBinding (namespaced) | ClusterRole + ClusterRoleBinding (cluster-wide)
- **Verbs**: get, list, watch, create, update, patch, delete
- **NetworkPolicy**: Allow/deny traffic between Pods; default = allow all
- **Pod Security Standards**: Privileged, Baseline, Restricted

### Service Mesh

- **Pattern**: Sidecar proxy injected into every Pod
- **Provides**: mTLS, traffic management, observability, retries
- **Implementations**: Istio (Envoy proxy), Linkerd
- **SMI**: Service Mesh Interface — standard API for service meshes

### Storage

```
StorageClass ──(dynamic provisioning)──► PersistentVolume (PV)
                                              ▲
                                              │ bound
                                              ▼
                                    PersistentVolumeClaim (PVC)
                                              ▲
                                              │ mounted
                                              ▼
                                            Pod
```

- **PV** — Cluster-level storage resource (provisioned by admin or dynamically)
- **PVC** — User's request for storage (size, access mode, storage class)
- **Access Modes**: ReadWriteOnce (RWO), ReadOnlyMany (ROX), ReadWriteMany (RWX)

---

## Domain 3 — Cloud Native Architecture (16%)

### Cloud Native Characteristics

Scalable | Resilient | Observable | Manageable | Loosely Coupled

### 12-Factor App (Key Factors)

1. **Codebase** — One repo, many deploys
2. **Dependencies** — Explicitly declare and isolate
3. **Config** — Store in environment variables
4. **Backing services** — Treat as attached resources
5. **Processes** — Stateless; share-nothing
6. **Port binding** — Export services via port binding
7. **Disposability** — Fast startup, graceful shutdown

### Microservices vs Monoliths

| | Monolith | Microservices |
|---|----------|---------------|
| Deploy | All or nothing | Independent per service |
| Scale | Entire app | Per service |
| Complexity | Low (initially) | Higher (networking, debugging) |
| Technology | Single stack | Polyglot |

### Autoscaling

| Scaler | What It Scales | Based On |
|--------|---------------|----------|
| HPA | Pod replicas | CPU, memory, custom metrics |
| VPA | Pod resource requests | Historical usage |
| Cluster Autoscaler | Nodes | Pending pods / underutilized nodes |
| KEDA | Pod replicas | Event-driven (queues, streams) |

### CNCF Maturity Levels

```
Sandbox ──► Incubating ──► Graduated
(early)     (growing)      (production-ready)
```

- **Graduated examples**: Kubernetes, Prometheus, Envoy, containerd, Fluentd, Helm, etcd
- **TOC** — Technical Oversight Committee; oversees project maturity

### Roles & SLx

- **SLI** (Indicator) — Metric measuring service (e.g., latency p99)
- **SLO** (Objective) — Target for SLI (e.g., p99 < 200ms)
- **SLA** (Agreement) — Contract with consequences if SLO is missed
- **SRE** — Applies software engineering to operations; error budgets
- **DevOps** — Culture of collaboration between dev and ops
- **Platform Engineering** — Build internal developer platforms

---

## Domain 4 — Cloud Native Observability (8%)

### Three Pillars

| Pillar | What | Tool |
|--------|------|------|
| Metrics | Numeric measurements over time | Prometheus, Grafana |
| Logs | Timestamped event records | Fluentd, Fluent Bit, EFK |
| Traces | Request path across services | Jaeger, Zipkin |

### Prometheus

- **Pull model** — Scrapes `/metrics` endpoints
- **PromQL** — Query language for metrics
- **Metric types**: Counter, Gauge, Histogram, Summary
- **Alertmanager** — Routes and deduplicates alerts

### OpenTelemetry

- CNCF project (incubating)
- Vendor-neutral collection of metrics, logs, and traces
- Uses **collectors** to receive, process, and export telemetry
- Replaces OpenTracing + OpenCensus

---

## Domain 5 — Cloud Native Application Delivery (8%)

### Deployment Strategies

| Strategy | Downtime | Risk | Rollback |
|----------|----------|------|----------|
| Rolling Update | None | Medium (gradual) | Fast (roll back) |
| Blue-Green | None | Low (full test first) | Instant (switch back) |
| Canary | None | Low (small % first) | Fast (route away) |
| Recreate | Yes | High | Slow (redeploy) |

### Helm

- **Chart** — Package of K8s templates
- **Release** — Instance of a chart deployed to a cluster
- **Values** — Configuration that customizes a chart
- **Repository** — Collection of charts
- Commands: `helm install`, `helm upgrade`, `helm rollback`, `helm list`

### GitOps Principles

1. **Declarative** — Desired state described in Git
2. **Versioned** — Git history = audit trail
3. **Automated** — Changes pulled and applied automatically
4. **Reconciled** — Agent continuously syncs cluster to Git state

- **ArgoCD** — Watches Git repos, syncs to cluster, shows drift
- **FluxCD** — Similar; pulls changes from Git into cluster

### CI/CD

- **CI** — Build, test, and validate on every commit
- **CD** — Automatically deploy validated changes
- **Tools**: Jenkins, GitHub Actions, GitLab CI, Tekton
- **Tekton** — Kubernetes-native CI/CD (CNCF project)
