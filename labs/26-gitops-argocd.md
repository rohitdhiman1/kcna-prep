# Lab 26: GitOps with ArgoCD

## Objective

Install ArgoCD on a local Kubernetes cluster, access the UI, connect a Git repository, create an Application, and observe ArgoCD automatically sync changes from Git.

## Prerequisites

- A running Kubernetes cluster (kind or minikube recommended)
- `kubectl` installed and configured
- `git` installed
- A GitHub account (for creating a test repo)
- Internet access

---

## Exercise 1: Install ArgoCD

### Step 1: Create a kind cluster (skip if you already have a cluster)

```bash
kind create cluster --name argocd-lab
```

Verify:

```bash
kubectl cluster-info
```

### Step 2: Create the ArgoCD namespace

```bash
kubectl create namespace argocd
```

### Step 3: Install ArgoCD

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Step 4: Wait for ArgoCD pods to be ready

```bash
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=120s
```

Verify all pods are running:

```bash
kubectl get pods -n argocd
```

You should see pods for:
- `argocd-server` (API server + UI)
- `argocd-repo-server` (clones Git repos)
- `argocd-application-controller` (watches and reconciles)
- `argocd-redis` (caching)
- `argocd-dex-server` (SSO, optional)

### Step 5: Install the ArgoCD CLI (optional but helpful)

**macOS:**

```bash
brew install argocd
```

**Linux:**

```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

---

## Exercise 2: Access the ArgoCD UI

### Step 1: Port-forward the ArgoCD server

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Leave this running in a separate terminal.

### Step 2: Get the initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo
```

### Step 3: Log in via the browser

Open your browser and go to:

```
https://localhost:8080
```

Accept the self-signed certificate warning, then log in:
- **Username:** `admin`
- **Password:** (the password from Step 2)

### Step 4: Log in via the CLI (optional)

```bash
argocd login localhost:8080 --insecure --username admin --password <password-from-step-2>
```

---

## Exercise 3: Create a Git Repository with Kubernetes Manifests

### Step 1: Create a new GitHub repo

Go to https://github.com/new and create a repository called `argocd-lab-config`. Make it **public** for simplicity.

### Step 2: Clone the repo locally

```bash
git clone https://github.com/<your-username>/argocd-lab-config.git
cd argocd-lab-config
```

### Step 3: Create Kubernetes manifests

Create a simple nginx deployment and service:

```bash
mkdir -p k8s

cat <<EOF > k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  labels:
    app: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.24
          ports:
            - containerPort: 80
EOF

cat <<EOF > k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
spec:
  selector:
    app: nginx-demo
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
EOF
```

### Step 4: Push to GitHub

```bash
git add .
git commit -m "Add initial nginx deployment and service"
git push origin main
```

---

## Exercise 4: Create an ArgoCD Application

### Option A: Using the CLI

```bash
argocd app create nginx-demo \
  --repo https://github.com/<your-username>/argocd-lab-config.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --self-heal \
  --auto-prune
```

### Option B: Using a YAML manifest (recommended for GitOps)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/argocd-lab-config.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

**Replace `<your-username>` with your actual GitHub username.**

### Step 2: Verify the Application was created

```bash
argocd app list
# or
kubectl get applications -n argocd
```

### Step 3: Check sync status

```bash
argocd app get nginx-demo
```

Or check in the ArgoCD UI at https://localhost:8080 — you should see the `nginx-demo` application with status **Synced** and **Healthy**.

### Step 4: Verify the resources in the cluster

```bash
kubectl get deployment nginx-demo
kubectl get svc nginx-demo
kubectl get pods -l app=nginx-demo
```

ArgoCD has deployed the manifests from Git into your cluster.

---

## Exercise 5: Make a Change in Git and Watch ArgoCD Sync

### Step 1: Update the deployment in Git

Edit the replica count:

```bash
cd argocd-lab-config

# Change replicas from 2 to 4
sed -i '' 's/replicas: 2/replicas: 4/' k8s/deployment.yaml
# On Linux (without macOS sed):
# sed -i 's/replicas: 2/replicas: 4/' k8s/deployment.yaml

git add .
git commit -m "Scale nginx to 4 replicas"
git push origin main
```

### Step 2: Watch ArgoCD detect and sync the change

ArgoCD polls Git every 3 minutes by default. You can force a sync:

```bash
argocd app sync nginx-demo
```

Or wait and watch:

```bash
argocd app get nginx-demo --refresh
```

### Step 3: Verify in the cluster

```bash
kubectl get pods -l app=nginx-demo
```

You should see 4 pods now.

### Step 4: Check the sync history in the UI

Open the ArgoCD UI and click on `nginx-demo`. You can see:
- The application tree (visual view of all resources)
- Sync history
- Health status of each resource

---

## Exercise 6: Observe Self-Healing (Drift Correction)

### Step 1: Make a manual change to the cluster

Manually scale the deployment — this creates **drift** from the Git state:

```bash
kubectl scale deployment nginx-demo --replicas=1
```

### Step 2: Watch ArgoCD correct the drift

Since we enabled `selfHeal: true`, ArgoCD will detect the drift and revert it back to 4 replicas (the value in Git).

```bash
# Watch the pods — ArgoCD will scale back to 4
kubectl get pods -l app=nginx-demo -w
```

Within a few minutes (or instantly if you trigger a sync), the replica count will be restored to 4.

### Step 3: Verify

```bash
kubectl get deployment nginx-demo -o jsonpath='{.spec.replicas}'
echo
```

Output should be `4`.

**This is the power of GitOps: the cluster always matches Git.**

---

## Exercise 7: Rollback via Git Revert

### Step 1: Revert the last commit

```bash
cd argocd-lab-config
git revert HEAD --no-edit
git push origin main
```

### Step 2: Sync and verify

```bash
argocd app sync nginx-demo
kubectl get pods -l app=nginx-demo
```

You should see 2 pods again (the original replica count). **Rollback in GitOps = git revert.**

---

## Cleanup

### Remove the ArgoCD Application

```bash
argocd app delete nginx-demo --yes
# or
kubectl delete application nginx-demo -n argocd
```

### Remove ArgoCD

```bash
kubectl delete namespace argocd
```

### Delete the kind cluster (optional)

```bash
kind delete cluster --name argocd-lab
```

### Remove the test repo

Delete the `argocd-lab-config` repository from GitHub if you no longer need it.

---

## Review Questions

1. What are the three core components of ArgoCD?
   - **API Server**, **Repo Server**, **Application Controller**

2. How does ArgoCD know what to deploy?
   - Via the **Application CRD**, which specifies a Git repo, path, and destination cluster/namespace

3. What does `selfHeal: true` do?
   - It reverts any **manual changes** made directly to the cluster, ensuring the cluster always matches Git

4. What does `prune: true` do?
   - It **deletes** resources from the cluster that have been removed from the Git repository

5. How do you rollback in a GitOps workflow?
   - **Revert the commit in Git** (`git revert`). The GitOps agent will sync the cluster to match the reverted state.
