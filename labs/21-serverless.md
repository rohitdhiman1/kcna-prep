# Lab 21: Serverless with Knative

## Objective

Install Knative Serving on a local cluster, deploy a Knative Service, observe scale-to-zero behavior, and watch it scale back up when traffic arrives.

## Prerequisites

- A running Kubernetes cluster (kind or minikube recommended).
- `kubectl` installed and configured.
- At least **4 GB RAM** allocated to your cluster (Knative is resource-intensive).
- Read the concept file: [Serverless](../concepts/serverless.md)

---

## Option A: Full Knative Installation (Recommended)

This option installs actual Knative Serving and demonstrates real scale-to-zero behavior.

### Step 1: Create a Cluster with Sufficient Resources

#### Using kind

```bash
cat <<'EOF' | kind create cluster --name knative-lab --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31080
    hostPort: 8080
    protocol: TCP
EOF
```

#### Using minikube

```bash
minikube start --cpus=4 --memory=4096 --profile=knative-lab
```

---

### Step 2: Install Knative Serving CRDs

Install the Knative Serving Custom Resource Definitions:

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.17.0/serving-crds.yaml
```

Wait for CRDs to be established:

```bash
kubectl wait --for=condition=Established --all crd --timeout=60s
```

---

### Step 3: Install Knative Serving Core

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.17.0/serving-core.yaml
```

Wait for Knative Serving components to be ready:

```bash
kubectl get pods -n knative-serving --watch
```

Wait until all pods show `Running` or `Completed` status. Press `Ctrl+C` when ready.

Expected pods:
- `activator-*`
- `autoscaler-*`
- `controller-*`
- `webhook-*`

---

### Step 4: Install a Networking Layer (Kourier)

Knative needs a networking layer. Kourier is the simplest option:

```bash
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.17.0/kourier.yaml
```

Configure Knative to use Kourier:

```bash
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'
```

Wait for Kourier to be ready:

```bash
kubectl get pods -n kourier-system --watch
```

---

### Step 5: Configure DNS (sslip.io for local development)

For local development, configure Knative to use sslip.io for automatic DNS:

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.17.0/serving-default-domain.yaml
```

Alternatively, configure a simple default domain:

```bash
kubectl patch configmap/config-domain \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"example.com":""}}'
```

---

### Step 6: Verify Knative Installation

```bash
kubectl get pods -n knative-serving
kubectl get pods -n kourier-system
```

All pods should be running. Check Knative is ready:

```bash
kubectl get ksvc
```

This should return "No resources found" (not an error).

---

### Step 7: Deploy a Knative Service

Deploy a simple "Hello World" Knative Service:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/target: "10"
        autoscaling.knative.dev/scale-to-zero-pod-retention-period: "30s"
    spec:
      containers:
      - image: ghcr.io/knative/helloworld-go:latest
        ports:
        - containerPort: 8080
        env:
        - name: TARGET
          value: "from Knative"
EOF
```

This creates a Knative Service with:
- A Go-based Hello World application.
- Autoscaling target of 10 concurrent requests per pod.
- Scale-to-zero after 30 seconds of no traffic (shorter than default for demo purposes).

### Check the Knative Service

```bash
kubectl get ksvc hello
```

Expected output:
```
NAME    URL                                LATESTCREATED   LATESTREADY     READY   REASON
hello   http://hello.default.example.com   hello-00001     hello-00001     True
```

### Check the pods

```bash
kubectl get pods -l serving.knative.dev/service=hello
```

You should see one pod running.

---

### Step 8: Send a Request

Get the URL of the Kourier service:

```bash
kubectl get svc kourier -n kourier-system
```

#### For kind clusters

Port-forward the Kourier service:

```bash
kubectl port-forward -n kourier-system svc/kourier 8080:80 &
```

Send a request with the Host header:

```bash
curl -H "Host: hello.default.example.com" http://localhost:8080
```

#### For minikube clusters

```bash
MINIKUBE_IP=$(minikube ip --profile=knative-lab)
KOURIER_PORT=$(kubectl get svc kourier -n kourier-system -o jsonpath='{.spec.ports[0].nodePort}')
curl -H "Host: hello.default.example.com" http://${MINIKUBE_IP}:${KOURIER_PORT}
```

Expected response:
```
Hello from Knative!
```

---

### Step 9: Observe Scale-to-Zero

Watch the pods for the hello service:

```bash
kubectl get pods -l serving.knative.dev/service=hello --watch
```

**Do not send any more requests.** Wait 30-60 seconds (we set the retention period to 30s).

You should see the pod terminate:

```
NAME                                     READY   STATUS        RESTARTS   AGE
hello-00001-deployment-abc123-xyz        2/2     Running       0          2m
hello-00001-deployment-abc123-xyz        2/2     Terminating   0          3m
hello-00001-deployment-abc123-xyz        0/2     Terminating   0          3m
```

After termination, check pods again:

