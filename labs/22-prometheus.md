# Lab 22 — Prometheus

**Objective**: Install Prometheus in a Kubernetes cluster using Helm (kube-prometheus-stack), access the Prometheus UI, run basic PromQL queries, explore Kubernetes metrics, and view alert rules.

---

## Prerequisites

- A running Kubernetes cluster (minikube, kind, or cloud-managed)
- `kubectl` installed and configured
- `helm` (v3) installed
- Concept file: [concepts/prometheus.md](../concepts/prometheus.md)

---

## Step 1 — Add the Prometheus Helm Repository

The `kube-prometheus-stack` chart bundles Prometheus, Alertmanager, Grafana, and pre-configured Kubernetes monitoring dashboards and alert rules.

```bash
# Add the prometheus-community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update your local Helm chart index
helm repo update
```

Verify the repo was added:

```bash
helm repo list
```

You should see `prometheus-community` in the output.

---

## Step 2 — Install kube-prometheus-stack

Create a dedicated namespace and install the stack:

```bash
# Create the monitoring namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.service.type=NodePort \
  --set prometheus.service.nodePort=30090
```

Wait for all Pods to be running:

```bash
kubectl get pods -n monitoring -w
```

Expected output (names will vary):

```
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-kube-prometheus-alertmanager-0    2/2     Running   0          2m
prometheus-grafana-xxxxxxxxxx-xxxxx                      3/3     Running   0          2m
prometheus-kube-prometheus-operator-xxxxxxxxxx-xxxxx      1/1     Running   0          2m
prometheus-kube-state-metrics-xxxxxxxxxx-xxxxx            1/1     Running   0          2m
prometheus-prometheus-node-exporter-xxxxx                 1/1     Running   0          2m
prometheus-prometheus-kube-prometheus-prometheus-0         2/2     Running   0          2m
```

Explore what was installed:

```bash
# List all resources in the monitoring namespace
kubectl get all -n monitoring

# List the ServiceMonitor CRDs (how Prometheus discovers targets)
kubectl get servicemonitors -n monitoring
```

---

## Step 3 — Access the Prometheus UI

### Option A: Port Forward (works with any cluster)

```bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
```

Then open your browser to: **http://localhost:9090**

### Option B: NodePort (if configured above)

If using minikube:

```bash
minikube service prometheus-kube-prometheus-prometheus -n monitoring
```

Or access directly at: **http://<node-ip>:30090**

---

## Step 4 — Explore the Prometheus UI

Once you have the Prometheus UI open, explore these sections:

### 4.1 — Status > Targets

Navigate to **Status > Targets** in the top menu.

This page shows all the endpoints Prometheus is scraping:

```
  +-------------------------------------------------------------------+
  |  Status > Targets                                                  |
  |                                                                    |
  |  serviceMonitor/monitoring/prometheus-kube-prometheus-apiserver     |
  |  (1/1 up)  Endpoint: https://10.96.0.1:443/metrics                |
  |                                                                    |
  |  serviceMonitor/monitoring/prometheus-kube-prometheus-kubelet       |
  |  (1/1 up)  Endpoint: https://192.168.49.2:10250/metrics           |
  |                                                                    |
  |  serviceMonitor/monitoring/prometheus-kube-prometheus-node-exporter |
  |  (1/1 up)  Endpoint: http://192.168.49.2:9100/metrics             |
  |                                                                    |
  |  ... more targets ...                                              |
  +-------------------------------------------------------------------+
```

Note that Prometheus automatically discovered these targets via **ServiceMonitor** CRDs in Kubernetes.

### 4.2 — Status > Configuration

Navigate to **Status > Configuration** to see the full Prometheus scrape configuration. This was auto-generated from ServiceMonitor CRDs.

### 4.3 — Status > Runtime & Build Information

Shows the Prometheus version, storage retention, and runtime details.

---

## Step 5 — Run Basic PromQL Queries

Navigate to the **Graph** tab (main page). Enter these queries in the expression box and click **Execute**.

### Query 1: Total Running Containers

```promql
kubelet_running_containers
```

Switch to the **Graph** tab to see the time-series visualization.

### Query 2: Node CPU Usage

```promql
rate(node_cpu_seconds_total{mode="idle"}[5m])
```

This shows the per-second rate of idle CPU time over the last 5 minutes.

### Query 3: Total HTTP Requests to the API Server

```promql
rate(apiserver_request_total[5m])
```

### Query 4: API Server Request Rate by Verb

```promql
sum by (verb) (rate(apiserver_request_total[5m]))
```

This groups the request rate by HTTP verb (GET, LIST, WATCH, POST, etc.).

### Query 5: Memory Usage per Node

