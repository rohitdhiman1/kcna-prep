# GitOps

**GitOps is an operational framework where the desired state of infrastructure and applications is declared in Git, and automated agents continuously reconcile the live system to match that declared state.**

---

## Table of Contents

- [GitOps Principles](#gitops-principles)
- [Git as the Single Source of Truth](#git-as-the-single-source-of-truth)
- [Push vs Pull Model](#push-vs-pull-model)
- [The GitOps Loop](#the-gitops-loop)
- [ArgoCD](#argocd)
- [FluxCD](#fluxcd)
- [ArgoCD vs FluxCD](#argocd-vs-fluxcd)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## GitOps Principles

GitOps is built on four core principles:

### 1. Declarative

The entire system (infrastructure and applications) is described **declaratively** — you define the desired state, not the steps to get there.

### 2. Versioned and Immutable

The desired state is stored in **Git**, providing:
- Full version history (who changed what, when)
- Built-in audit trail
- Ability to revert to any previous state

### 3. Pulled Automatically

Approved changes in Git are **automatically applied** to the system. No manual `kubectl apply` needed.

### 4. Continuously Reconciled

An agent running inside the cluster **continuously compares** the actual state of the cluster with the desired state in Git and corrects any **drift**.

```
+-------------------------------------------------------+
|                 FOUR GITOPS PRINCIPLES                 |
|                                                       |
|   1. DECLARATIVE                                      |
|      "What" not "How" — YAML in Git                   |
|                                                       |
|   2. VERSIONED & IMMUTABLE                            |
|      Git = audit trail, rollback built-in             |
|                                                       |
|   3. PULLED AUTOMATICALLY                             |
|      Agent pulls from Git, no manual deploys          |
|                                                       |
|   4. CONTINUOUSLY RECONCILED                          |
|      Agent detects drift and self-heals               |
+-------------------------------------------------------+
```

---

## Git as the Single Source of Truth

In GitOps, **Git is the single source of truth** for the desired state of the system. This means:

- Every change goes through a Git commit (and ideally a pull request)
- The cluster state should always match what is in Git
- If someone makes a manual change to the cluster, the GitOps agent will **revert it** to match Git
- Rollback = `git revert` — no special tooling needed

```
+-----------------+       +-------------------+       +------------------+
|   Developer     |       |   Git Repository  |       |   K8s Cluster    |
|                 |       |                   |       |                  |
| 1. Writes YAML  |       | Stores desired    |       | Runs actual      |
| 2. Opens PR     +------>| state as code     +------>| workloads        |
| 3. Merges PR    |       |                   |       |                  |
+-----------------+       | - deployments     |       | Agent ensures    |
                          | - services        |       | cluster matches  |
                          | - configmaps      |       | Git at all times |
                          +-------------------+       +------------------+
```

---

## Push vs Pull Model

### Push Model (Traditional CI/CD)

The CI/CD pipeline **pushes** changes to the cluster. The pipeline has cluster credentials.

```
+--------+     +-----------+     +-------------------+
| Git    | --> | CI/CD     | --> | K8s Cluster       |
| Push   |     | Pipeline  |     |                   |
+--------+     | (Jenkins, |     | kubectl apply ... |
               | GH Actions)|    +-------------------+
               +-----------+
                    |
                    +-- Has cluster credentials (security risk)
                    +-- No drift detection
                    +-- No continuous reconciliation
```

### Pull Model (GitOps)

An agent **inside the cluster** watches Git and **pulls** changes. The CI/CD pipeline never touches the cluster.

```
+--------+     +-----------+     +-------------------+
| Git    | --> | CI/CD     |     | K8s Cluster       |
| Push   |     | Pipeline  |     |                   |
+--------+     +-----------+     |  +--------------+ |
                    |             |  | GitOps Agent | |
                    |             |  | (ArgoCD /    | |
                    v             |  |  FluxCD)     | |
            +---------------+    |  +------+-------+ |
            | Container     |    |         |         |
            | Registry      |    |         | watches |
            +---------------+    |         v         |
                                 |  +-------------+  |
                                 |  | Git Repo    |  |
                                 |  +-------------+  |
                                 +-------------------+
```

**Key difference:** In the pull model, the cluster pulls its own state from Git. The CI/CD pipeline only builds images and updates the Git repo — it never directly accesses the cluster.

**Why pull is better:**
- No cluster credentials stored in CI/CD systems
- Continuous drift detection and reconciliation
- Git is the only way to make changes (auditability)

---

## The GitOps Loop

```
+------------------------------------------------------------------+
|                        THE GITOPS LOOP                           |
|                                                                  |
|                                                                  |
|    +----------------+         +-----------------+                |
|    |  Git Repo      |         |  K8s Cluster    |                |
|    |  (desired      |         |  (actual state) |                |
|    |   state)       |         |                 |                |
|    +-------+--------+         +--------+--------+                |
|            |                           |                         |
|            |   1. Agent polls Git      |                         |
|            |<--------------------------+                         |
|            |                           |                         |
|            |   2. Compares desired     |                         |
|            |      vs actual state      |                         |
|            |                           |                         |
|            |   3a. In Sync?            |                         |
|            |       Do nothing          |                         |
|            |                           |                         |
|            |   3b. Drift detected?     |                         |
|            +-------------------------->|                         |
|                 4. Reconcile:          |                         |
|                    Apply changes       |                         |
|                    to match Git        |                         |
|                                        |                         |
|            +<--------------------------+                         |
|            |   5. Report status back   |                         |
|            |                           |                         |
|            |   (loop repeats)          |                         |
|            +-------------------------->|                         |
|                                                                  |
+------------------------------------------------------------------+

Summary:  Git  -->  Agent  -->  Cluster  -->  Detect Drift  -->  Reconcile
           ^                                                        |
           +--------------------------------------------------------+
```

---

## ArgoCD

ArgoCD is a **declarative, GitOps continuous delivery tool** for Kubernetes. It is a CNCF graduated project.

### Architecture

```
+------------------------------------------------------------------+
|                       ARGOCD ARCHITECTURE                        |
|                                                                  |
|   +---------------------+                                       |
|   |    ArgoCD Server    |                                       |
|   |                     |                                       |
|   |  +--------------+   |   +-----------------------------+     |
|   |  | API Server   |   |   |  Git Repository             |     |
|   |  | - REST/gRPC  |   |   |  (GitHub, GitLab, Bitbucket)|     |
|   |  | - Web UI     |   |   +-------------+---------------+     |
|   |  | - CLI access |   |                 |                     |
|   |  +--------------+   |                 |                     |
|   |                     |                 | polls               |
|   |  +--------------+   |                 |                     |
|   |  | Repo Server  |<--+-----------------+                     |
|   |  | - Clones     |   |                                       |
|   |  |   repos      |   |                                       |
|   |  | - Generates  |   |                                       |
|   |  |   manifests  |   |                                       |
|   |  +--------------+   |                                       |
|   |                     |                                       |
|   |  +----------------+ |   +-----------------------------+     |
|   |  | Application    | |   |  Kubernetes Cluster         |     |
|   |  | Controller     +-+-->|                             |     |
|   |  | - Watches Git  | |   |  Applies resources          |     |
|   |  | - Compares     | |   |  Detects drift              |     |
|   |  |   desired vs   | |   |  Reconciles                 |     |
|   |  |   live state   | |   +-----------------------------+     |
|   |  +----------------+ |                                       |
|   +---------------------+                                       |
+------------------------------------------------------------------+
```

### Core Components

| Component              | Role                                                      |
|------------------------|-----------------------------------------------------------|
| **API Server**         | Exposes REST/gRPC API, serves Web UI, handles CLI requests |
| **Repo Server**        | Clones Git repos, generates Kubernetes manifests           |
| **Application Controller** | Monitors running apps, compares live state vs desired, reconciles drift |

### Application CRD

ArgoCD extends Kubernetes with a custom resource called `Application`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app-config.git
    targetRevision: main
    path: k8s/
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Delete resources removed from Git
      selfHeal: true     # Revert manual changes in cluster
    syncOptions:
      - CreateNamespace=true
```

### Sync Policies

| Policy      | Description                                              |
|-------------|----------------------------------------------------------|
| **Manual**  | User must click "Sync" in UI or run `argocd app sync`   |
| **Automated** | ArgoCD auto-syncs when Git changes are detected       |
| **Self-Heal** | Reverts manual cluster changes to match Git           |
| **Prune**   | Deletes resources from cluster that were removed from Git |

---

## FluxCD

FluxCD is a set of **continuous delivery tools** for Kubernetes that follow GitOps principles. It is also a CNCF graduated project.

### Architecture

FluxCD runs as a set of controllers inside the cluster:

```
+------------------------------------------------------------------+
|                       FLUXCD ARCHITECTURE                        |
|                                                                  |
|   +---------------------------+                                 |
|   |     Kubernetes Cluster    |                                 |
|   |                           |                                 |
|   |  +---------------------+  |    +------------------------+   |
|   |  | Source Controller   |  |    | Git Repository         |   |
|   |  | - Fetches from Git  |--+--->| (stores desired state) |   |
|   |  | - Fetches from Helm |  |    +------------------------+   |
|   |  +----------+----------+  |                                 |
|   |             |              |    +------------------------+   |
|   |             v              |    | Helm Repository        |   |
|   |  +---------------------+  |    | (stores Helm charts)   |   |
|   |  | Kustomize Controller|  |    +------------------------+   |
|   |  | - Applies manifests |  |                                 |
|   |  | - Runs Kustomize    |  |    +------------------------+   |
|   |  +---------------------+  |    | Container Registry     |   |
|   |                           |    | (stores images)        |   |
|   |  +---------------------+  |    +------------------------+   |
|   |  | Helm Controller     |  |                                 |
|   |  | - Manages Helm      |  |                                 |
|   |  |   releases          |  |                                 |
|   |  +---------------------+  |                                 |
|   |                           |                                 |
|   |  +---------------------+  |                                 |
|   |  | Notification Ctrl   |  |                                 |
|   |  | - Sends alerts      |  |                                 |
|   |  +---------------------+  |                                 |
|   +---------------------------+                                 |
+------------------------------------------------------------------+
```

### Key CRDs

**GitRepository** — defines where to fetch source code from:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/my-app-config
  ref:
    branch: main
```

**Kustomization** — defines what to apply from the source:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  path: ./k8s
  prune: true
  sourceRef:
    kind: GitRepository
    name: my-app
  targetNamespace: production
```

---

## ArgoCD vs FluxCD

| Feature             | ArgoCD                            | FluxCD                             |
|---------------------|------------------------------------|------------------------------------|
| **CNCF Status**     | Graduated                          | Graduated                          |
| **Architecture**    | Centralized server + UI            | Distributed controllers (no UI)    |
| **Web UI**          | Yes (rich built-in UI)             | No (use Weave GitOps or CLI)       |
| **CLI**             | `argocd` CLI                       | `flux` CLI                         |
| **Multi-cluster**   | Built-in (manage multiple clusters)| Via Kustomization references       |
| **Config style**    | Application CRD                    | GitRepository + Kustomization CRDs |
| **Helm support**    | Yes (renders in-cluster)           | Yes (HelmRelease CRD)             |
| **Kustomize**       | Yes                                | Yes (native)                       |
| **Notifications**   | Built-in                           | Notification Controller            |
| **Learning curve**  | Moderate (UI helps)                | Moderate (CLI-centric)             |
| **Best for**        | Teams wanting a UI, multi-cluster  | Teams wanting lightweight, composable tooling |

```
ArgoCD:                              FluxCD:
+------------------+                 +------------------+
| Single server    |                 | Multiple small   |
| with UI + API    |                 | controllers      |
| + Controllers    |                 | (no UI by        |
|                  |                 |  default)        |
| Good for:        |                 |                  |
| - Visualization  |                 | Good for:        |
| - Multi-cluster  |                 | - Lightweight    |
| - Teams wanting  |                 | - Composable     |
|   a dashboard    |                 | - Kustomize-     |
+------------------+                 |   native         |
                                     +------------------+
```

---

## What to Remember for the Exam

1. **GitOps has four principles:** Declarative, Versioned (Git), Automatically applied, Continuously reconciled.

2. **Git is the single source of truth.** The desired state lives in Git. If the cluster drifts, the agent corrects it.

3. **GitOps uses the PULL model.** The agent inside the cluster pulls from Git. This is more secure than the push model because no cluster credentials leave the cluster.

4. **ArgoCD** is a CNCF graduated GitOps tool with a **Web UI**, **Application CRD**, and three core components: API Server, Repo Server, Application Controller.

5. **FluxCD** is a CNCF graduated GitOps tool that runs as **distributed controllers** with key CRDs: **GitRepository** (source) and **Kustomization** (what to apply).

6. **Self-healing** means the agent reverts manual changes in the cluster to match the Git-declared state.

7. **Prune** means the agent deletes resources from the cluster that have been removed from Git.

8. **Push vs Pull:** Push model = CI/CD pipeline applies to cluster (has cluster creds). Pull model = agent in cluster watches Git (more secure, built-in drift detection).

9. **Both ArgoCD and FluxCD support Helm and Kustomize** for manifest generation.

10. **Rollback in GitOps = git revert.** Since Git is the source of truth, reverting a commit triggers the agent to reconcile the cluster to the previous state.
