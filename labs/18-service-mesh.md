# Lab 18: Service Mesh Overview with Linkerd

## Objective

Understand what a service mesh does, optionally install Linkerd on a Kubernetes cluster, inject sidecar proxies into a deployment, and observe the mesh in action via the Linkerd dashboard.

**Note:** This is a heavier lab and service mesh is a conceptual topic on the KCNA exam. You do not need to install Linkerd to pass the exam. Focus on understanding the concepts; the hands-on steps are optional but educational.

## Prerequisites

- A running Kubernetes cluster (kind, minikube, or cloud).
- `kubectl` installed and configured.
- Read the concept file: [Service Mesh](../concepts/service-mesh.md)

---

## Step 1: Understand What a Service Mesh Does

Before installing anything, understand what happens when you add a service mesh to a cluster:

| Without Service Mesh | With Service Mesh |
|---------------------|-------------------|
| Direct pod-to-pod communication | Traffic routed through sidecar proxies |
| No automatic mTLS | Automatic mutual TLS between all services |
| Manual retry/timeout logic in app code | Retries, timeouts, circuit breaking handled by proxy |
| No per-request observability | Detailed metrics, traces, and logs for every request |
| No traffic splitting | Canary deployments and traffic splitting built-in |

A service mesh adds a **data plane** (sidecar proxies in every pod) and a **control plane** (manages proxy configuration).

### How sidecar injection works

When you "inject" a service mesh into a deployment:

1. A sidecar proxy container (e.g., `linkerd-proxy`) is added to each pod.
2. An init container configures iptables to redirect all traffic through the proxy.
3. The proxy intercepts all inbound and outbound traffic transparently.
4. The application code does not need to change.

---

## Step 2: Install the Linkerd CLI

Download and install the Linkerd CLI:

```bash
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
```

Add Linkerd to your PATH:

```bash
export PATH=$HOME/.linkerd2/bin:$PATH
```

Verify the CLI is installed:

```bash
linkerd version --client
```

Expected output:

```
Client version: stable-2.14.x
```

---

## Step 3: Validate the Cluster

Before installing the Linkerd control plane, check that your cluster meets the requirements:

```bash
linkerd check --pre
```

This runs a series of checks including:
- Kubernetes version compatibility
- RBAC permissions
- Clock synchronization
- Pod security standards

All checks should pass with a green checkmark. Fix any issues before proceeding.

---

## Step 4: Install Linkerd CRDs and Control Plane

### Install the Custom Resource Definitions (CRDs)

```bash
linkerd install --crds | kubectl apply -f -
```

### Install the control plane

```bash
linkerd install | kubectl apply -f -
```

### Wait for the control plane to be ready

```bash
linkerd check
```

This runs post-installation checks. Wait until all checks pass. The control plane installs several components in the `linkerd` namespace:

```bash
kubectl get pods -n linkerd
```

Expected output:

```
NAME                                      READY   STATUS    RESTARTS   AGE
linkerd-destination-7b4c5d6e8f-x2k4m     4/4     Running   0          60s
linkerd-identity-6a3b4c5d7e-y3l5n         2/2     Running   0          60s
linkerd-proxy-injector-5c6d7e8f9a-z4m6p   2/2     Running   0          60s
```

Key components:
- **linkerd-identity** — manages TLS certificates for mTLS.
- **linkerd-destination** — service discovery and traffic policy.
- **linkerd-proxy-injector** — automatically injects sidecar proxies.

---

## Step 5: Deploy a Sample Application

Deploy a simple multi-service application:

```bash
kubectl create namespace mesh-demo
```

```bash
kubectl create deployment web --image=nginx:1.25 --port=80 -n mesh-demo
kubectl expose deployment web --port=80 -n mesh-demo

kubectl create deployment api --image=hashicorp/http-echo --port=5678 -n mesh-demo -- -text="API response"
kubectl expose deployment api --port=80 --target-port=5678 -n mesh-demo
```

Verify the deployments:

```bash
kubectl get pods -n mesh-demo
```

Notice each pod has only **1/1** containers (no sidecar yet):

```
NAME                   READY   STATUS    RESTARTS   AGE
api-5d4f7b8c9-x2k4m   1/1     Running   0          30s
web-6e5f8c9d0-y3l5n   1/1     Running   0          35s
```

---

## Step 6: Inject Sidecar Proxies

### Method 1: Inject by annotating the namespace

```bash
kubectl annotate namespace mesh-demo linkerd.io/inject=enabled
```

Then restart the deployments to trigger injection:

```bash
kubectl rollout restart deployment -n mesh-demo
```

### Method 2: Inject using the CLI (alternative)

```bash
kubectl get deployment web -n mesh-demo -o yaml | linkerd inject - | kubectl apply -f -
kubectl get deployment api -n mesh-demo -o yaml | linkerd inject - | kubectl apply -f -
```

