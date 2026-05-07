# The Kubernetes API

**The Kubernetes API is the central RESTful interface through which all cluster interactions occur, accessed primarily through kubectl, and organized into versioned API groups.**

---

## Table of Contents

- [API Server as Central Interface](#api-server-as-central-interface)
- [RESTful Model](#restful-model)
- [API Groups and Versioning](#api-groups-and-versioning)
- [Resource URLs](#resource-urls)
- [kubectl — The CLI Client](#kubectl--the-cli-client)
- [Declarative vs Imperative Approaches](#declarative-vs-imperative-approaches)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## API Server as Central Interface

Every interaction with Kubernetes goes through the API server:

```
  +----------+    +----------+    +-----------+    +----------+
  | kubectl   |    | Dashboard|    | CI/CD     |    | Custom   |
  |           |    |          |    | Pipelines |    | Clients  |
  +-----+----+    +-----+----+    +-----+-----+    +-----+----+
        |              |              |                  |
        +--------------+--------------+------------------+
                                |
                          HTTPS (6443)
                                |
                                v
                    +-----------+----------+
                    |    kube-apiserver      |
                    |                       |
                    |  AuthN -> AuthZ ->    |
                    |  Admission -> etcd    |
                    +-----------+----------+
                                |
              +-----------------+------------------+
              |                 |                   |
              v                 v                   v
          +------+      +----------+        +--------+
          | etcd  |      | scheduler |        | kubelet |
          +------+      +----------+        +--------+
```

---

## RESTful Model

The Kubernetes API follows REST principles. Every resource is an **object** that can be created, read, updated, or deleted using standard HTTP methods:

| HTTP Method | kubectl Equivalent    | Description                     |
|-------------|----------------------|---------------------------------|
| GET         | `kubectl get`        | List or retrieve resources      |
| POST        | `kubectl create`     | Create a new resource           |
| PUT         | `kubectl replace`    | Replace a resource entirely     |
| PATCH       | `kubectl patch`/`apply` | Partially update a resource  |
| DELETE      | `kubectl delete`     | Delete a resource               |

### Request/Response Format

```
  Request:
  GET /api/v1/namespaces/default/pods/nginx HTTP/1.1
  Authorization: Bearer <token>
  Accept: application/json

  Response:
  200 OK
  Content-Type: application/json
  {
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
      "name": "nginx",
      "namespace": "default"
    },
    "spec": { ... },
    "status": { ... }
  }
```

---

## API Groups and Versioning

Kubernetes organizes resources into **API groups** with independent versioning. This allows the API to evolve without breaking backward compatibility.

### Core API Group (Legacy)

Resources in the core group use `/api/v1` (no group name in the URL):

```
  /api/v1/pods
  /api/v1/services
  /api/v1/configmaps
  /api/v1/secrets
  /api/v1/namespaces
  /api/v1/nodes
  /api/v1/persistentvolumes
  /api/v1/serviceaccounts
```

### Named API Groups

Other resources use `/apis/<group>/<version>`:

```
  /apis/apps/v1/deployments
  /apis/apps/v1/replicasets
  /apis/apps/v1/statefulsets
  /apis/apps/v1/daemonsets
  /apis/batch/v1/jobs
  /apis/batch/v1/cronjobs
  /apis/networking.k8s.io/v1/networkpolicies
  /apis/networking.k8s.io/v1/ingresses
  /apis/rbac.authorization.k8s.io/v1/roles
  /apis/rbac.authorization.k8s.io/v1/clusterroles
  /apis/storage.k8s.io/v1/storageclasses
```

### API Version Stages

| Stage           | Meaning                                              |
|-----------------|------------------------------------------------------|
| **v1**          | Stable, generally available (GA), production-ready   |
| **v1beta1**     | Beta — features mostly complete, may change          |
| **v1alpha1**    | Alpha — experimental, may be removed, off by default |

### Common apiVersion Values in YAML

```yaml
# Core resources
apiVersion: v1          # Pod, Service, ConfigMap, Secret, Namespace

# Apps group
apiVersion: apps/v1     # Deployment, ReplicaSet, StatefulSet, DaemonSet

# Batch group
apiVersion: batch/v1    # Job, CronJob

# Networking group
apiVersion: networking.k8s.io/v1  # Ingress, NetworkPolicy

# RBAC group
apiVersion: rbac.authorization.k8s.io/v1  # Role, ClusterRole, RoleBinding
```

### API Groups Diagram

```
  Kubernetes API
       |
       +-- /api/v1  (core group)
       |      |-- pods
       |      |-- services
       |      |-- configmaps
       |      |-- secrets
       |      |-- namespaces
       |      |-- nodes
       |
       +-- /apis/apps/v1
       |      |-- deployments
       |      |-- replicasets
       |      |-- statefulsets
       |      |-- daemonsets
       |
       +-- /apis/batch/v1
       |      |-- jobs
       |      |-- cronjobs
       |
       +-- /apis/networking.k8s.io/v1
       |      |-- ingresses
       |      |-- networkpolicies
       |
       +-- /apis/rbac.authorization.k8s.io/v1
              |-- roles
              |-- clusterroles
              |-- rolebindings
              |-- clusterrolebindings
```

---

## Resource URLs

Kubernetes API URLs follow a consistent pattern:

### Cluster-Scoped Resources (no namespace)

```
  /api/v1/nodes
  /api/v1/nodes/{name}
  /api/v1/namespaces
  /api/v1/persistentvolumes
```

### Namespace-Scoped Resources

```
  /api/v1/namespaces/{namespace}/pods
  /api/v1/namespaces/{namespace}/pods/{name}
  /api/v1/namespaces/{namespace}/services/{name}
  /apis/apps/v1/namespaces/{namespace}/deployments/{name}
```

### URL Structure Pattern

```
  /api/v1/namespaces/default/pods/nginx
   |    |     |         |      |     |
   |    |     |         |      |     +-- Resource name
   |    |     |         |      +-------- Resource type
   |    |     |         +--------------- Namespace
   |    |     +------------------------- (indicates namespaced)
   |    +------------------------------- API version
   +------------------------------------ API prefix
```

### Subresources

Some resources have subresources accessed via additional path segments:

```
  /api/v1/namespaces/default/pods/nginx/log        # Pod logs
  /api/v1/namespaces/default/pods/nginx/exec       # Exec into Pod
  /api/v1/namespaces/default/pods/nginx/status     # Pod status
  /apis/apps/v1/namespaces/default/deployments/web/scale   # Scale
```

---

## kubectl -- The CLI Client

kubectl is the command-line tool for interacting with the Kubernetes API server. It translates your commands into API requests.

```
  kubectl get pods
      |
      v
  GET /api/v1/namespaces/default/pods
  Authorization: Bearer <token from ~/.kube/config>
      |
      v
  kube-apiserver returns JSON
      |
      v
  kubectl formats and displays as a table
```

### Essential kubectl Commands

#### Viewing Resources

```bash
# List resources
kubectl get pods                        # List pods in default namespace
kubectl get pods -n kube-system         # List pods in a specific namespace
kubectl get pods --all-namespaces       # List pods across all namespaces
kubectl get pods -A                     # Short form of --all-namespaces
kubectl get pods -o wide                # Show more details (node, IP)
kubectl get pods -o yaml                # Output as YAML
kubectl get pods -o json                # Output as JSON
kubectl get all                         # List common resources

# Detailed information
kubectl describe pod nginx              # Detailed info about a pod
kubectl describe node worker-1          # Detailed info about a node

# Discover API resources
kubectl api-resources                   # List all available resource types
kubectl api-versions                    # List all available API versions
kubectl explain pod                     # Documentation for a resource type
kubectl explain pod.spec.containers     # Documentation for nested fields
```

#### Creating and Managing Resources

```bash
# Imperative creation
kubectl run nginx --image=nginx                        # Create a pod
kubectl create deployment web --image=nginx            # Create a deployment
kubectl create namespace dev                           # Create a namespace
kubectl create configmap my-config --from-literal=key=value

# Declarative creation/updates
kubectl apply -f pod.yaml                              # Create or update
kubectl apply -f ./manifests/                          # Apply a directory

# Deleting resources
kubectl delete pod nginx                               # Delete by name
kubectl delete -f pod.yaml                             # Delete by manifest
kubectl delete pods --all                              # Delete all pods
```

#### Debugging and Troubleshooting

```bash
# View logs
kubectl logs nginx                      # Logs from a pod
kubectl logs nginx -c sidecar           # Logs from specific container
kubectl logs nginx --previous           # Logs from previous instance
kubectl logs nginx -f                   # Follow (stream) logs
kubectl logs -l app=web                 # Logs by label selector

# Execute commands in containers
kubectl exec nginx -- ls /              # Run a command
kubectl exec -it nginx -- /bin/sh       # Interactive shell

# Port forwarding
kubectl port-forward pod/nginx 8080:80  # Forward local:container port
kubectl port-forward svc/web 8080:80    # Forward to a service

# Copying files
kubectl cp nginx:/tmp/file.txt ./file.txt   # Copy from pod
kubectl cp ./file.txt nginx:/tmp/file.txt   # Copy to pod
```

### Output Formats

| Flag             | Format                                |
|------------------|---------------------------------------|
| (default)        | Human-readable table                  |
| `-o wide`        | Table with extra columns              |
| `-o yaml`        | YAML format                           |
| `-o json`        | JSON format                           |
| `-o name`        | Resource name only                    |
| `-o jsonpath='{}'` | Custom JSONPath expression          |
| `-o custom-columns=...` | Custom table columns            |

### kubeconfig

kubectl uses a configuration file (usually `~/.kube/config`) to know which cluster to talk to:

```
  ~/.kube/config
  +----------------------------------+
  | clusters:                        |  <-- API server addresses
  |   - name: my-cluster             |
  |     server: https://10.0.0.1:6443|
  |                                  |
  | users:                           |  <-- Credentials
  |   - name: admin                  |
  |     client-certificate: ...      |
  |                                  |
  | contexts:                        |  <-- Cluster + User combos
  |   - name: admin@my-cluster       |
  |     cluster: my-cluster          |
  |     user: admin                  |
  |     namespace: default           |
  |                                  |
  | current-context: admin@my-cluster|  <-- Active context
  +----------------------------------+
```

Useful context commands:

```bash
kubectl config get-contexts              # List all contexts
kubectl config current-context           # Show active context
kubectl config use-context <name>        # Switch context
kubectl config set-context --current --namespace=dev  # Set default namespace
```

---

## Declarative vs Imperative Approaches

### Imperative: Tell Kubernetes WHAT TO DO

```bash
# Direct commands — actions happen immediately
kubectl run nginx --image=nginx
kubectl create deployment web --image=nginx --replicas=3
kubectl scale deployment web --replicas=5
kubectl set image deployment/web nginx=nginx:1.25
kubectl delete pod nginx
```

- Quick for one-off tasks and debugging
- No record of what you did
- Hard to reproduce
- Not suitable for production workflows

### Declarative: Tell Kubernetes WHAT YOU WANT

```yaml
# deployment.yaml — describes the desired state
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

```bash
# Apply the desired state — Kubernetes figures out what to do
kubectl apply -f deployment.yaml
```

- Repeatable and version-controllable (Git)
- Self-documenting (the YAML IS the documentation)
- Supports GitOps workflows
- **Recommended for production**

### Comparison

```
  IMPERATIVE                          DECLARATIVE
  ===============                     ===============
  "Create 3 nginx pods"               "I want 3 nginx pods running"

  kubectl create deployment           kubectl apply -f deploy.yaml
  nginx --replicas=3

  You tell K8s the steps              You tell K8s the end state
  K8s executes once                   K8s continuously reconciles
  No audit trail                      YAML in Git = audit trail
  Good for debugging                  Good for production
```

### Dry Run and Diff

```bash
# Preview what would happen without making changes
kubectl apply -f deployment.yaml --dry-run=client    # Client-side validation
kubectl apply -f deployment.yaml --dry-run=server    # Server-side validation

# Show differences between local file and live cluster state
kubectl diff -f deployment.yaml

# Generate YAML without creating the resource
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
```

---

## What to Remember for the Exam

1. **API server is the only entry point** to the cluster. All communication (kubectl, kubelets, controllers, dashboards) goes through it.

2. **RESTful model**: Resources are objects manipulated with standard HTTP methods (GET, POST, PUT, PATCH, DELETE).

3. **API Groups**: Core resources use `/api/v1`; others use `/apis/<group>/<version>`. Know the common groups: `apps/v1`, `batch/v1`, `networking.k8s.io/v1`.

4. **Version stages**: v1 = stable/GA, v1beta1 = beta, v1alpha1 = alpha/experimental.

5. **apiVersion in YAML**: `v1` for Pods/Services/ConfigMaps, `apps/v1` for Deployments/StatefulSets/DaemonSets, `batch/v1` for Jobs/CronJobs.

6. **kubectl is the primary CLI client**. Key commands: get, describe, create, apply, delete, logs, exec.

7. **Declarative (apply) is preferred** over imperative (create/run) for production workloads because it is repeatable, version-controllable, and self-documenting.

8. **kubectl explain** is invaluable for exploring resource specifications: `kubectl explain pod.spec.containers`.

9. **kubeconfig** (typically `~/.kube/config`) contains clusters, users, and contexts. The current-context determines which cluster kubectl talks to.

10. **Output formats**: `-o yaml`, `-o json`, `-o wide`, `-o jsonpath` are essential for querying and debugging.

11. **Dry run**: Use `--dry-run=client -o yaml` to generate YAML templates without creating resources.
