# Lab 15: Ingress with NGINX Ingress Controller

## Objective

Install the NGINX Ingress Controller, deploy sample applications, and configure Ingress resources with host-based and path-based routing rules to expose services externally.

## Prerequisites

- A running Kubernetes cluster (kind, minikube, or cloud).
- `kubectl` installed and configured.
- Read the concept file: [Ingress and DNS](../concepts/ingress-dns.md)

**Note:** If using minikube, enable the ingress addon instead of manual installation:
```bash
minikube addons enable ingress
```
Then skip to Step 2.

---

## Step 1: Install the NGINX Ingress Controller

### Apply the official NGINX Ingress Controller manifest

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
```

### Wait for the ingress controller to be ready

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### Verify the installation

```bash
kubectl get pods -n ingress-nginx
```

Expected output:

```
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-7d4db76476-k2rhz   1/1     Running   0          60s
```

### Check the ingress controller service

```bash
kubectl get svc -n ingress-nginx
```

Expected output:

```
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.96.45.123    <pending>      80:31080/TCP,443:31443/TCP   90s
ingress-nginx-controller-admission   ClusterIP      10.96.78.234    <none>         443/TCP                      90s
```

**Note:** On local clusters (kind, minikube), `EXTERNAL-IP` will show `<pending>` since there is no cloud load balancer. You will use NodePort or port-forwarding to access the ingress.

### For kind clusters: set up port forwarding

```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80 &
```

This forwards `localhost:8080` to the ingress controller. Keep this running.

---

## Step 2: Create Sample Deployments and Services

### Deploy app-one (returns a custom message)

```bash
kubectl create deployment app-one --image=hashicorp/http-echo --port=5678 -- -text="Hello from App One"
kubectl expose deployment app-one --port=80 --target-port=5678
```

### Deploy app-two (returns a different message)

```bash
kubectl create deployment app-two --image=hashicorp/http-echo --port=5678 -- -text="Hello from App Two"
kubectl expose deployment app-two --port=80 --target-port=5678
```

### Verify the deployments and services

```bash
kubectl get deployments
kubectl get services
```

Expected output:

```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
app-one   1/1     1            1           30s
app-two   1/1     1            1           25s

NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
app-one      ClusterIP   10.96.101.10    <none>        80/TCP    30s
app-two      ClusterIP   10.96.102.20    <none>        80/TCP    25s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   10d
```

### Test the services internally

```bash
kubectl run test-curl --image=curlimages/curl --rm -it --restart=Never -- curl -s http://app-one
kubectl run test-curl --image=curlimages/curl --rm -it --restart=Never -- curl -s http://app-two
```

You should see "Hello from App One" and "Hello from App Two" respectively.

---

## Step 3: Create an Ingress with Path-Based Routing

Route traffic to different services based on the URL path.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /app-one
        pathType: Prefix
        backend:
          service:
            name: app-one
            port:
              number: 80
      - path: /app-two
        pathType: Prefix
        backend:
          service:
            name: app-two
            port:
              number: 80
EOF
```

### Verify the Ingress resource

```bash
kubectl get ingress
```

Expected output:

```
NAME                 CLASS   HOSTS   ADDRESS   PORTS   AGE
path-based-ingress   nginx   *                 80      10s
```

### Describe the Ingress for details

```bash
kubectl describe ingress path-based-ingress
```

Look at the `Rules` section to confirm the path-to-service mapping.

---

## Step 4: Test Path-Based Routing

### Test with curl (using port-forward on kind, or the external IP on cloud)

```bash
# For kind with port-forward running on 8080:
curl http://localhost:8080/app-one
curl http://localhost:8080/app-two

# For minikube:
# curl http://$(minikube ip)/app-one
# curl http://$(minikube ip)/app-two
```

Expected output:

```
Hello from App One
```

```
Hello from App Two
```

Each path routes to the correct backend service.

---

## Step 5: Create an Ingress with Host-Based Routing

Route traffic based on the `Host` header (virtual hosting).

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: one.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-one
            port:
              number: 80
  - host: two.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-two
            port:
              number: 80
EOF
```

### Verify

```bash
kubectl get ingress host-based-ingress
```

Expected output:

```
NAME                 CLASS   HOSTS                            ADDRESS   PORTS   AGE
host-based-ingress   nginx   one.example.com,two.example.com             80      10s
```

---

## Step 6: Test Host-Based Routing

Use the `-H` flag in curl to set the `Host` header:

```bash
# For kind with port-forward:
curl -H "Host: one.example.com" http://localhost:8080/
curl -H "Host: two.example.com" http://localhost:8080/

# For minikube:
# curl -H "Host: one.example.com" http://$(minikube ip)/
# curl -H "Host: two.example.com" http://$(minikube ip)/
```

Expected output:

```
Hello from App One
```

```
Hello from App Two
```

The ingress controller inspects the `Host` header and routes to the matching backend.

---

## Step 7: Combine Host-Based and Path-Based Routing

You can combine both routing strategies in a single Ingress:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: combined-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /one
        pathType: Prefix
        backend:
          service:
            name: app-one
            port:
              number: 80
      - path: /two
        pathType: Prefix
        backend:
          service:
            name: app-two
            port:
              number: 80
EOF
```

### Test combined routing

```bash
curl -H "Host: myapp.example.com" http://localhost:8080/one
curl -H "Host: myapp.example.com" http://localhost:8080/two
```

---

## Step 8: Examine Ingress Controller Logs

View the NGINX Ingress Controller logs to see how requests are being routed:

```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=20
```

You should see access logs showing the incoming requests, Host headers, and backend selections.

---

## Cleanup

```bash
# Stop the port-forward if running
kill %1 2>/dev/null

# Delete all resources
kubectl delete ingress path-based-ingress host-based-ingress combined-ingress
kubectl delete deployment app-one app-two
kubectl delete service app-one app-two
```

To uninstall the NGINX Ingress Controller entirely:

```bash
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
```

---

## Key Takeaways

1. An **Ingress Controller** (like NGINX) must be installed before Ingress resources work.
2. **Ingress** resources define routing rules; the controller implements them.
3. **Path-based routing** sends requests to different services based on the URL path.
4. **Host-based routing** uses the HTTP `Host` header for virtual hosting.
5. The `ingressClassName` field links the Ingress to a specific controller.
6. The `rewrite-target` annotation is needed when backend apps do not expect the prefix path.
7. On local clusters, use port-forwarding or NodePort to access the ingress controller.

---

## Next Steps

- Read about [RBAC and Security](../concepts/rbac-security.md) to learn how access control works.
- Try [Lab 16: RBAC](16-rbac.md) to configure role-based access control.
