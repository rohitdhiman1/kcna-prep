# Lab 20: Autoscaling in Kubernetes

## Objective

Deploy a workload, configure Horizontal Pod Autoscaler (HPA), generate load, and observe pods scaling up and down automatically.

## Prerequisites

- A running Kubernetes cluster (kind, minikube, or cloud).
- `kubectl` installed and configured.
- Read the concept file: [Autoscaling](../concepts/autoscaling.md)

**Note:** If using minikube, enable metrics-server with:
```bash
minikube addons enable metrics-server
```

---

## Step 1: Verify or Install metrics-server

HPA requires metrics-server to collect CPU and memory metrics from pods.

### Check if metrics-server is already running

```bash
kubectl get pods -n kube-system | grep metrics-server
```

If you see a running `metrics-server` pod, skip to Step 2.

### Install metrics-server (if not present)

For kind or bare clusters, install metrics-server:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**For kind clusters**, metrics-server may fail due to TLS. Patch it to allow insecure TLS:

```bash
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/args/-",
      "value": "--kubelet-insecure-tls"
    }
  ]'
```

### Verify metrics-server is working

Wait about 60 seconds, then run:

```bash
kubectl top nodes
```

You should see CPU and memory usage for your nodes. If you get an error, wait a bit longer.

```bash
kubectl top pods -A
```

This shows resource usage for all pods across all namespaces.

---

## Step 2: Create a Test Deployment

Create a simple PHP Apache server deployment that consumes CPU when handling requests.

```bash
kubectl create deployment php-apache \
  --image=registry.k8s.io/hpa-example \
  --requests='cpu=200m' \
  --expose --port=80
```

This does two things:
1. Creates a **Deployment** named `php-apache` with a container that has `cpu: 200m` request.
2. Creates a **Service** named `php-apache` exposing port 80.

### Verify the deployment is running

```bash
kubectl get deployment php-apache
kubectl get service php-apache
kubectl get pods -l app=php-apache
```

Wait until the pod is in `Running` state.

---

## Step 3: Create a Horizontal Pod Autoscaler

Create an HPA that targets 50% average CPU utilization:

```bash
kubectl autoscale deployment php-apache \
  --cpu-percent=50 \
  --min=1 \
  --max=10
```

This tells Kubernetes:
- Keep CPU utilization at **50%** of the requested CPU (200m * 50% = 100m).
- Scale between **1** and **10** replicas.

### Verify the HPA

```bash
kubectl get hpa php-apache
```

Expected output (after a minute for metrics to populate):

```
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          30s
```

The `TARGETS` column shows `current%/target%`. Initially it may show `<unknown>/50%` until metrics-server reports data.

### Examine HPA details

```bash
kubectl describe hpa php-apache
```

Look at the `Conditions` section and events for scaling decisions.

---

## Step 4: Generate Load

Open a **second terminal** (or use `&` to run in background). Start a busybox pod that continuously sends requests to the php-apache service:

```bash
kubectl run -i --tty load-generator --rm --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

This creates a pod called `load-generator` that hits the php-apache service every 10ms.

**Keep this running.** Do not close this terminal.

---

## Step 5: Watch Pods Scale Up

In your **first terminal**, watch the HPA and pods:

```bash
kubectl get hpa php-apache --watch
```

After 1-2 minutes, you should see the CPU percentage rise and replicas increase:

```
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%     1         10        1          2m
php-apache   Deployment/php-apache   250%/50%   1         10        1          3m
php-apache   Deployment/php-apache   250%/50%   1         10        5          3m30s
php-apache   Deployment/php-apache   68%/50%    1         10        7          4m
php-apache   Deployment/php-apache   45%/50%    1         10        7          5m
```

Press `Ctrl+C` to stop watching, then check pods:

```bash
kubectl get pods -l app=php-apache
```

You should see multiple replicas running.

### Understand the scaling

The HPA controller:
1. Queries metrics-server for CPU usage every 15 seconds.
2. Calculates desired replicas: `ceil(currentReplicas * (currentCPU / targetCPU))`.
3. Adjusts the Deployment's replica count.

---

## Step 6: Watch Pods Scale Down

Stop the load generator:
- Go to the second terminal and press `Ctrl+C`.
- Or delete the pod: `kubectl delete pod load-generator`

Now watch the HPA scale down:

```bash
kubectl get hpa php-apache --watch
```

**Note:** Scale-down is slower by design. The default cooldown for scale-down is **5 minutes**. You will need to wait 5-10 minutes to see replicas decrease.

```
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   45%/50%   1         10        7          8m
php-apache   Deployment/php-apache   0%/50%    1         10        7          10m
php-apache   Deployment/php-apache   0%/50%    1         10        1          15m
```

Eventually replicas will return to 1 (the minimum).

---

## Step 7: Explore HPA with YAML

View the HPA resource in YAML format to understand its full structure:

```bash
kubectl get hpa php-apache -o yaml
```

Key fields to note:
- `spec.scaleTargetRef` — what resource the HPA controls.
- `spec.minReplicas` and `spec.maxReplicas` — scaling bounds.
- `spec.metrics` — what metrics drive scaling.
- `status.currentReplicas` — current replica count.
- `status.currentMetrics` — current metric values.

---

## Step 8: Create HPA from YAML (Advanced)

Create a more detailed HPA using a YAML manifest:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-v2
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 2
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 120
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60
EOF
```

This HPA:
- Uses **autoscaling/v2** API for advanced features.
- Scales on both **CPU and memory**.
- Has custom **scaling behavior**: scale up by max 4 pods per minute, scale down by max 50% per minute.
- Has a shorter stabilization window for scale-down (120 seconds instead of the default 300).

Delete the previous HPA first to avoid conflicts:

```bash
kubectl delete hpa php-apache
```

Check the new HPA:

```bash
kubectl get hpa php-apache-v2
kubectl describe hpa php-apache-v2
```

---

## Cleanup

```bash
kubectl delete hpa php-apache-v2
kubectl delete deployment php-apache
kubectl delete service php-apache
kubectl delete pod load-generator --ignore-not-found
```

---

## Key Takeaways

1. **metrics-server** is required for HPA to work with CPU/memory metrics.
2. Pods must have **resource requests** set for HPA to calculate utilization.
3. HPA checks metrics every **15 seconds** by default.
4. Scale-up is fast; scale-down is intentionally slow (5 min cooldown) to prevent flapping.
5. The `autoscaling/v2` API supports multiple metrics and custom scaling behavior.
6. Use `kubectl get hpa --watch` to observe scaling decisions in real time.
7. The formula: `desiredReplicas = ceil(currentReplicas * currentMetric / targetMetric)`.

---

## Next Steps

- Read about [Serverless](../concepts/serverless.md) to understand scale-to-zero (which HPA cannot do).
- Try [Lab 21: Serverless with Knative](21-serverless.md) for event-driven autoscaling.