```promql
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

This calculates used memory (total minus available).

### Query 6: Pods in Non-Running State

```promql
kube_pod_status_phase{phase!="Running", phase!="Succeeded"}
```

### Query 7: Container Restarts

```promql
sum by (pod) (kube_pod_container_status_restarts_total)
```

Shows restart counts grouped by Pod name.

---

## Step 6 — Explore Kubernetes Metrics

Prometheus scrapes many Kubernetes metrics automatically. Here are the key metric sources:

### 6.1 — kube-state-metrics

These metrics describe the **state of Kubernetes objects** (Deployments, Pods, Nodes):

```promql
# Number of desired replicas per deployment
kube_deployment_spec_replicas

# Number of available replicas per deployment
kube_deployment_status_replicas_available

# Pod status phases
kube_pod_status_phase

# Node conditions
kube_node_status_condition{condition="Ready"}
```

### 6.2 — node-exporter Metrics

These metrics describe **hardware and OS-level** data on each node:

```promql
# Filesystem available bytes
node_filesystem_avail_bytes

# Network receive bytes rate
rate(node_network_receive_bytes_total[5m])

# System load (1-minute average)
node_load1
```

### 6.3 — kubelet Metrics

These metrics come from the kubelet on each node:

```promql
# Running pods per node
kubelet_running_pods

# Container start duration
kubelet_container_start_duration_seconds_count
```

---

## Step 7 — View Alert Rules

### 7.1 — View in the UI

Navigate to **Alerts** in the top menu. You will see pre-configured alert rules organized by groups:

```
  +-------------------------------------------------------------------+
  |  Alerts                                                            |
  |                                                                    |
  |  kubernetes-apps (4 rules)                                         |
  |    [inactive] KubePodCrashLooping                                  |
  |    [inactive] KubePodNotReady                                      |
  |    [inactive] KubeDeploymentReplicasMismatch                       |
  |    [inactive] KubeContainerWaiting                                 |
  |                                                                    |
  |  kubernetes-system (3 rules)                                       |
  |    [inactive] KubeNodeNotReady                                     |
  |    [inactive] KubeNodeUnreachable                                  |
  |    [inactive] KubeletTooManyPods                                   |
  |                                                                    |
  |  ... many more groups ...                                          |
  +-------------------------------------------------------------------+
```

### 7.2 — View Alert Rules via kubectl

The kube-prometheus-stack uses Kubernetes CRDs called `PrometheusRule`:

```bash
# List all PrometheusRule resources
kubectl get prometheusrules -n monitoring

# View a specific rule in detail
kubectl get prometheusrule prometheus-kube-prometheus-kubernetes-apps \
  -n monitoring -o yaml
```

### 7.3 — Understand an Alert Rule

Example alert rule for Pod crash looping:

```yaml
- alert: KubePodCrashLooping
  expr: |
    max_over_time(kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}[5m]) >= 1
  for: 15m
  labels:
    severity: warning
  annotations:
    description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
```

Key fields:
- `expr` — The PromQL expression that triggers the alert
- `for` — How long the condition must be true before firing
- `labels.severity` — Used by Alertmanager for routing
- `annotations` — Human-readable descriptions

---

## Step 8 — Trigger a Test Alert (Optional)

Create a Pod that will crash loop to see an alert fire:

```bash
# Create a Pod that crashes immediately
kubectl run crasher --image=busybox -- /bin/sh -c "exit 1"
```

Wait 15 minutes, then check the Alerts page in the Prometheus UI. You should see `KubePodCrashLooping` change from `inactive` to `pending` to `firing`.

Clean up:

```bash
kubectl delete pod crasher
```

---

## Step 9 — Clean Up

To remove the entire Prometheus stack:

```bash
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring
```

To remove CRDs (they persist after Helm uninstall):

```bash
kubectl delete crd alertmanagerconfigs.monitoring.coreos.com
kubectl delete crd alertmanagers.monitoring.coreos.com
kubectl delete crd podmonitors.monitoring.coreos.com
kubectl delete crd probes.monitoring.coreos.com
kubectl delete crd prometheusagents.monitoring.coreos.com
kubectl delete crd prometheuses.monitoring.coreos.com
kubectl delete crd prometheusrules.monitoring.coreos.com
kubectl delete crd scrapeconfigs.monitoring.coreos.com
kubectl delete crd servicemonitors.monitoring.coreos.com
kubectl delete crd thanosrulers.monitoring.coreos.com
```

---

## Key Takeaways

1. **kube-prometheus-stack** is the standard Helm chart for deploying Prometheus + Alertmanager + Grafana + pre-built Kubernetes monitoring.

2. **ServiceMonitor CRDs** tell Prometheus which services to scrape — this is how Kubernetes service discovery works in practice.

3. **kube-state-metrics** provides Kubernetes object state metrics; **node-exporter** provides hardware/OS metrics. Both are automatically deployed by the stack.

4. **PromQL** lets you query metrics with functions like `rate()`, `sum()`, and `sum by()` for grouping.

5. **PrometheusRule CRDs** define alert rules declaratively in Kubernetes. The stack comes with dozens of pre-configured Kubernetes alerts.
