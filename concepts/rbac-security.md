# RBAC and Security

> Role-Based Access Control (RBAC) governs who can do what in a Kubernetes cluster. Combined with Pod Security Standards and policy engines, it forms the security foundation.

---

## Authentication vs Authorization

These are two distinct steps in the Kubernetes API request flow:

```
  API Request Flow

  User / ServiceAccount
       │
       ▼
  ┌──────────────────┐
  │  1. AUTHENTICATION│    "Who are you?"
  │     (AuthN)       │    Verify identity
  └────────┬─────────┘
           │ Identity confirmed
           ▼
  ┌──────────────────┐
  │  2. AUTHORIZATION │    "What can you do?"
  │     (AuthZ - RBAC)│    Check permissions
  └────────┬─────────┘
           │ Allowed
           ▼
  ┌──────────────────┐
  │  3. ADMISSION    │    "Is this request valid?"
  │     CONTROL      │    Mutate or reject
  └────────┬─────────┘
           │ Accepted
           ▼
  ┌──────────────────┐
  │  4. API SERVER   │    Process the request
  │     (etcd)       │    Store/retrieve data
  └──────────────────┘
```

### Authentication Methods

| Method | How It Works | Used By |
|--------|-------------|---------|
| **X.509 Client Certificates** | Client presents a TLS certificate signed by the cluster CA | kubectl (kubeconfig), kubelet |
| **Bearer Tokens** | Token passed in HTTP Authorization header | ServiceAccounts, bootstrap tokens |
| **OIDC (OpenID Connect)** | Delegated to an external identity provider (Google, Azure AD, Okta) | Human users in enterprise setups |
| **Webhook Token Auth** | API server calls an external service to validate the token | Custom auth systems |

### Authorization Modes

| Mode | Description |
|------|------------|
| **RBAC** | Role-based, most common, recommended |
| **ABAC** | Attribute-based, requires file edits, legacy |
| **Webhook** | External service decides, flexible |
| **Node** | Special authorizer for kubelet |

**RBAC is the default and recommended authorization mode.** It is the focus of the KCNA exam.

---

## RBAC — Role-Based Access Control

RBAC uses four Kubernetes objects to define permissions:

```
  RBAC Object Model

  Namespace-scoped:              Cluster-scoped:
  ┌───────────────┐              ┌───────────────────┐
  │     Role      │              │   ClusterRole     │
  │               │              │                   │
  │ Defines what  │              │ Defines what      │
  │ actions are   │              │ actions are       │
  │ allowed on    │              │ allowed on        │
  │ which         │              │ which resources   │
  │ resources     │              │ (cluster-wide)    │
  └───────┬───────┘              └─────────┬─────────┘
          │                                │
          │ bound by                       │ bound by
          ▼                                ▼
  ┌───────────────┐              ┌───────────────────┐
  │ RoleBinding   │              │ClusterRoleBinding │
  │               │              │                   │
  │ Binds a Role  │              │ Binds a           │
  │ to a user,    │              │ ClusterRole to a  │
  │ group, or SA  │              │ user, group, or   │
  │ (within a     │              │ SA (cluster-wide) │
  │  namespace)   │              │                   │
  └───────────────┘              └───────────────────┘
```

### Role (Namespace-Scoped)

Defines **what** actions are allowed on **which** resources within a namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""]           # "" = core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

### ClusterRole (Cluster-Scoped)

Same as Role, but applies across **all namespaces** or to cluster-scoped resources (nodes, PVs):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

### RoleBinding (Namespace-Scoped)

Binds a Role (or ClusterRole) to a subject within a specific namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: development
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding (Cluster-Scoped)

Binds a ClusterRole to a subject across the entire cluster:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-binding
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### RBAC Flow Diagram

```
  RBAC Decision Flow

  Request: "Can user jane GET pods in namespace development?"

  ┌──────────┐     ┌───────────────┐     ┌──────────────┐
  │  jane    │────►│ RoleBinding   │────►│    Role      │
  │ (User)   │     │ "read-pods-   │     │ "pod-reader" │
  │          │     │  binding"     │     │              │
  │          │     │               │     │ resources:   │
  │          │     │ namespace:    │     │  - pods      │
  │          │     │  development  │     │ verbs:       │
  │          │     │               │     │  - get       │
  │          │     │ subject: jane │     │  - list      │
  │          │     │ roleRef:      │     │  - watch     │
  │          │     │  pod-reader   │     │              │
  └──────────┘     └───────────────┘     └──────────────┘

  Result: ALLOWED (jane has GET permission for pods in development)
```

### Common RBAC Verbs

| Verb | HTTP Method | Description |
|------|------------|-------------|
| `get` | GET (single) | Read a specific resource |
| `list` | GET (collection) | List all resources |
| `watch` | GET (streaming) | Watch for changes |
| `create` | POST | Create a resource |
| `update` | PUT | Update a resource |
| `patch` | PATCH | Partially update |
| `delete` | DELETE | Delete a resource |
| `deletecollection` | DELETE | Delete all resources |

### ServiceAccounts