```bash
kubectl get pods -l serving.knative.dev/service=hello
```

**No pods running!** The service has scaled to zero. This is the key serverless behavior.

---

### Step 10: Observe Scale-from-Zero

Now send another request:

```bash
curl -H "Host: hello.default.example.com" http://localhost:8080
```

In another terminal, watch the pods:

```bash
kubectl get pods -l serving.knative.dev/service=hello --watch
```

You will see a new pod start:
```
NAME                                     READY   STATUS              RESTARTS   AGE
hello-00001-deployment-abc123-xyz        0/2     ContainerCreating   0          1s
hello-00001-deployment-abc123-xyz        1/2     Running             0          3s
hello-00001-deployment-abc123-xyz        2/2     Running             0          5s
```

The request was **buffered by the Knative Activator** while the pod started. The caller experienced a brief delay (cold start), but the request was served.

---

### Step 11: Explore Knative Resources

Knative creates several resources behind the scenes:

```bash
# The Knative Service (top-level resource)
kubectl get ksvc hello -o yaml

# The Configuration (manages Revisions)
kubectl get configuration hello

# The Revision (immutable snapshot of a deployment)
kubectl get revision

# The Route (maps URLs to Revisions)
kubectl get route hello

# The underlying Kubernetes Deployment
kubectl get deploy -l serving.knative.dev/service=hello
```

Resource hierarchy:

```
  Knative Service (ksvc)
       │
       ├──► Configuration ──► Revision (hello-00001)
       │                         └──► Deployment ──► ReplicaSet ──► Pods
       │
       └──► Route
               └──► maps URL to Revision(s)
```

---

### Step 12: Deploy a New Revision (Traffic Splitting)

Update the service with a new environment variable:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
      - image: ghcr.io/knative/helloworld-go:latest
        ports:
        - containerPort: 8080
        env:
        - name: TARGET
          value: "from Knative v2"
  traffic:
  - latestRevision: true
    percent: 50
  - revisionName: hello-00001
    percent: 50
EOF
```

This splits traffic 50/50 between the old and new revisions.

```bash
kubectl get revisions
kubectl get route hello -o yaml
```

Send multiple requests and observe the responses alternate:

```bash
for i in $(seq 1 10); do
  curl -s -H "Host: hello.default.example.com" http://localhost:8080
done
```

You should see responses from both "from Knative" and "from Knative v2".

---

## Cleanup

```bash
kubectl delete ksvc hello
```

If you created a dedicated cluster:

```bash
# For kind
kind delete cluster --name knative-lab

# For minikube
minikube delete --profile=knative-lab
```

---

## Option B: Simpler Alternative (No Knative)

If Knative is too resource-intensive for your machine, this alternative demonstrates the **concept** of scale-to-zero using a standard Kubernetes Deployment scaled manually.

### Step 1: Create a Deployment

```bash
kubectl create deployment hello-serverless \
  --image=hashicorp/http-echo \
  -- -text="Hello from simulated serverless"

kubectl expose deployment hello-serverless --port=5678 --type=ClusterIP
```

### Step 2: Verify It Works

```bash
kubectl port-forward svc/hello-serverless 5678:5678 &
curl http://localhost:5678
```

### Step 3: Simulate Scale-to-Zero

```bash
kubectl scale deployment hello-serverless --replicas=0
kubectl get pods -l app=hello-serverless
```

No pods running. The "service" is at zero.

### Step 4: Simulate Scale-from-Zero

```bash
kubectl scale deployment hello-serverless --replicas=1
kubectl get pods -l app=hello-serverless --watch
```

Wait for the pod to be ready, then:

```bash
curl http://localhost:5678
```

### Step 5: Understand the Difference

This manual approach shows the concept, but real serverless (Knative) provides:
- **Automatic** scale-to-zero (no manual `kubectl scale`).
- **Request buffering** during cold start (the Activator component).
- **Automatic scale-up** based on concurrent requests.
- **Revision management** and traffic splitting.

### Cleanup

```bash
kubectl delete deployment hello-serverless
kubectl delete service hello-serverless
```

---

## Key Takeaways

1. **Knative Serving** runs serverless workloads on Kubernetes with automatic scale-to-zero.
2. The **Activator** buffers requests when pods are at zero and triggers scale-up.
3. **Cold starts** cause a brief delay when scaling from zero.
4. Each code change creates a new **Revision**, enabling traffic splitting (canary, blue-green).
5. Knative creates standard Kubernetes resources (Deployments, Services) under the hood.
6. **Scale-to-zero** is the key differentiator between serverless and traditional container deployments.
7. Knative requires a networking layer (Kourier, Istio, or Contour).

---

## Next Steps

- Review the [CNCF Ecosystem](../concepts/cncf-ecosystem.md) to understand where Knative fits (CNCF Incubating project).
- Explore [Autoscaling](../concepts/autoscaling.md) for comparison with HPA and KEDA.
