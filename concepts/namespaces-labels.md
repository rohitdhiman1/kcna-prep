# Namespaces, Labels, and Annotations

**Namespaces provide logical isolation for resources within a cluster, while labels and selectors enable flexible grouping and filtering, and annotations store non-identifying metadata.**

---

## Table of Contents

- [Namespaces](#namespaces)
- [Default Namespaces](#default-namespaces)
- [Labels and Selectors](#labels-and-selectors)
- [Annotations](#annotations)
- [Labels vs Annotations](#labels-vs-annotations)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## Namespaces

Namespaces are a way to divide cluster resources between multiple users, teams, or environments.

```
  +===========================================================+
  |                    KUBERNETES CLUSTER                      |
  |                                                           |
  |  +------------------+  +------------------+               |
  |  | Namespace: dev   |  | Namespace: prod  |               |
  |  |                  |  |                  |               |
  |  | Deployment: web  |  | Deployment: web  |  Same name,  |
  |  | Service: web-svc |  | Service: web-svc |  different   |
  |  | ConfigMap: config|  | ConfigMap: config|  namespaces!  |
  |  +------------------+  +------------------+               |
  |                                                           |
  |  +------------------+  +------------------+               |
  |  | Namespace:       |  | Namespace:       |               |
  |  | kube-system      |  | kube-public      |               |
  |  |                  |  |                  |               |
  |  | CoreDNS          |  | (public info)    |               |
  |  | kube-proxy       |  |                  |               |
  |  | etcd, apiserver  |  |                  |               |
  |  +------------------+  +------------------+               |
  +===========================================================+
```

### What Namespaces Provide

- **Name scoping** — Resources can have the same name in different namespaces
- **Access control** — RBAC policies can be scoped to a namespace
- **Resource quotas** — Limit CPU, memory, and object counts per namespace
- **Organization** — Separate environments, teams, or projects

### What Namespaces Do NOT Provide

- **Network isolation** — Pods in different namespaces can communicate by default (use NetworkPolicies for isolation)
- **Node isolation** — Pods from different namespaces can run on the same node

### Namespace-Scoped vs Cluster-Scoped Resources

```
  Namespace-Scoped:              Cluster-Scoped:
  (exist within a namespace)     (exist across the whole cluster)

  - Pods                         - Nodes
  - Services                     - Namespaces themselves
  - Deployments                  - PersistentVolumes
  - ConfigMaps                   - ClusterRoles
  - Secrets                      - ClusterRoleBindings
  - ReplicaSets                  - StorageClasses
  - Jobs / CronJobs              - CustomResourceDefinitions
  - Roles / RoleBindings
  - ServiceAccounts
```

```bash
# List all namespace-scoped resources
kubectl api-resources --namespaced=true

# List all cluster-scoped resources
kubectl api-resources --namespaced=false
```

---

## Default Namespaces

Every Kubernetes cluster comes with four built-in namespaces:

| Namespace          | Purpose                                                        |
|--------------------|----------------------------------------------------------------|
| **default**        | Where resources go if no namespace is specified                |
| **kube-system**    | System components (API server, scheduler, CoreDNS, kube-proxy) |
| **kube-public**    | Publicly accessible data (cluster-info ConfigMap)              |
| **kube-node-lease**| Node heartbeat leases for efficient node health detection      |

```
  +-- default          <-- Your resources go here by default
  |
  +-- kube-system      <-- Control plane & system Pods
  |     |-- coredns-xxxxx
  |     |-- etcd-master
  |     |-- kube-apiserver-master
  |     |-- kube-controller-manager-master
  |     |-- kube-scheduler-master
  |     |-- kube-proxy-xxxxx (one per node)
  |
  +-- kube-public      <-- Readable by everyone (even unauthenticated)
  |     |-- cluster-info ConfigMap
  |
  +-- kube-node-lease  <-- Lease objects for node heartbeats
        |-- node-lease-worker-1
        |-- node-lease-worker-2
```

### Working with Namespaces

```bash
# List namespaces
kubectl get namespaces
kubectl get ns

# Create a namespace
kubectl create namespace dev

# Create a namespace from YAML
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: staging
EOF

# Deploy to a specific namespace
kubectl run nginx --image=nginx -n dev
kubectl apply -f deployment.yaml -n dev

# List resources in a specific namespace
kubectl get pods -n dev
kubectl get all -n kube-system

# List resources across ALL namespaces
kubectl get pods -A
kubectl get pods --all-namespaces

# Set default namespace for kubectl
kubectl config set-context --current --namespace=dev

# Delete a namespace (DELETES ALL RESOURCES IN IT)
kubectl delete namespace dev
```

---

## Labels and Selectors

Labels are **key-value pairs** attached to objects that are used for identification and selection.

### Label Format

```
  metadata:
    labels:
      app: web               # identifying the application
      tier: frontend          # identifying the tier
      env: production         # identifying the environment
      version: v2.1.0         # identifying the version
      team: platform          # identifying the owning team

  Rules:
  - Key: max 63 chars (or prefix/name, prefix max 253 chars)
  - Value: max 63 chars, alphanumeric, -, _, .
  - Must begin and end with alphanumeric character
```

### How Labels Connect Resources

```
  Deployment                        Service
  selector:                         selector:
    matchLabels:                      app: web
      app: web                        tier: frontend
      tier: frontend
       |                               |
       +---------- both select --------+
                       |
                       v
            +--------------------+
            | Pod                |
            | labels:            |
            |   app: web         |  <-- matches both selectors
            |   tier: frontend   |
            |   version: v2      |
            +--------------------+
```

### Selector Types

#### 1. Equality-Based Selectors

```bash
# Select Pods where app equals "web"
kubectl get pods -l app=web

# Select Pods where tier does not equal "backend"
kubectl get pods -l tier!=backend

# Multiple conditions (AND logic)
kubectl get pods -l app=web,tier=frontend
```

Operators: `=`, `==`, `!=`

#### 2. Set-Based Selectors

```bash
# Select where env is in the set {production, staging}
kubectl get pods -l 'env in (production, staging)'

# Select where tier is NOT in the set {backend}
kubectl get pods -l 'tier notin (backend)'

# Select Pods that HAVE the label "app" (any value)
kubectl get pods -l 'app'

# Select Pods that do NOT have the label "canary"
kubectl get pods -l '!canary'
```

Operators: `in`, `notin`, `exists`, `!exists`

### Labels in YAML (Selectors)

```yaml
# Equality-based (used by Services, ReplicationControllers)
selector:
  app: web
  tier: frontend

# Set-based (used by Deployments, ReplicaSets, Jobs)
selector:
  matchLabels:
    app: web
  matchExpressions:
  - key: tier
    operator: In
    values: [frontend, backend]
  - key: env
    operator: NotIn
    values: [test]
```

### Managing Labels

```bash
# Add a label
kubectl label pod nginx env=production

# Update an existing label (requires --overwrite)
kubectl label pod nginx env=staging --overwrite

# Remove a label (use key with a minus sign)
kubectl label pod nginx env-

# Show labels in output
kubectl get pods --show-labels

# Add a column for specific label
kubectl get pods -L app,env
```

---

## Annotations

Annotations are **key-value pairs** that store **non-identifying metadata**. They are NOT used for selection.

### Annotation Format

```yaml
metadata:
  annotations:
    description: "Main web application frontend"
    build.version: "git-abc123def"
    contact: "team-platform@company.com"
    config.kubernetes.io/managed-by: "argocd"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
```

### Common Use Cases

| Use Case                    | Example Annotation                              |
|-----------------------------|-------------------------------------------------|
| Build/release information   | `build.version: "abc123"`                       |
| Contact/ownership info      | `owner: "team-platform"`                        |
| Tool configuration          | `prometheus.io/scrape: "true"`                  |
| Deployment tool tracking    | `config.kubernetes.io/managed-by: "argocd"`     |
| Descriptions                | `description: "User-facing API gateway"`        |
| Last applied configuration  | `kubectl.kubernetes.io/last-applied-configuration` |

### Annotations vs Labels

Annotations can contain:
- Arbitrary strings (no length limit on values like labels have)
- Structured data (JSON, URLs, multi-line text)
- Non-identifying information

### Managing Annotations

```bash
# Add an annotation
kubectl annotate pod nginx description="Web server pod"

# Update an annotation
kubectl annotate pod nginx description="Updated description" --overwrite

# Remove an annotation
kubectl annotate pod nginx description-
```

---

## Labels vs Annotations

```
  +-------------------+----------------------------------------+
  |                   |  LABELS            | ANNOTATIONS        |
  +-------------------+--------------------+--------------------+
  | Purpose           | Identify & select  | Store metadata     |
  | Used by selectors | YES                | NO                 |
  | Used by Services  | YES (to find Pods) | NO                 |
  | Used by k8s core  | YES (scheduling,   | Some (last-applied)|
  |                   |  grouping)         |                    |
  | Value constraints | Max 63 chars       | No practical limit |
  | Examples          | app=web, env=prod  | git-sha, owner,    |
  |                   | tier=frontend      | description, URL   |
  +-------------------+--------------------+--------------------+
```

### Rule of Thumb

- **Labels**: Use when you need to **select or filter** resources. Use for anything that Kubernetes controllers, Services, or schedulers need to match on.
- **Annotations**: Use for **everything else** — metadata that humans or external tools need but Kubernetes itself does not use for selection.

```
  Will Kubernetes use this to select/group/match?
       |
       +-- YES --> Use a LABEL
       |
       +-- NO  --> Use an ANNOTATION
```

---

## What to Remember for the Exam

1. **Namespaces** provide logical separation within a cluster. Resources can have the same name in different namespaces.

2. **Four default namespaces**: `default` (your resources), `kube-system` (system components), `kube-public` (publicly readable), `kube-node-lease` (node heartbeats).

3. **Namespaces do NOT provide network isolation** by default. Use NetworkPolicies for that.

4. **Deleting a namespace deletes all resources** within it.

5. **Labels** are key-value pairs used for **identification and selection**. Services, Deployments, and other controllers use labels to find their target Pods.

6. **Two selector types**: Equality-based (`app=web`, `env!=test`) and set-based (`env in (prod, staging)`, `!canary`).

7. **Annotations** store non-identifying metadata. They are NOT used for selection. No strict size limits on values.

8. **Labels vs Annotations**: If Kubernetes needs to select/match on it, use a label. For everything else, use an annotation.

9. **kubectl -l** for filtering by labels: `kubectl get pods -l app=web,env=prod`.

10. **--show-labels** flag shows all labels in output. **-L label-key** adds a specific label as a column.

11. **Some resources are cluster-scoped** (Nodes, Namespaces, PVs, ClusterRoles) and do not belong to any namespace.