Pods use **ServiceAccounts** for authentication. Every namespace has a `default` ServiceAccount.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
```

Bind it to a role:
```yaml
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: production
```

---

## Pod Security Standards

Pod Security Standards define three levels of restriction for pod configurations:

```
  Pod Security Standards

  ┌────────────────────────────────────────────────────────────┐
  │                                                            │
  │  Privileged          Baseline            Restricted        │
  │  ┌──────────┐       ┌──────────┐        ┌──────────┐     │
  │  │          │       │          │        │          │     │
  │  │ No       │       │ Prevents │        │ Heavily  │     │
  │  │ restric- │       │ known    │        │ restrict-│     │
  │  │ tions    │       │ privilege│        │ ed, best │     │
  │  │          │       │ escala-  │        │ practices│     │
  │  │ (system  │       │ tions    │        │          │     │
  │  │  compo-  │       │          │        │ (hardened│     │
  │  │  nents)  │       │ (default │        │  work-   │     │
  │  │          │       │  for most│        │  loads)  │     │
  │  │          │       │  work-   │        │          │     │
  │  │          │       │  loads)  │        │          │     │
  │  └──────────┘       └──────────┘        └──────────┘     │
  │                                                            │
  │  Least secure ◄──────────────────────► Most secure        │
  │                                                            │
  └────────────────────────────────────────────────────────────┘
```

| Level | What It Allows | Use Case |
|-------|---------------|----------|
| **Privileged** | Everything — no restrictions | System components (kube-system) |
| **Baseline** | Prevents known privilege escalations, no hostNetwork, no privileged containers | Most application workloads |
| **Restricted** | Everything in Baseline + must run as non-root, drop all capabilities, read-only root filesystem | Hardened, security-critical workloads |

Pod Security Standards are enforced via the **Pod Security Admission** controller (built-in since K8s 1.25), applied per namespace using labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

Three modes:
- **enforce** — reject pods that violate
- **warn** — allow but show a warning
- **audit** — allow but log the violation

---

## SecurityContext

SecurityContext configures security settings at the **pod** or **container** level:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:              # Pod-level
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: nginx
    securityContext:            # Container-level (overrides pod-level)
      runAsNonRoot: true
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
```

Key SecurityContext fields:
- `runAsUser` / `runAsGroup` — run as a specific UID/GID
- `runAsNonRoot` — refuse to start if running as root
- `readOnlyRootFilesystem` — prevent writes to the container filesystem
- `allowPrivilegeEscalation` — prevent gaining more privileges than the parent process
- `capabilities` — add or drop Linux capabilities

---

## Open Policy Agent (OPA) / Gatekeeper

OPA is a **general-purpose policy engine**. Gatekeeper integrates OPA into Kubernetes as an admission controller.

```
  OPA Gatekeeper Flow

  ┌──────────┐     ┌──────────────┐     ┌─────────────────┐
  │  kubectl │     │  API Server  │     │   Gatekeeper    │
  │  apply   │────►│              │────►│  (Admission     │
  │          │     │  Admission   │     │   Webhook)      │
  └──────────┘     │  Webhook     │     │                 │
                   └──────────────┘     │  Evaluates      │
                                        │  OPA policies   │
                                        │  (Rego lang)    │
                                        │                 │
                                        │  ALLOW / DENY   │
                                        └─────────────────┘

  Example policies:
  - All images must come from approved registries
  - All pods must have resource limits
  - No containers may run as root
  - All resources must have required labels
```

OPA Gatekeeper uses two custom resources:
- **ConstraintTemplate** — defines the policy logic (written in Rego)
- **Constraint** — applies the template to specific resources

For the KCNA exam, know that OPA/Gatekeeper exists for **policy enforcement** and works as an **admission controller**. You do not need to write Rego policies.

---

## Key Exam Points

- **Authentication** = proving identity (certs, tokens, OIDC). **Authorization** = checking permissions (RBAC).
- RBAC has four objects: **Role**, **ClusterRole**, **RoleBinding**, **ClusterRoleBinding**.
- Role/RoleBinding are **namespace-scoped**. ClusterRole/ClusterRoleBinding are **cluster-scoped**.
- A **RoleBinding** can reference a ClusterRole (grants ClusterRole permissions only within that namespace).
- **ServiceAccounts** are used by pods for authentication. Every namespace has a `default` SA.
- **Pod Security Standards**: Privileged (no restrictions), Baseline (reasonable defaults), Restricted (hardened).
- Pod Security Admission has three modes: enforce, warn, audit.
- **SecurityContext** controls security at pod and container level (runAsUser, capabilities, readOnlyRootFilesystem).
- **OPA/Gatekeeper** provides custom policy enforcement as an admission controller.

---

## What to Remember for the Exam

1. **RBAC is the standard** authorization mode. Know the four objects and whether they are namespaced or cluster-scoped.
2. **Verbs** — get, list, watch, create, update, patch, delete. These map to HTTP methods.
3. **Pod Security Standards** — three levels (Privileged, Baseline, Restricted). Applied to namespaces via labels.
4. **SecurityContext** — pod-level and container-level settings. Key fields: runAsNonRoot, readOnlyRootFilesystem, allowPrivilegeEscalation, capabilities.
5. **ServiceAccounts** — each pod authenticates using a ServiceAccount. Default SA is created automatically.
6. **OPA/Gatekeeper** — external policy engine, admission webhook, written in Rego language.
7. **API request flow**: Authentication --> Authorization (RBAC) --> Admission Control --> etcd.