### Verify sidecar injection

```bash
kubectl get pods -n mesh-demo
```

Now each pod should show **2/2** containers (application + linkerd-proxy):

```
NAME                   READY   STATUS    RESTARTS   AGE
api-7f8g9h0i1j-a1b2c  2/2     Running   0          30s
web-8g9h0i1j2k-b2c3d  2/2     Running   0          35s
```

### Inspect the injected containers

```bash
kubectl describe pod -l app=web -n mesh-demo
```

Look for two containers:
- `web` — your application container
- `linkerd-proxy` — the sidecar proxy

And an init container:
- `linkerd-init` — configures iptables for traffic interception

---

## Step 7: Install the Linkerd Viz Extension (Dashboard)

The dashboard is provided by the Linkerd Viz extension:

```bash
linkerd viz install | kubectl apply -f -
```

Wait for it to be ready:

```bash
linkerd viz check
```

### Launch the dashboard

```bash
linkerd viz dashboard &
```

This opens a web browser with the Linkerd dashboard. You can see:
- **Live traffic** between services
- **Success rates** and latency percentiles (p50, p95, p99)
- **TCP connections** and bytes transferred
- **mTLS status** for each connection

### View metrics from the CLI

```bash
linkerd viz stat deployment -n mesh-demo
```

Expected output:

```
NAME   MESHED   SUCCESS   RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99   TCP_CONN
api    1/1      100.00%   0.5rps   1ms           2ms           3ms           1
web    1/1      100.00%   0.3rps   1ms           2ms           4ms           1
```

---

## Step 8: Generate Traffic and Observe

Create a load generator to produce traffic:

```bash
kubectl run load-gen --image=busybox:1.36 -n mesh-demo --restart=Never -- \
  /bin/sh -c "while true; do wget -qO- http://web > /dev/null 2>&1; wget -qO- http://api > /dev/null 2>&1; sleep 0.5; done"
```

Now check the dashboard or CLI for live metrics:

```bash
linkerd viz stat deployment -n mesh-demo
```

### View live traffic

```bash
linkerd viz top deployment/web -n mesh-demo
```

This shows live request-by-request traffic, similar to the Unix `top` command.

### Check mTLS status

```bash
linkerd viz edges deployment -n mesh-demo
```

Expected output:

```
SRC          DST    SRC_NS      DST_NS      SECURED
load-gen     api    mesh-demo   mesh-demo   TRUE
load-gen     web    mesh-demo   mesh-demo   TRUE
```

The `SECURED` column confirms that mTLS is active for all connections.

---

## Step 9: Understand What the Mesh Provides (Exam Focus)

For the KCNA exam, focus on these key concepts:

### 1. Mutual TLS (mTLS)
- The mesh automatically encrypts all pod-to-pod traffic.
- Certificates are managed by the control plane (linkerd-identity).
- No application code changes required.

### 2. Observability
- The sidecar proxy emits metrics for every request (latency, success rate, volume).
- No need to instrument application code.
- Golden signals (latency, traffic, errors, saturation) are available out of the box.

### 3. Reliability
- Retries, timeouts, and circuit breaking can be configured at the mesh level.
- The application does not need to implement these patterns.

### 4. Traffic Management
- Traffic splitting for canary deployments.
- Header-based routing for A/B testing.
- Rate limiting and load balancing policies.

### 5. Common Service Mesh Projects
- **Linkerd** — lightweight, simple, CNCF graduated project.
- **Istio** — feature-rich, more complex, uses Envoy proxy.
- **Consul Connect** — HashiCorp's service mesh, integrates with Consul.

---

## Cleanup

```bash
# Delete the demo namespace
kubectl delete namespace mesh-demo

# Remove Linkerd Viz
linkerd viz uninstall | kubectl delete -f -

# Remove Linkerd control plane
linkerd uninstall | kubectl delete -f -
```

---

## Key Takeaways

1. A service mesh adds a **sidecar proxy** to every pod, intercepting all network traffic.
2. The mesh provides **automatic mTLS**, **observability**, and **traffic management** without code changes.
3. The **control plane** manages certificates, configuration, and policy.
4. The **data plane** (sidecar proxies) handles the actual traffic routing and encryption.
5. **Linkerd** is a CNCF graduated project; **Istio** uses Envoy and is more feature-rich.
6. For the KCNA exam, understand the concept and benefits rather than memorizing installation commands.
7. Sidecar injection can be done per-namespace (annotation) or per-deployment (CLI).

---

## Next Steps

- Read about [Storage](../concepts/storage.md) to understand persistent data in Kubernetes.
- Try [Lab 19: Storage](19-storage.md) to work with PersistentVolumes and dynamic provisioning.
