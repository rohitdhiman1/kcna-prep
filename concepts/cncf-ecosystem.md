# CNCF Ecosystem

> The Cloud Native Computing Foundation (CNCF) is the vendor-neutral home for cloud native open-source projects, including Kubernetes.

---

## What Is CNCF?

The **Cloud Native Computing Foundation (CNCF)** is part of the **Linux Foundation**. It was founded in 2015 with Kubernetes as its first hosted project.

### Mission

> "To make cloud native computing ubiquitous."

### Role

- **Hosts and nurtures** open-source cloud native projects.
- Provides **governance** and structure for project maturity.
- Promotes **vendor neutrality** — no single company controls the projects.
- Organizes **KubeCon + CloudNativeCon**, the largest cloud native conference.
- Maintains the **CNCF Landscape** — a map of the entire cloud native ecosystem.
- Offers **certifications**: CKA, CKAD, CKS, KCNA, KCSA.

### Governance

- **Technical Oversight Committee (TOC)**: Oversees project acceptance and maturity levels. Provides technical vision.
- **Governing Board**: Business oversight, budget, marketing. Composed of member company representatives.
- **End User Community**: Organizations using cloud native in production that provide feedback and direction.

---

## Project Maturity Levels

CNCF projects follow a three-stage maturity pipeline:

```
  ┌────────────────────────────────────────────────────────────┐
  │              CNCF Project Maturity Pipeline                 │
  │                                                              │
  │   ┌───────────┐      ┌──────────────┐      ┌────────────┐  │
  │   │           │      │              │      │            │  │
  │   │  SANDBOX  │─────►│  INCUBATING  │─────►│ GRADUATED  │  │
  │   │           │      │              │      │            │  │
  │   └───────────┘      └──────────────┘      └────────────┘  │
  │                                                              │
  │   Early stage         Growing adoption      Production-     │
  │   Experimental        Proven value           ready          │
  │   Innovation          Healthy community      Mature         │
  │   Minimal adoption    Good governance        Widely adopted │
  │                                                              │
  │   Entry criteria:     Criteria:              Criteria:      │
  │   - Novel approach    - Healthy contrib.     - Adopted by   │
  │   - TOC sponsor       - Production use       many orgs     │
  │   - Follows CNCF      - Clear governance    - Committer    │
  │     guidelines        - Growing community    process       │
  │                                              - Security     │
  │                                                audit done   │
  │                                              - Follows CNCF │
  │                                                best pracs   │
  └────────────────────────────────────────────────────────────┘
```

### Sandbox
- **Early-stage** projects with potential.
- Experimental; not yet recommended for production.
- Low barrier to entry — requires a TOC sponsor.
- Many projects start here and may never advance.

### Incubating
- Projects that have demonstrated **growing adoption** and a **healthy community**.
- Used in production by some organizations.
- Must have clear governance and contributor diversity (not single-company dominated).
- Examples: Knative, Backstage, Argo, Dapr, KEDA.

### Graduated
- **Production-proven** projects used by many organizations worldwide.
- Must pass a **security audit**.
- Strong governance, diverse contributors, mature processes.
- These are the projects you can confidently rely on.

---

## Key Graduated Projects

| Project | What It Does |
|---------|-------------|
| **Kubernetes** | Container orchestration platform |
| **Prometheus** | Metrics collection and alerting (pull-based monitoring) |
| **Envoy** | High-performance service proxy (used in Istio, Ambassador) |
| **containerd** | Industry-standard container runtime |
| **CoreDNS** | DNS server for service discovery in Kubernetes |
| **Helm** | Package manager for Kubernetes (charts) |
| **etcd** | Distributed key-value store (Kubernetes backing store) |
| **Fluentd** | Log collection and forwarding |
| **Jaeger** | Distributed tracing |
| **Open Policy Agent (OPA)** | Policy engine for authorization |
| **TUF** | The Update Framework — secure software updates |
| **Vitess** | Database clustering for horizontal scaling of MySQL |
| **Harbor** | Container image registry with security scanning |
| **Rook** | Storage orchestration for Kubernetes |
| **TiKV** | Distributed transactional key-value database |
| **Linkerd** | Service mesh |
| **Flux** | GitOps for Kubernetes |
| **Argo** | Workflow engine, CI/CD, GitOps for Kubernetes |
| **Cilium** | eBPF-based networking and security |
| **OpenTelemetry** | Observability framework (metrics, logs, traces) |
| **Istio** | Service mesh |
| **Falco** | Runtime security and threat detection |
| **cert-manager** | X.509 certificate management for Kubernetes |

