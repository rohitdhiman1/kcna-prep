# Lab 16: Role-Based Access Control (RBAC)

## Objective

Create a ServiceAccount with limited permissions using Roles and RoleBindings, then verify access restrictions using `kubectl auth can-i` and the `--as` impersonation flag.

## Prerequisites

- A running Kubernetes cluster (kind, minikube, or cloud).
- `kubectl` installed and configured.
- Read the concept file: [RBAC and Security](../concepts/rbac-security.md)

---

## Step 1: Understand RBAC Components

Before we begin, review the four key RBAC resources:

| Resource | Scope | Purpose |
|----------|-------|---------|
| **Role** | Namespace | Defines permissions within a namespace |
| **ClusterRole** | Cluster-wide | Defines permissions across all namespaces |
| **RoleBinding** | Namespace | Binds a Role to a subject (user, group, or ServiceAccount) |
| **ClusterRoleBinding** | Cluster-wide | Binds a ClusterRole to a subject |

---

## Step 2: Create a Namespace for Testing

```bash
kubectl create namespace rbac-lab
```

### Deploy some test resources in this namespace

```bash
kubectl run web --image=nginx:1.25 --namespace=rbac-lab
kubectl run api --image=nginx:1.25 --namespace=rbac-lab
kubectl create deployment backend --image=nginx:1.25 --namespace=rbac-lab
```

Verify the resources:

```bash
kubectl get all -n rbac-lab
```

---

## Step 3: Create a ServiceAccount

```bash
kubectl create serviceaccount pod-reader -n rbac-lab
```

Verify:

```bash
kubectl get serviceaccount pod-reader -n rbac-lab
```

Expected output:

```
NAME         SECRETS   AGE
pod-reader   0         5s
```

### View ServiceAccount details

```bash
kubectl get serviceaccount pod-reader -n rbac-lab -o yaml
```

---

## Step 4: Create a Role with Limited Permissions

Create a Role that only allows `get` and `list` operations on pods:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader-role
  namespace: rbac-lab
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
EOF
```

### Examine the Role

```bash
kubectl describe role pod-reader-role -n rbac-lab
```

Expected output:

```
Name:         pod-reader-role
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  pods       []                 []              [get list]
```

This Role allows **only** reading pods. It does not allow creating, deleting, updating, or watching pods, nor accessing any other resource type.

---

## Step 5: Create a RoleBinding

Bind the Role to the ServiceAccount:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: rbac-lab
subjects:
- kind: ServiceAccount
  name: pod-reader
  namespace: rbac-lab
roleRef:
  kind: Role
  name: pod-reader-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

### Examine the RoleBinding

```bash
kubectl describe rolebinding pod-reader-binding -n rbac-lab
```

Expected output:

```
Name:         pod-reader-binding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  pod-reader-role
Subjects:
  Kind            Name        Namespace
  ----            ----        ---------
  ServiceAccount  pod-reader  rbac-lab
```

---

## Step 6: Test Permissions with kubectl auth can-i

The `kubectl auth can-i` command checks if a specific action is allowed.

### Check what the ServiceAccount can do

```bash
# Should return "yes" - listing pods is allowed
kubectl auth can-i list pods -n rbac-lab --as=system:serviceaccount:rbac-lab:pod-reader

# Should return "yes" - getting pods is allowed
kubectl auth can-i get pods -n rbac-lab --as=system:serviceaccount:rbac-lab:pod-reader

# Should return "no" - creating pods is NOT allowed
kubectl auth can-i create pods -n rbac-lab --as=system:serviceaccount:rbac-lab:pod-reader

# Should return "no" - deleting pods is NOT allowed
kubectl auth can-i delete pods -n rbac-lab --as=system:serviceaccount:rbac-lab:pod-reader

# Should return "no" - listing deployments is NOT allowed
kubectl auth can-i list deployments -n rbac-lab --as=system:serviceaccount:rbac-lab:pod-reader

