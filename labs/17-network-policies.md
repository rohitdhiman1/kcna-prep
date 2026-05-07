# Lab 17: Network Policies

## Objective

Deploy a multi-tier application (frontend, backend, database), verify default open communication, apply a default deny policy, then create targeted Network Policies to allow only specific pod-to-pod traffic flows.

## Prerequisites

- A running Kubernetes cluster (kind, minikube, or cloud).
- `kubectl` installed and configured.
- A CNI plugin that supports Network Policies (Calico, Cilium, or Weave). **Note:** The default CNI in kind (kindnet) and minikube (bridge) do NOT enforce Network Policies. See the setup note below.
- Read the concept file: [Network Policies](../concepts/network-policies.md)

**Important:** To test Network Policies on kind, install Calico:

```bash
# Create a kind cluster without the default CNI
kind create cluster --name netpol-lab --config - <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: "192.168.0.0/16"
EOF

# Install Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Wait for Calico to be ready
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n kube-system --timeout=120s
```

For minikube, use the Calico CNI:
```bash
minikube start --cni=calico
```

---

## Step 1: Create a Test Namespace

```bash
kubectl create namespace netpol-lab
```

---

## Step 2: Deploy the Three-Tier Application

### Deploy the frontend

```bash
kubectl run frontend --image=nginx:1.25 --labels="app=frontend,tier=frontend" -n netpol-lab --port=80
```

### Deploy the backend

```bash
kubectl run backend --image=nginx:1.25 --labels="app=backend,tier=backend" -n netpol-lab --port=80
```

### Deploy the database

```bash
kubectl run database --image=nginx:1.25 --labels="app=database,tier=database" -n netpol-lab --port=80
```

### Verify all pods are running

```bash
kubectl get pods -n netpol-lab -o wide --show-labels
```

Expected output:

```
NAME       READY   STATUS    RESTARTS   AGE   IP             NODE     LABELS
backend    1/1     Running   0          30s   192.168.0.6    ...      app=backend,tier=backend
database   1/1     Running   0          25s   192.168.0.7    ...      app=database,tier=database
frontend   1/1     Running   0          35s   192.168.0.5    ...      app=frontend,tier=frontend
```

---

## Step 3: Verify Default Open Communication

By default, all pods can communicate with all other pods. Verify this:

### Frontend can reach backend

```bash
BACKEND_IP=$(kubectl get pod backend -n netpol-lab -o jsonpath='{.status.podIP}')
kubectl exec frontend -n netpol-lab -- curl -s --max-time 3 http://$BACKEND_IP
```

You should see the NGINX welcome page.

### Frontend can reach database

```bash
DATABASE_IP=$(kubectl get pod database -n netpol-lab -o jsonpath='{.status.podIP}')
kubectl exec frontend -n netpol-lab -- curl -s --max-time 3 http://$DATABASE_IP
```

You should see the NGINX welcome page.

### Backend can reach database

```bash
kubectl exec backend -n netpol-lab -- curl -s --max-time 3 http://$DATABASE_IP
```

You should see the NGINX welcome page.

### Database can reach frontend

```bash
FRONTEND_IP=$(kubectl get pod frontend -n netpol-lab -o jsonpath='{.status.podIP}')
kubectl exec database -n netpol-lab -- curl -s --max-time 3 http://$FRONTEND_IP
```

You should see the NGINX welcome page. All pods can communicate freely.

---

## Step 4: Apply a Default Deny Ingress Policy

This policy blocks ALL incoming traffic to pods in the namespace:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: netpol-lab
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
```

- `podSelector: {}` matches ALL pods in the namespace.
- `policyTypes: [Ingress]` with no `ingress` rules means all incoming traffic is denied.

### Verify the policy

```bash
kubectl get networkpolicy -n netpol-lab
```

Expected output:

```
NAME                   POD-SELECTOR   AGE
default-deny-ingress   <none>         10s
```

---

## Step 5: Verify Communication Is Blocked

### Frontend to backend (should timeout)

```bash
kubectl exec frontend -n netpol-lab -- curl -s --max-time 3 http://$BACKEND_IP
```

Expected: the command hangs for 3 seconds and returns an error (connection timed out or empty reply).

### Frontend to database (should timeout)

```bash
kubectl exec frontend -n netpol-lab -- curl -s --max-time 3 http://$DATABASE_IP
```

Expected: timeout/failure.

### Backend to database (should timeout)

```bash
kubectl exec backend -n netpol-lab -- curl -s --max-time 3 http://$DATABASE_IP
```

Expected: timeout/failure.

All ingress traffic is now blocked. No pod can receive incoming connections.

---

## Step 6: Allow Frontend to Backend Communication

Create a policy that allows the frontend to send traffic to the backend:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
EOF
```

