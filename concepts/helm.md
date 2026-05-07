# Helm

**Helm is the package manager for Kubernetes — it packages, distributes, and manages Kubernetes applications as reusable units called Charts.**

---

## Table of Contents

- [Why Helm?](#why-helm)
- [Core Concepts](#core-concepts)
- [Chart Structure](#chart-structure)
- [Helm Workflow](#helm-workflow)
- [Essential Helm Commands](#essential-helm-commands)
- [Template Engine and Values](#template-engine-and-values)
- [Helm vs Raw Manifests vs Kustomize](#helm-vs-raw-manifests-vs-kustomize)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## Why Helm?

Managing a Kubernetes application often requires many YAML files: Deployments, Services, ConfigMaps, Secrets, Ingress, etc. Helm solves several problems:

- **Packaging:** Bundle all manifests into a single unit (Chart)
- **Templating:** Use variables instead of hard-coding values
- **Versioning:** Track releases, upgrade, and rollback
- **Sharing:** Publish and consume charts from repositories

```
WITHOUT Helm:                          WITH Helm:

  deployment.yaml                      $ helm install my-app ./my-chart
  service.yaml                             |
  configmap.yaml          vs               +-- deploys ALL resources
  ingress.yaml                             +-- with customized values
  secret.yaml                              +-- tracks as a "release"
  hpa.yaml                                 +-- can upgrade / rollback
  (apply each one manually)
```

---

## Core Concepts

### 1. Chart

A **Chart** is a package of pre-configured Kubernetes resources. Think of it like an apt/yum/brew package but for Kubernetes.

- Contains templates, default values, metadata, and optional dependencies
- Can represent anything: a simple nginx server, a full database cluster, or a complex microservices app

### 2. Repository

A **Repository** is where charts are stored and shared. Similar to Docker Hub for container images.

- Examples: Artifact Hub (hub.helm.sh), Bitnami, official Helm stable repo
- Can be HTTP servers, OCI registries, or cloud storage

### 3. Release

A **Release** is a running instance of a Chart in a cluster. When you `helm install` a chart, it creates a release.

- Each release has a **name** and a **revision number**
- You can have multiple releases of the same chart (e.g., `mysql-prod` and `mysql-staging`)
- Upgrades increment the revision; rollbacks revert to a previous revision

```
+-------------------+
|      Chart        |   <-- The package (template)
|   (e.g., nginx)   |
+--------+----------+
         |
         |  helm install my-nginx ./nginx-chart
         v
+-------------------+
|     Release       |   <-- Running instance in cluster
|  name: my-nginx   |
|  revision: 1      |
+-------------------+
         |
         |  helm upgrade my-nginx ./nginx-chart
         v
+-------------------+
|     Release       |
|  name: my-nginx   |
|  revision: 2      |
+-------------------+
```

---

## Chart Structure

```
my-chart/
  |
  +-- Chart.yaml            # Metadata: name, version, description, dependencies
  +-- values.yaml            # Default configuration values
  +-- charts/                # Sub-charts (dependencies)
  +-- templates/             # Kubernetes manifest templates
  |     +-- deployment.yaml
  |     +-- service.yaml
  |     +-- ingress.yaml
  |     +-- _helpers.tpl     # Template helper functions
  |     +-- NOTES.txt        # Post-install usage notes
  +-- .helmignore            # Files to exclude from packaging
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: A Helm chart for my application
version: 0.1.0          # Chart version
appVersion: "1.0.0"     # Application version
```

### values.yaml

```yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
```

### templates/deployment.yaml (using Go templates)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

---

## Helm Workflow

```
+-----------+       +------------------+       +-----------------+
|           |       |                  |       |                 |
| Developer |       |  Chart Repo      |       |  K8s Cluster    |
|           |       |  (Artifact Hub)  |       |                 |
+-----+-----+       +--------+---------+       +--------+--------+
      |                      |                          |
      |  1. helm repo add   |                          |
      |--------------------->|                          |
      |                      |                          |
      |  2. helm search      |                          |
      |--------------------->|                          |
      |                      |                          |
      |  3. helm install     |                          |
      |-------------------------------------------->    |
      |              (renders templates +               |
      |               applies to cluster)               |
      |                                                 |
      |  4. helm upgrade     |                          |
      |-------------------------------------------->    |
      |              (updates the release)              |
      |                                                 |
      |  5. helm rollback    |                          |
      |-------------------------------------------->    |
      |              (reverts to previous revision)     |
      |                                                 |
      |  6. helm uninstall   |                          |
      |-------------------------------------------->    |
      |              (removes all release resources)    |
      |                                                 |
```

**Key point:** Helm v3 is **client-only** — there is no server-side component (Helm v2 had Tiller, which was removed for security reasons).

```
Helm v2:                           Helm v3:

+--------+     +--------+         +--------+
| helm   | --> | Tiller | --> K8s | helm   | --> K8s API
| client |     | (in-   |         | client |     directly
+--------+     | cluster)|        +--------+
               +--------+
               (REMOVED in v3)
```

---

## Essential Helm Commands

### Repository Management

```bash
# Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update repository index
helm repo update

# Search for charts
helm search repo nginx
helm search hub nginx    # Search Artifact Hub
```

### Install and Manage Releases

```bash
# Install a chart (creates a release)
helm install my-nginx bitnami/nginx

# Install with custom values
helm install my-nginx bitnami/nginx --set replicaCount=3
helm install my-nginx bitnami/nginx -f custom-values.yaml

# List releases
helm list
helm list --all-namespaces

# Get release info
helm status my-nginx
helm get values my-nginx
helm get manifest my-nginx
```

### Upgrade and Rollback

```bash
# Upgrade a release
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# View release history
helm history my-nginx

# Rollback to a specific revision
helm rollback my-nginx 1

# Uninstall a release
helm uninstall my-nginx
```

### Developing Charts

```bash
# Create a new chart scaffold
helm create my-chart

# Lint a chart for errors
helm lint ./my-chart

# Render templates locally (dry-run)
helm template my-release ./my-chart

# Install with dry-run to preview
helm install my-release ./my-chart --dry-run
```

---

## Template Engine and Values

Helm uses the **Go template language** to inject values into manifest templates.

### Value Sources (in order of precedence, highest last)

```
+---------------------+
| values.yaml         |  <-- Default values in the chart
+---------------------+
         |
         v  overridden by
+---------------------+
| -f custom.yaml      |  <-- User-provided values file
+---------------------+
         |
         v  overridden by
+---------------------+
| --set key=value     |  <-- Command-line overrides
+---------------------+
```

### Built-in Objects

| Object         | Description                              |
|----------------|------------------------------------------|
| `.Values`      | Values from values.yaml and overrides    |
| `.Release`     | Release metadata (Name, Namespace, etc.) |
| `.Chart`       | Chart.yaml metadata (Name, Version)      |
| `.Capabilities`| Cluster capabilities (API versions, K8s version) |

### Template Example

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-svc
  labels:
    app: {{ .Chart.Name }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
  selector:
    app: {{ .Chart.Name }}
```

With `values.yaml`:
```yaml
service:
  type: ClusterIP
  port: 80
```

Running `helm template my-app ./my-chart` renders:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
  labels:
    app: my-chart
spec:
  type: ClusterIP
  ports:
    - port: 80
  selector:
    app: my-chart
```

---

## Helm vs Raw Manifests vs Kustomize

| Feature              | Raw Manifests       | Helm                        | Kustomize                  |
|----------------------|---------------------|-----------------------------|----------------------------|
| **Approach**         | Plain YAML files    | Template engine + values    | Patch-based overlays       |
| **Parameterization** | None (copy & edit)  | Go templates + values.yaml  | Strategic merge patches    |
| **Packaging**        | No                  | Yes (Charts)                | No                         |
| **Versioning**       | Manual (Git)        | Built-in (revisions)        | Manual (Git)               |
| **Sharing**          | Copy files          | Chart repositories          | Copy bases/overlays        |
| **Complexity**       | Low                 | Medium                      | Low-Medium                 |
| **Rollback**         | Manual              | Built-in (helm rollback)    | Manual (Git revert)        |
| **Built into kubectl** | Yes              | No (separate binary)        | Yes (kubectl -k)           |
| **Best for**         | Simple apps         | Complex apps, sharing       | Environment variations     |

```
Raw Manifests:        Helm:                    Kustomize:

deployment.yaml       Chart (template)         base/
service.yaml    vs    + values.yaml      vs      deployment.yaml
configmap.yaml        = rendered manifests       service.yaml
                                               overlays/
                                                 prod/
                                                   patch.yaml
                                                 staging/
                                                   patch.yaml
```

---

## What to Remember for the Exam

1. **Helm is the Kubernetes package manager.** It packages K8s resources into **Charts**, stores them in **Repositories**, and deploys them as **Releases**.

2. **Chart** = package of templates + default values. **Release** = a deployed instance of a chart. **Repository** = where charts are stored.

3. **Helm v3 has no Tiller.** It talks directly to the Kubernetes API. Tiller was removed due to security concerns (it had broad cluster permissions).

4. **values.yaml** provides default configuration. Users override with `-f custom-values.yaml` or `--set key=value`.

5. **Key commands:** `helm install` (deploy), `helm upgrade` (update), `helm rollback` (revert), `helm list` (show releases), `helm uninstall` (remove).

6. **Chart structure:** `Chart.yaml` (metadata), `values.yaml` (defaults), `templates/` (K8s manifest templates).

7. **Helm uses Go templates** with built-in objects: `.Values`, `.Release`, `.Chart`.

8. **Helm vs Kustomize:** Helm uses templates and packaging; Kustomize uses patches and overlays. Kustomize is built into kubectl (`kubectl apply -k`). They can be used together.
