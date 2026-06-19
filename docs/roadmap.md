# KCNA Exam Prep — Roadmap

> A structured path through every exam domain. Each link points to a concept file or hands-on lab.

---

## Phase 1 — Kubernetes Fundamentals (46%)

_The biggest chunk of the exam. Master this first._

| # | Topic | Concept | Lab |
|---|-------|---------|-----|
| 1 | Cluster Architecture | [concepts/kubernetes-architecture.md](../concepts/kubernetes-architecture.md) | [labs/01-cluster-setup.md](../labs/01-cluster-setup.md) |
| 2 | Control Plane Components | [concepts/control-plane.md](../concepts/control-plane.md) | [labs/02-explore-control-plane.md](../labs/02-explore-control-plane.md) |
| 3 | Worker Node Components | [concepts/worker-nodes.md](../concepts/worker-nodes.md) | [labs/03-inspect-worker-nodes.md](../labs/03-inspect-worker-nodes.md) |
| 4 | Kubernetes API & kubectl | [concepts/kubernetes-api.md](../concepts/kubernetes-api.md) | [labs/04-kubectl-basics.md](../labs/04-kubectl-basics.md) |
| 5 | Pods | [concepts/pods.md](../concepts/pods.md) | [labs/05-pods.md](../labs/05-pods.md) |
| 6 | Deployments & ReplicaSets | [concepts/deployments.md](../concepts/deployments.md) | [labs/06-deployments.md](../labs/06-deployments.md) |
| 7 | Services | [concepts/services.md](../concepts/services.md) | [labs/07-services.md](../labs/07-services.md) |
| 8 | Namespaces, Labels & Selectors | [concepts/namespaces-labels.md](../concepts/namespaces-labels.md) | [labs/08-namespaces-labels.md](../labs/08-namespaces-labels.md) |
| 9 | ConfigMaps & Secrets | [concepts/configmaps-secrets.md](../concepts/configmaps-secrets.md) | [labs/09-configmaps-secrets.md](../labs/09-configmaps-secrets.md) |
| 10 | Jobs, CronJobs, DaemonSets, StatefulSets | [concepts/workload-resources.md](../concepts/workload-resources.md) | [labs/10-workload-resources.md](../labs/10-workload-resources.md) |
| 11 | Scheduling (Taints, Tolerations, Affinity) | [concepts/scheduling.md](../concepts/scheduling.md) | [labs/11-scheduling.md](../labs/11-scheduling.md) |
| 12 | Containers & Images | [concepts/containers.md](../concepts/containers.md) | [labs/12-containers.md](../labs/12-containers.md) |

---

## Phase 2 — Container Orchestration (22%)

| # | Topic | Concept | Lab |
|---|-------|---------|-----|
| 13 | Orchestration Fundamentals | [concepts/orchestration-fundamentals.md](../concepts/orchestration-fundamentals.md) | — |
| 14 | Container Runtimes (CRI, containerd, CRI-O) | [concepts/container-runtimes.md](../concepts/container-runtimes.md) | [labs/13-container-runtimes.md](../labs/13-container-runtimes.md) |
| 15 | Networking & CNI | [concepts/networking.md](../concepts/networking.md) | [labs/14-networking.md](../labs/14-networking.md) |
| 16 | Ingress & DNS | [concepts/ingress-dns.md](../concepts/ingress-dns.md) | [labs/15-ingress.md](../labs/15-ingress.md) |
| 17 | RBAC & Security | [concepts/rbac-security.md](../concepts/rbac-security.md) | [labs/16-rbac.md](../labs/16-rbac.md) |
| 18 | Network Policies | [concepts/network-policies.md](../concepts/network-policies.md) | [labs/17-network-policies.md](../labs/17-network-policies.md) |
| 19 | Service Mesh (Istio, Linkerd) | [concepts/service-mesh.md](../concepts/service-mesh.md) | [labs/18-service-mesh.md](../labs/18-service-mesh.md) |
| 20 | Storage (CSI, PV, PVC) | [concepts/storage.md](../concepts/storage.md) | [labs/19-storage.md](../labs/19-storage.md) |

---

## Phase 3 — Cloud Native Architecture (16%)

