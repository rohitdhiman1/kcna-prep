# Lab 25: Helm

## Objective

Install Helm, add a chart repository, install a chart, customize it with values, upgrade a release, roll back, and uninstall.

## Prerequisites

- A running Kubernetes cluster (kind, minikube, or cloud-managed)
- `kubectl` installed and configured
- Internet access (to download Helm and charts)

---

## Exercise 1: Install Helm

### Step 1: Install Helm CLI

**macOS (Homebrew):**

```bash
brew install helm
```

**Linux (script):**

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Windows (Chocolatey):**

```bash
choco install kubernetes-helm
```

### Step 2: Verify the installation

```bash
helm version
```

You should see output like:

```
version.BuildInfo{Version:"v3.x.x", ...}
```

### Step 3: Confirm Helm can talk to your cluster

```bash
helm list
```

This should return an empty table (no releases yet). If it errors, your `kubectl` context may not be set correctly.

---

## Exercise 2: Add a Chart Repository

### Step 1: Add the Bitnami repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

### Step 2: Update the repository index

```bash
helm repo update
```

### Step 3: Search for charts

```bash
helm search repo nginx
```

You should see `bitnami/nginx` listed with its version and description.

### Step 4: View chart details

```bash
helm show chart bitnami/nginx
```

This displays the `Chart.yaml` metadata.

### Step 5: View default values

```bash
helm show values bitnami/nginx
```

This shows all configurable values and their defaults. This output can be long — you can pipe it to `less`:

```bash
helm show values bitnami/nginx | less
```

---

## Exercise 3: Install a Chart

### Step 1: Install nginx with default values

```bash
helm install my-nginx bitnami/nginx
```

Helm will output release notes with instructions on how to access the application.

### Step 2: Verify the release

```bash
helm list
```

You should see `my-nginx` with status `deployed` and revision `1`.

### Step 3: Check the Kubernetes resources created

```bash
kubectl get all -l app.kubernetes.io/instance=my-nginx
```

You should see a Deployment, ReplicaSet, Pod(s), and Service.

### Step 4: Check release status

```bash
helm status my-nginx
```

---

## Exercise 4: Customize with Values

### Step 1: Upgrade with inline value overrides

Change the replica count:

```bash
helm upgrade my-nginx bitnami/nginx --set replicaCount=3
```

### Step 2: Verify the upgrade

```bash
helm list
```

Revision should now be `2`.

```bash
kubectl get pods -l app.kubernetes.io/instance=my-nginx
```

You should see 3 pods.

### Step 3: Create a custom values file

```bash
cat <<EOF > custom-values.yaml
replicaCount: 2
service:
  type: ClusterIP
EOF
```

### Step 4: Upgrade using the values file

```bash
helm upgrade my-nginx bitnami/nginx -f custom-values.yaml
```

### Step 5: Verify the values in use

```bash
helm get values my-nginx
```

This shows only the user-supplied values (overrides).

### Step 6: View the rendered manifests

```bash
helm get manifest my-nginx | head -60
```

This shows the actual Kubernetes YAML that Helm applied.

---

## Exercise 5: Rollback a Release

### Step 1: View release history

```bash
helm history my-nginx
```

You should see multiple revisions with their statuses.

### Step 2: Rollback to revision 1

```bash
helm rollback my-nginx 1
```

### Step 3: Verify the rollback

```bash
helm list
```

The revision number increases (rollback creates a new revision), but the configuration matches revision 1.

```bash
helm get values my-nginx
```

### Step 4: Check the pods

```bash
kubectl get pods -l app.kubernetes.io/instance=my-nginx
```

The replica count should match the original installation (default value).

---

## Exercise 6: Uninstall a Release

### Step 1: Uninstall the release

```bash
helm uninstall my-nginx
```

### Step 2: Verify removal

```bash
helm list
```

The release should be gone.

```bash
kubectl get all -l app.kubernetes.io/instance=my-nginx
```

All resources should be removed.

---

## Exercise 7: Create Your Own Chart (Bonus)

### Step 1: Scaffold a new chart

```bash
helm create my-app
```

### Step 2: Explore the generated structure

```bash
ls -la my-app/
ls -la my-app/templates/
```

Look at the key files:

```bash
cat my-app/Chart.yaml
cat my-app/values.yaml
cat my-app/templates/deployment.yaml
```

### Step 3: Lint the chart

```bash
helm lint ./my-app
```

### Step 4: Dry-run to see rendered output

```bash
helm template my-release ./my-app
```

This renders the templates with default values without installing anything.

### Step 5: Install the chart

```bash
helm install my-release ./my-app
```

### Step 6: Verify

```bash
helm list
kubectl get all -l app.kubernetes.io/instance=my-release
```

### Cleanup

```bash
helm uninstall my-release
rm -rf my-app custom-values.yaml
```

---

## Exercise 8: Useful Helm Commands Reference

Try each of these commands to get familiar:

```bash
# List all repos
helm repo list

# Remove a repo
helm repo remove bitnami

# Search Artifact Hub (online)
helm search hub wordpress

# Install in a specific namespace
helm install my-nginx bitnami/nginx --namespace my-ns --create-namespace

# Dry-run install (preview without applying)
helm install my-nginx bitnami/nginx --dry-run

# Show all values (including defaults)
helm get values my-nginx --all
```

---

## Review Questions

1. What are the three core Helm concepts?
   - **Charts** (packages), **Repositories** (chart storage), **Releases** (deployed instances)

2. What command installs a Helm chart?
   - `helm install <release-name> <chart>`

3. How do you override values at install/upgrade time?
   - `--set key=value` for inline or `-f values.yaml` for a file

4. What happens when you run `helm rollback`?
   - It creates a **new revision** with the configuration of the specified older revision

5. Does Helm v3 require a server-side component like Tiller?
   - **No** — Helm v3 is client-only and talks directly to the Kubernetes API
