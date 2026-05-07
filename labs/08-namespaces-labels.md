# Lab 08: Namespaces and Labels

## Objective

Create and work with namespaces, deploy resources into specific namespaces, add and remove labels from resources, and use label selectors to filter resources.

## Prerequisites

- A running Kubernetes cluster (from [Lab 01](01-cluster-setup.md)).
- `kubectl` installed and configured.
- Read the concept file: [Namespaces and Labels](../concepts/namespaces-labels.md)

---

## Step 1: Explore Existing Namespaces

```bash
kubectl get namespaces
```

Expected output:

```
NAME                 STATUS   AGE
default              Active   30m
kube-node-lease      Active   30m
kube-public          Active   30m
kube-system          Active   30m
local-path-storage   Active   30m
```

- **default** — where resources go if you do not specify a namespace.
- **kube-system** — Kubernetes system components.
- **kube-public** — publicly readable by all users (rarely used).
- **kube-node-lease** — holds Lease objects for node heartbeats.

---

## Step 2: Create Namespaces

### Create using imperative command

```bash
kubectl create namespace dev
kubectl create namespace staging
```

### Create using YAML

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    environment: production
EOF
```

### Verify

```bash
kubectl get namespaces
```

You should now see `dev`, `staging`, and `prod` in the list.

---

## Step 3: Deploy Resources in Specific Namespaces

### Deploy to the dev namespace

```bash
kubectl create deployment web-dev --image=nginx:1.27 --replicas=2 -n dev
```

### Deploy to the staging namespace

```bash
kubectl create deployment web-staging --image=nginx:1.27 --replicas=1 -n staging
```

### Deploy to the prod namespace

```bash
kubectl create deployment web-prod --image=nginx:1.27 --replicas=3 -n prod
```

### Verify resources in each namespace

```bash
kubectl get pods -n dev
kubectl get pods -n staging
kubectl get pods -n prod
```

### List pods across all namespaces

```bash
kubectl get pods -A
```

The `-A` (or `--all-namespaces`) flag shows pods from every namespace.

---

## Step 4: Set a Default Namespace

Instead of typing `-n dev` every time, you can set a default namespace for your context:

```bash
kubectl config set-context --current --namespace=dev
```

Now commands will target the `dev` namespace by default:

```bash
kubectl get pods
```

This shows pods in `dev` without needing `-n dev`.

### Reset back to default

```bash
kubectl config set-context --current --namespace=default
```

---

## Step 5: Add Labels to Resources

### Add labels to pods

Create a few pods in the default namespace:

```bash
kubectl run app1 --image=nginx:1.27 --labels="app=nginx,tier=frontend,env=dev"
kubectl run app2 --image=nginx:1.27 --labels="app=nginx,tier=backend,env=dev"
kubectl run app3 --image=nginx:1.27 --labels="app=httpd,tier=frontend,env=prod"
kubectl run app4 --image=httpd:2.4 --labels="app=httpd,tier=backend,env=prod"
```

### View labels

```bash
kubectl get pods --show-labels
```

Expected output:

```
NAME   READY   STATUS    RESTARTS   AGE   LABELS
app1   1/1     Running   0          10s   app=nginx,env=dev,tier=frontend
app2   1/1     Running   0          10s   app=nginx,env=dev,tier=backend
app3   1/1     Running   0          10s   app=httpd,env=prod,tier=frontend
app4   1/1     Running   0          10s   app=httpd,env=prod,tier=backend
```

---

## Step 6: Add and Remove Labels from Existing Resources

### Add a label

```bash
kubectl label pod app1 version=v1
```

### Verify

```bash
kubectl get pod app1 --show-labels
```

The `version=v1` label is now present.

### Overwrite an existing label

```bash
kubectl label pod app1 env=staging --overwrite
```

### Remove a label

Use a minus sign (`-`) after the label key:

```bash
kubectl label pod app1 version-
```

### Verify

```bash
kubectl get pod app1 --show-labels
```

The `version` label is gone, and `env` now shows `staging`.

---

## Step 7: Filter Resources with Label Selectors

### Equality-based selectors

```bash
# Get all pods with app=nginx
kubectl get pods -l app=nginx

# Get all pods NOT in prod environment
kubectl get pods -l env!=prod
```

### Set-based selectors

```bash
# Get pods where env is dev OR prod
kubectl get pods -l 'env in (dev,prod)'

# Get pods where tier is NOT backend
kubectl get pods -l 'tier notin (backend)'
```

### Multiple selectors (AND logic)

```bash
# Get frontend nginx pods
kubectl get pods -l app=nginx,tier=frontend
```

Expected output:

```
NAME   READY   STATUS    RESTARTS   AGE
app1   1/1     Running   0          2m
```

### Use selectors with other commands

```bash
# Describe all frontend pods
kubectl describe pods -l tier=frontend

# Delete all dev environment pods
kubectl get pods -l env=dev
```

---

## Step 8: Label Nodes

Labels on nodes are used for scheduling decisions (see [Lab 11](11-scheduling.md)).

### Add a label to a node

```bash
kubectl label node kcna-lab-worker disktype=ssd
```

### Verify

```bash
kubectl get nodes --show-labels | grep disktype
```

### Remove the label

```bash
kubectl label node kcna-lab-worker disktype-
```

---

## Step 9: Using Labels with Services

Services use label selectors to find their backend pods:

```bash
kubectl expose pod app1 --name=frontend-svc --port=80 --target-port=80
```

```bash
kubectl describe service frontend-svc
```

Look at the **Selector** field. It matches the labels on `app1`.

---

## Step 10: Cross-Namespace Service Access

Services in different namespaces are accessible using their fully qualified DNS name:

```bash
kubectl expose deployment web-dev --name=web-dev-svc --port=80 -n dev
```

From a pod in the default namespace, access it using the full DNS name:

```bash
kubectl run dns-cross --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://web-dev-svc.dev.svc.cluster.local
```

The format is `<service-name>.<namespace>.svc.cluster.local`.

---

## Cleanup

```bash
kubectl delete pod app1 app2 app3 app4
kubectl delete service frontend-svc
kubectl delete service web-dev-svc -n dev
kubectl delete deployment web-dev -n dev
kubectl delete deployment web-staging -n staging
kubectl delete deployment web-prod -n prod
kubectl delete namespace dev staging prod
```

---

## Key Takeaways

1. **Namespaces** provide logical isolation within a cluster. They scope resource names and can be used with RBAC for access control.
2. Use `-n <namespace>` to target a specific namespace, or `-A` for all namespaces.
3. **Labels** are key-value pairs attached to resources. They are the primary mechanism for grouping and selecting resources.
4. Label selectors support equality-based (`=`, `!=`) and set-based (`in`, `notin`) operators.
5. Multiple selectors are combined with AND logic (comma-separated).
6. Services use label selectors to discover their backend pods.
7. Resources in different namespaces communicate via fully qualified DNS names: `<service>.<namespace>.svc.cluster.local`.
8. `kubectl label` can add, update (with `--overwrite`), and remove (with `-` suffix) labels on any resource.

---

## Next Steps

- Proceed to [Lab 09: ConfigMaps and Secrets](09-configmaps-secrets.md) to manage configuration data.
- Read about [RBAC and Security](../concepts/rbac-security.md) for namespace-level access control.