| # | Topic | Concept | Lab |
|---|-------|---------|-----|
| 21 | Cloud Native Principles & 12-Factor App | [concepts/cloud-native-principles.md](../concepts/cloud-native-principles.md) | — |
| 22 | Microservices vs Monoliths | [concepts/microservices.md](../concepts/microservices.md) | — |
| 23 | Autoscaling (HPA, VPA, Cluster Autoscaler) | [concepts/autoscaling.md](../concepts/autoscaling.md) | [labs/20-autoscaling.md](../labs/20-autoscaling.md) |
| 24 | Serverless & Knative | [concepts/serverless.md](../concepts/serverless.md) | [labs/21-serverless.md](../labs/21-serverless.md) |
| 25 | CNCF Ecosystem & Governance | [concepts/cncf-ecosystem.md](../concepts/cncf-ecosystem.md) | — |
| 26 | Roles & Personas (SRE, DevOps, SLI/SLO/SLA) | [concepts/roles-personas.md](../concepts/roles-personas.md) | — |
| 27 | Open Standards (OCI, CRI, CNI, CSI, SMI) | [concepts/open-standards.md](../concepts/open-standards.md) | — |

---

## Phase 4 — Cloud Native Observability (8%)

| # | Topic | Concept | Lab |
|---|-------|---------|-----|
| 28 | Observability Fundamentals (Metrics, Logs, Traces) | [concepts/observability.md](../concepts/observability.md) | — |
| 29 | Prometheus & Alertmanager | [concepts/prometheus.md](../concepts/prometheus.md) | [labs/22-prometheus.md](../labs/22-prometheus.md) |
| 30 | Grafana, Fluentd, Jaeger | [concepts/observability-tools.md](../concepts/observability-tools.md) | [labs/23-grafana-logging.md](../labs/23-grafana-logging.md) |
| 31 | OpenTelemetry | [concepts/opentelemetry.md](../concepts/opentelemetry.md) | — |

---

## Phase 5 — Cloud Native Application Delivery (8%)

| # | Topic | Concept | Lab |
|---|-------|---------|-----|
| 32 | Deployment Strategies (Rolling, Blue-Green, Canary) | [concepts/deployment-strategies.md](../concepts/deployment-strategies.md) | [labs/24-deployment-strategies.md](../labs/24-deployment-strategies.md) |
| 33 | Helm | [concepts/helm.md](../concepts/helm.md) | [labs/25-helm.md](../labs/25-helm.md) |
| 34 | GitOps (ArgoCD, FluxCD) | [concepts/gitops.md](../concepts/gitops.md) | [labs/26-gitops-argocd.md](../labs/26-gitops-argocd.md) |
| 35 | CI/CD Pipelines | [concepts/cicd.md](../concepts/cicd.md) | [labs/27-cicd-pipeline.md](../labs/27-cicd-pipeline.md) |

---

## Quick Reference

| Resource | Description |
|----------|-------------|
| [Practice Exam 1](practice-exam.md) | 60 multiple-choice questions matching real exam format and domain weightage |
| [Practice Exam 2](practice-exam-2.md) | A second 60-question exam with different questions, same domain weightage |
| [Cheat Sheet](cheat-sheet.md) | One-page quick-review reference for last-minute revision |
| [Glossary](glossary.md) | Alphabetical list of key terms and definitions |

---

## Suggested Study Order

1. **Week 1–2** — Phase 1 (topics 1–12): Kubernetes fundamentals are nearly half the exam.
2. **Week 3** — Phase 2 (topics 13–20): Orchestration, networking, security, storage.
3. **Week 4** — Phase 3 (topics 21–27): Architecture patterns, CNCF ecosystem.
4. **Week 5** — Phase 4 + 5 (topics 28–35): Observability and delivery — lighter weight, but free marks.
5. **Week 6** — Review weak areas, take practice exams, re-do labs.

---

## Progress Tracker

Use this checklist to track completion:

- [ ] Phase 1 — Kubernetes Fundamentals
- [ ] Phase 2 — Container Orchestration
- [ ] Phase 3 — Cloud Native Architecture
- [ ] Phase 4 — Cloud Native Observability
- [ ] Phase 5 — Cloud Native Application Delivery
- [ ] Practice exams completed
