# KCNA Exam Prep

A complete, self-contained study guide for the **Kubernetes and Cloud Native Associate (KCNA)** certification exam by CNCF / Linux Foundation.

---

## Exam Snapshot

| Detail | Info |
|--------|------|
| Format | Online, proctored, multiple-choice |
| Questions | 60 |
| Duration | 90 minutes |
| Passing Score | 75% |
| Retakes | 2 attempts included |

---

## Repository Structure

```
kcna-prep/
│
├── concepts/          # Theory & diagrams for every exam topic
│   ├── kubernetes-architecture.md
│   ├── control-plane.md
│   ├── worker-nodes.md
│   ├── pods.md
│   ├── deployments.md
│   ├── services.md
│   ├── networking.md
│   ├── rbac-security.md
│   ├── observability.md
│   ├── gitops.md
│   └── ...            # one file per topic (35 total)
│
├── labs/              # Hands-on exercises with step-by-step instructions
│   ├── 01-cluster-setup.md
│   ├── 02-explore-control-plane.md
│   ├── 05-pods.md
│   ├── 06-deployments.md
│   ├── 07-services.md
│   └── ...            # 27 labs covering practical skills
│
├── docs/
│   ├── roadmap.md     # Study roadmap with links to all concepts & labs
│   ├── practice-exam.md  # 60-question practice exam with answer key
│   ├── cheat-sheet.md    # Quick-review reference for revision
│   └── glossary.md       # Key terms and definitions A–Z
│
└── README.md          # You are here
```

### How the pieces fit together

- **`concepts/`** — Each file covers one exam topic with clear explanations and simple ASCII/text diagrams embedded inline. No mermaid — diagrams use plain-text box-and-arrow style for maximum portability.
- **`labs/`** — Numbered, hands-on exercises you run on a real cluster (kind, minikube, or a cloud provider). Each lab references the matching concept file.
- **`docs/roadmap.md`** — The master navigation file. It maps every exam domain to its concept and lab files with direct hyperlinks so you can jump between theory and practice seamlessly.

---

## Exam Domains & Weightage

```
+-----------------------------------------------+--------+
|  Domain                                        | Weight |
+-----------------------------------------------+--------+
|  1. Kubernetes Fundamentals                    |  46%   |
|  2. Container Orchestration                    |  22%   |
|  3. Cloud Native Architecture                  |  16%   |
|  4. Cloud Native Observability                 |   8%   |
|  5. Cloud Native Application Delivery          |   8%   |
+-----------------------------------------------+--------+
```

### Domain 1 — Kubernetes Fundamentals (46%)

The largest domain. Covers the core building blocks.

```
  ┌─────────────────── Cluster ───────────────────┐
  │                                                │
  │  Control Plane              Worker Nodes       │
  │  ┌──────────────┐          ┌──────────────┐    │
  │  │ api-server   │◄────────►│ kubelet      │    │
  │  │ etcd         │          │ kube-proxy   │    │
  │  │ scheduler    │          │ container    │    │
  │  │ controller-  │          │  runtime     │    │
  │  │  manager     │          │ (containerd) │    │
  │  └──────────────┘          └──────────────┘    │
  │                                                │
  └────────────────────────────────────────────────┘
```

**Key topics:**
- Cluster architecture (control plane + worker nodes)
- Kubernetes API, API groups, kubectl
- Core resources: Pods, Deployments, ReplicaSets, Services, Namespaces
- ConfigMaps, Secrets, Jobs, CronJobs, DaemonSets, StatefulSets
- Labels, selectors, annotations
- Scheduling: taints, tolerations, node affinity, resource requests/limits
- Containers, images, registries, container vs VM

### Domain 2 — Container Orchestration (22%)

How Kubernetes manages containers at scale.

```
  ┌─── Orchestration Layer ────────────────────────────┐
  │                                                     │
  │  Runtime          Networking        Security        │
  │  ┌──────────┐    ┌──────────┐    ┌──────────────┐  │
  │  │ CRI      │    │ CNI      │    │ RBAC         │  │
  │  │ containerd│    │ Calico   │    │ Pod Security │  │
  │  │ CRI-O    │    │ Cilium   │    │ Net Policies │  │
  │  └──────────┘    │ CoreDNS  │    │ OPA          │  │
  │                  │ Ingress  │    └──────────────┘  │
  │  Storage         └──────────┘                      │
  │  ┌──────────┐                  Service Mesh        │
  │  │ CSI      │                  ┌──────────────┐    │
  │  │ PV / PVC │                  │ Istio        │    │
  │  │ Storage  │                  │ Linkerd      │    │
  │  │  Classes │                  │ mTLS/sidecar │    │
  │  └──────────┘                  └──────────────┘    │
  │                                                     │
  └─────────────────────────────────────────────────────┘
```