This policy:
- Applies to pods with label `app: backend` (the target).
- Allows ingress from pods with label `app: frontend` (the source).
- Only allows TCP traffic on port 80.

### Test: frontend to backend (should work now)

```bash
kubectl exec frontend -n netpol-lab -- curl -s --max-time 3 http://$BACKEND_IP
```

Expected: NGINX welcome page (success).

### Test: database to backend (should still be blocked)

```bash
kubectl exec database -n netpol-lab -- curl -s --max-time 3 http://$BACKEND_IP
```

Expected: timeout/failure. Only the frontend is allowed.

---

## Step 7: Allow Backend to Database Communication

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 80
EOF
```

### Test: backend to database (should work now)

```bash
kubectl exec backend -n netpol-lab -- curl -s --max-time 3 http://$DATABASE_IP
```

Expected: NGINX welcome page (success).

### Test: frontend to database (should still be blocked)

```bash
kubectl exec frontend -n netpol-lab -- curl -s --max-time 3 http://$DATABASE_IP
```

Expected: timeout/failure. Only the backend is allowed to reach the database.

---

## Step 8: Verify the Complete Traffic Flow

Summarize what is now allowed and blocked:

| Source | Destination | Allowed? |
|--------|------------|----------|
| frontend | backend | Yes |
| frontend | database | No |
| backend | database | Yes |
| backend | frontend | No |
| database | frontend | No |
| database | backend | No |

This enforces a proper three-tier architecture where:
- Frontend talks to backend only.
- Backend talks to database only.
- Database accepts connections only from backend.

### Run a comprehensive test

```bash
echo "=== frontend -> backend ==="
kubectl exec frontend -n netpol-lab -- curl -s --max-time 3 -o /dev/null -w "%{http_code}" http://$BACKEND_IP 2>/dev/null || echo "BLOCKED"

echo "=== frontend -> database ==="
kubectl exec frontend -n netpol-lab -- curl -s --max-time 3 -o /dev/null -w "%{http_code}" http://$DATABASE_IP 2>/dev/null || echo "BLOCKED"

echo "=== backend -> database ==="
kubectl exec backend -n netpol-lab -- curl -s --max-time 3 -o /dev/null -w "%{http_code}" http://$DATABASE_IP 2>/dev/null || echo "BLOCKED"

echo "=== database -> frontend ==="
kubectl exec database -n netpol-lab -- curl -s --max-time 3 -o /dev/null -w "%{http_code}" http://$FRONTEND_IP 2>/dev/null || echo "BLOCKED"
```

---

## Step 9: Examine Network Policies

### List all policies

```bash
kubectl get networkpolicy -n netpol-lab
```

Expected output:

```
NAME                        POD-SELECTOR     AGE
allow-backend-to-database   app=database     2m
allow-frontend-to-backend   app=backend      5m
default-deny-ingress        <none>           8m
```

### Describe a specific policy

```bash
kubectl describe networkpolicy allow-frontend-to-backend -n netpol-lab
```

### View a policy in YAML

```bash
kubectl get networkpolicy allow-frontend-to-backend -n netpol-lab -o yaml
```

---

## Cleanup

```bash
kubectl delete namespace netpol-lab
```

If you created a separate kind cluster for this lab:

```bash
kind delete cluster --name netpol-lab
```

---

## Key Takeaways

1. By default, Kubernetes allows **all pod-to-pod communication** (no isolation).
2. A **default deny** policy blocks all traffic, then you selectively allow specific flows.
3. Network Policies use **label selectors** to identify source and target pods.
4. Policies are **additive** -- multiple policies combine to form the effective rules.
5. Network Policies require a **CNI plugin that supports them** (Calico, Cilium, Weave).
6. The `podSelector` in the policy spec defines which pods the policy **applies to** (the target).
7. The `ingress.from` section defines which pods are **allowed to send traffic** (the source).
8. Network Policies are a key security mechanism for implementing **zero-trust networking**.

---

## Next Steps

- Read about [Service Mesh](../concepts/service-mesh.md) to learn about advanced traffic management.
- Try [Lab 18: Service Mesh](18-service-mesh.md) for an overview of service mesh concepts.