---

## CNCF Landscape

The **CNCF Landscape** (https://landscape.cncf.io/) is an interactive map of the entire cloud native ecosystem.

```
  ┌───────────────────────────────────────────────────┐
  │              CNCF Landscape Categories             │
  │                                                     │
  │  App Definition    Orchestration    Runtime         │
  │  & Development     & Management                    │
  │  ┌────────────┐   ┌────────────┐   ┌────────────┐ │
  │  │ Helm       │   │ Kubernetes │   │ containerd │ │
  │  │ Backstage  │   │ Crossplane │   │ CRI-O      │ │
  │  │ Dapr       │   │ Kustomize  │   │ gVisor     │ │
  │  └────────────┘   └────────────┘   └────────────┘ │
  │                                                     │
  │  Provisioning      Observability    Serverless     │
  │  ┌────────────┐   ┌────────────┐   ┌────────────┐ │
  │  │ Terraform  │   │ Prometheus │   │ Knative    │ │
  │  │ Pulumi     │   │ Grafana    │   │ OpenFaaS   │ │
  │  │ Ansible    │   │ Jaeger     │   │            │ │
  │  └────────────┘   └────────────┘   └────────────┘ │
  │                                                     │
  │  Security          Networking                       │
  │  ┌────────────┐   ┌────────────┐                   │
  │  │ Falco      │   │ Envoy      │                   │
  │  │ OPA        │   │ Cilium     │                   │
  │  │ cert-mgr   │   │ CoreDNS    │                   │
  │  └────────────┘   └────────────┘                   │
  │                                                     │
  │  1000+ projects across all categories               │
  └───────────────────────────────────────────────────┘
```

The Landscape includes:
- **CNCF projects** (Sandbox, Incubating, Graduated).
- **Non-CNCF projects** that are part of the ecosystem (e.g., Terraform, Grafana).
- **Commercial products** and vendors.
- Categories covering every layer of the cloud native stack.

---

## Special Interest Groups (SIGs)

**SIGs** are community groups within the **Kubernetes project** that focus on specific areas of the codebase or functionality.

### Key Kubernetes SIGs

| SIG | Focus Area |
|-----|-----------|
| SIG Architecture | Overall architecture, API conventions |
| SIG Auth | Authentication, authorization, security policy |
| SIG Autoscaling | HPA, VPA, Cluster Autoscaler |
| SIG CLI | kubectl and CLI tools |
| SIG Network | Networking, Services, Ingress, DNS |
| SIG Node | Node lifecycle, kubelet, container runtime |
| SIG Storage | Persistent volumes, CSI, storage classes |
| SIG Apps | Deployments, StatefulSets, Jobs, CronJobs |
| SIG Scheduling | Scheduler, resource management |
| SIG Release | Kubernetes release process |
| SIG Docs | Documentation |
| SIG Testing | Testing infrastructure and practices |

### How SIGs Work
- Open to anyone — meetings are public, notes are shared.
- Each SIG has **chairs** and **technical leads**.
- SIGs own specific parts of the Kubernetes codebase.
- SIGs propose and review changes through **KEPs** (Kubernetes Enhancement Proposals).

---

## Working Groups

**Working Groups** are cross-cutting groups that span multiple SIGs to address broader topics.

Examples:
- **WG Policy**: Policies across security, networking, and multi-tenancy.
- **WG Reliability**: Improving Kubernetes reliability.
- **WG Batch**: Batch processing and job scheduling.

Working Groups are **temporary** — they are created to address specific topics and are dissolved when the work is complete (unlike SIGs, which are permanent).

---

## Kubernetes Enhancement Proposals (KEPs)

A **KEP** is the formal process for proposing, discussing, and approving significant changes to Kubernetes.

```
  ┌──────────────────────────────────────────────────────────┐
  │                  KEP Lifecycle                            │
  │                                                            │
  │  ┌──────────┐   ┌──────────┐   ┌──────────┐              │
  │  │          │   │          │   │          │              │
  │  │  Draft   │──►│Provisional──►│Implement-│──►Completed  │
  │  │          │   │          │   │  able    │              │
  │  └──────────┘   └──────────┘   └──────────┘              │
  │       │                                                    │
  │       └──► Rejected / Withdrawn / Deferred                │
  │                                                            │
  └──────────────────────────────────────────────────────────┘
```

### KEP Process

1. **Author writes a KEP** describing the problem, proposed solution, and alternatives considered.
2. **KEP is filed as a PR** to the kubernetes/enhancements repository.
3. The relevant **SIG reviews** and discusses the proposal.
4. KEP progresses through stages:
   - **Draft** → Initial proposal.
   - **Provisional** → SIG agrees the problem is worth solving.
   - **Implementable** → Design is approved; work can begin.
   - **Completed** → Feature is implemented and merged.
5. A KEP can be **rejected, withdrawn, or deferred** at any stage.

### What a KEP Contains
- **Motivation**: Why is this change needed?
- **Proposal**: How will it be implemented?
- **Graduation criteria**: Requirements for moving from alpha → beta → stable.
- **Risks and mitigations**: What could go wrong?
- **Alternatives**: Other approaches considered.
- **Test plan**: How the feature will be tested.

### Why KEPs Matter
- Ensure significant changes are **well-thought-out** and **community-reviewed**.
- Provide **documentation** and history for every major feature.
- Enable **transparent decision-making**.

---

## CNCF TAG (Technical Advisory Groups)

TAGs (formerly called SIGs at the CNCF level, not to be confused with Kubernetes SIGs) provide **cross-project guidance** within CNCF.

| TAG | Focus |
|-----|-------|
| TAG App Delivery | Application definition, deployment, operations |
| TAG Contributor Strategy | Contributor experience and growth |
| TAG Environmental Sustainability | Reducing cloud native environmental impact |
| TAG Network | Networking standards and practices |
| TAG Observability | Observability standards and practices |
| TAG Runtime | Container runtimes and execution environments |
| TAG Security | Security best practices and assessments |
| TAG Storage | Storage interfaces and implementations |

---

## What to Remember for the Exam

1. **CNCF** is part of the **Linux Foundation**. Its mission is to make cloud native computing ubiquitous.
2. Three maturity levels: **Sandbox → Incubating → Graduated**. Know the progression and what each level means.
3. **Graduated** projects are production-proven. Key ones to know: **Kubernetes, Prometheus, Envoy, containerd, CoreDNS, Helm, etcd, Fluentd, Argo, Flux**.
4. The **CNCF Landscape** maps 1000+ projects across the cloud native ecosystem.
5. **SIGs** (Special Interest Groups) are permanent groups in the Kubernetes project that own specific areas (networking, storage, scheduling, etc.).
6. **Working Groups** are temporary, cross-SIG groups for specific topics.
7. **KEPs** (Kubernetes Enhancement Proposals) are the formal process for significant Kubernetes changes. They go through Draft → Provisional → Implementable → Completed.
8. The **Technical Oversight Committee (TOC)** oversees project acceptance and maturity at the CNCF level.
9. **TAGs** provide cross-project guidance at the CNCF level (different from Kubernetes SIGs).
10. CNCF promotes **vendor neutrality** — no single company controls the projects.