**Key topics:**
- Container runtimes: CRI, containerd, CRI-O, runc, OCI specs
- Networking: CNI plugins, pod-to-pod communication, Services, DNS, Ingress
- Security: RBAC, Pod Security Standards, Network Policies, OPA/Gatekeeper
- Service Mesh: sidecar pattern, Istio, Linkerd, SMI
- Storage: CSI, Persistent Volumes, PVCs, StorageClasses

### Domain 3 — Cloud Native Architecture (16%)

Patterns, principles, and the CNCF ecosystem.

**Key topics:**
- Cloud native characteristics: scalable, resilient, observable, manageable
- Microservices vs monoliths, 12-Factor App
- Autoscaling: HPA, VPA, Cluster Autoscaler, KEDA
- Serverless: FaaS, Knative, event-driven architecture
- CNCF governance: Sandbox → Incubating → Graduated
- SIGs, KEPs, SRE, DevOps, Platform Engineering
- SLIs, SLOs, SLAs
- Open standards: OCI, CRI, CNI, CSI, SMI

### Domain 4 — Cloud Native Observability (8%)

```
  Observability Pillars
  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │  Metrics   │  │   Logs     │  │  Traces    │
  │            │  │            │  │            │
  │ Prometheus │  │ Fluentd    │  │ Jaeger     │
  │ Grafana    │  │ Fluent Bit │  │ Zipkin     │
  │ AlertMgr   │  │ EFK stack  │  │            │
  └────────────┘  └────────────┘  └────────────┘
         \              |              /
          \             |             /
           ▼            ▼            ▼
        ┌───────────────────────────────┐
        │       OpenTelemetry           │
        │  (unified collection layer)   │
        └───────────────────────────────┘
```

**Key topics:**
- Three pillars: metrics, logs, traces
- Prometheus: pull model, PromQL basics, metric types
- Grafana, Fluentd/Fluent Bit, Jaeger/Zipkin
- OpenTelemetry
- Cost management and right-sizing

### Domain 5 — Cloud Native Application Delivery (8%)

```
  Code ──► Build ──► Test ──► Deploy ──► Operate
                                │
               ┌────────────────┴─────────────────┐
               │        Deployment Strategies      │
               │                                   │
               │  Rolling    Blue/Green   Canary   │
               │  ┌─────┐   ┌───┬───┐   ┌─────┐  │
               │  │v1→v2│   │ v1│ v2│   │95/5 │  │
               │  │     │   │   │   │   │v1/v2│  │
               │  └─────┘   └───┴───┘   └─────┘  │
               └───────────────────────────────────┘

  GitOps: Git ──(push)──► ArgoCD/Flux ──(sync)──► Cluster
```

**Key topics:**
- Deployment strategies: rolling, blue-green, canary
- Helm: charts, templates, releases
- GitOps principles: declarative, versioned, automated, reconciled
- ArgoCD, FluxCD
- CI/CD: pipelines, stages, Jenkins, GitHub Actions, Tekton

---

## Getting Started

### Prerequisites

- A local Kubernetes cluster — [kind](https://kind.sigs.k8s.io/), [minikube](https://minikube.sigs.k8s.io/), or [k3s](https://k3s.io/)
- `kubectl` installed and configured
- Docker or another container runtime
- Basic Linux/terminal comfort

### Study Flow

1. **Start with the [Roadmap](docs/roadmap.md)** — it lays out the full study plan with links to every concept and lab.
2. **Read the concept file** for a topic to understand the theory.
3. **Do the matching lab** to reinforce with hands-on practice.
4. **Check off progress** in the roadmap as you go.

### Suggested Timeline

| Week | Focus | Domains |
|------|-------|---------|
| 1–2 | Kubernetes Fundamentals | Domain 1 (46%) |
| 3 | Container Orchestration | Domain 2 (22%) |
| 4 | Cloud Native Architecture | Domain 3 (16%) |
| 5 | Observability + App Delivery | Domains 4 & 5 (16%) |
| 6 | Review + Practice Exams | All |

---

## Resources

- [Official KCNA Curriculum (CNCF)](https://github.com/cncf/curriculum)
- [Linux Foundation — KCNA Certification](https://training.linuxfoundation.org/certification/kubernetes-cloud-native-associate/)
- [Kubernetes Official Docs](https://kubernetes.io/docs/)
- [CNCF Landscape](https://landscape.cncf.io/)

---

## License

This repository is for personal study use. Kubernetes and CNCF are trademarks of The Linux Foundation.