# Should return "no" - accessing pods in default namespace is NOT allowed
kubectl auth can-i list pods -n default --as=system:serviceaccount:rbac-lab:pod-reader
```

### List all permissions for the ServiceAccount

```bash
kubectl auth can-i --list -n rbac-lab --as=system:serviceaccount:rbac-lab:pod-reader
```

Expected output (relevant lines):

```
Resources   Non-Resource URLs   Resource Names   Verbs
pods        []                  []               [get list]
...
```

---

## Step 7: Test with --as Impersonation

The `--as` flag lets you impersonate a user or ServiceAccount to test access.

### Operations that should succeed

```bash
# List pods - should work
kubectl get pods -n rbac-lab --as=system:serviceaccount:rbac-lab:pod-reader
```

Expected output:

```
NAME                       READY   STATUS    RESTARTS   AGE
api                        1/1     Running   0          2m
backend-5d4f7b8c9-x2k4m   1/1     Running   0          2m
web                        1/1     Running   0          2m
```

```bash
# Get a specific pod - should work
kubectl get pod web -n rbac-lab --as=system:serviceaccount:rbac-lab:pod-reader
```

### Operations that should fail

```bash
# Try to delete a pod - should be denied
kubectl delete pod web -n rbac-lab --as=system:serviceaccount:rbac-lab:pod-reader
```

Expected error:

```
Error from server (Forbidden): pods "web" is forbidden: User "system:serviceaccount:rbac-lab:pod-reader" cannot delete resource "pods" in API group "" in the namespace "rbac-lab"
```

```bash
# Try to create a pod - should be denied
kubectl run test --image=nginx -n rbac-lab --as=system:serviceaccount:rbac-lab:pod-reader
```

Expected error:

```
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:rbac-lab:pod-reader" cannot create resource "pods" in API group "" in the namespace "rbac-lab"
```

```bash
# Try to list services - should be denied
kubectl get services -n rbac-lab --as=system:serviceaccount:rbac-lab:pod-reader
```

Expected error:

```
Error from server (Forbidden): services is forbidden: User "system:serviceaccount:rbac-lab:pod-reader" cannot list resource "services" in API group "" in the namespace "rbac-lab"
```

---

## Step 8: Expand Permissions with a ClusterRole

Create a ClusterRole that grants read-only access to multiple resource types:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-viewer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
EOF
```

### Bind it to a new ServiceAccount with a RoleBinding (namespace-scoped)

```bash
kubectl create serviceaccount ns-viewer -n rbac-lab
```

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ns-viewer-binding
  namespace: rbac-lab
subjects:
- kind: ServiceAccount
  name: ns-viewer
  namespace: rbac-lab
roleRef:
  kind: ClusterRole
  name: namespace-viewer
  apiGroup: rbac.authorization.k8s.io
EOF
```

**Key insight:** You can bind a ClusterRole using a RoleBinding to limit its scope to a single namespace. The ClusterRole defines the permissions, but the RoleBinding restricts where they apply.

### Verify the expanded permissions

```bash
# Can list pods - yes
kubectl auth can-i list pods -n rbac-lab --as=system:serviceaccount:rbac-lab:ns-viewer

# Can list services - yes (unlike pod-reader)
kubectl auth can-i list services -n rbac-lab --as=system:serviceaccount:rbac-lab:ns-viewer

# Can list deployments - yes
kubectl auth can-i list deployments -n rbac-lab --as=system:serviceaccount:rbac-lab:ns-viewer

# Cannot delete anything - no
kubectl auth can-i delete pods -n rbac-lab --as=system:serviceaccount:rbac-lab:ns-viewer

# Cannot access other namespaces - no
kubectl auth can-i list pods -n default --as=system:serviceaccount:rbac-lab:ns-viewer
```

---

## Step 9: View Built-In ClusterRoles

Kubernetes ships with several default ClusterRoles:

```bash
kubectl get clusterroles | grep -E "^(admin|edit|view|cluster-admin)"
```

Expected output:

```
admin                                                          2024-01-01T00:00:00Z
cluster-admin                                                  2024-01-01T00:00:00Z
edit                                                           2024-01-01T00:00:00Z
view                                                           2024-01-01T00:00:00Z
```

### Inspect the built-in "view" ClusterRole

```bash
kubectl describe clusterrole view
```

This shows all the read-only permissions bundled into the `view` role. These built-in roles follow the principle of least privilege.

---

## Cleanup

```bash
kubectl delete namespace rbac-lab
kubectl delete clusterrole namespace-viewer
```

Deleting the namespace removes all resources within it, including ServiceAccounts, Roles, RoleBindings, and pods.

---

## Key Takeaways

1. **RBAC** controls who can do what in a Kubernetes cluster.
2. **Roles** define permissions (what verbs on what resources); **RoleBindings** assign them to subjects.
3. **ServiceAccounts** are identities for pods and automation; they are namespaced.
4. The `--as` flag lets you impersonate any user or ServiceAccount to test permissions.
5. `kubectl auth can-i` is the quickest way to check specific permissions.
6. A **ClusterRole** bound with a **RoleBinding** limits the ClusterRole's scope to one namespace.
7. Always follow the **principle of least privilege** -- grant only the minimum permissions needed.

---

## Next Steps

- Read about [Network Policies](../concepts/network-policies.md) to learn how to control pod-to-pod traffic.
- Try [Lab 17: Network Policies](17-network-policies.md) to implement network-level access control.
