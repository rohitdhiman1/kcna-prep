# Lab 23 — Grafana and Logging

**Objective**: Access Grafana (deployed with kube-prometheus-stack), explore pre-built Kubernetes dashboards, create a simple custom dashboard, deploy Fluent Bit as a DaemonSet, and view collected logs.

---

## Prerequisites

- A running Kubernetes cluster (minikube, kind, or cloud-managed)
- `kubectl` installed and configured
- `helm` (v3) installed
- **kube-prometheus-stack installed** (from Lab 22). If not, run:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring
```

- Concept files:
  - [concepts/observability-tools.md](../concepts/observability-tools.md)
  - [concepts/observability.md](../concepts/observability.md)

---

## Part 1 — Grafana

### Step 1 — Access Grafana

#### Option A: Port Forward

```bash
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

Open your browser to: **http://localhost:3000**

#### Option B: minikube

```bash
minikube service prometheus-grafana -n monitoring
```

### Step 2 — Log In

Default credentials for kube-prometheus-stack Grafana:

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `prom-operator` |

To retrieve the password if you forgot it:

```bash
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

### Step 3 — Explore Pre-Built Dashboards

The kube-prometheus-stack installs many dashboards automatically.

Navigate to **Dashboards > Browse** (or the hamburger menu > Dashboards).

You should see folders and dashboards like:

```
  +-------------------------------------------------------------------+
  |  Dashboards                                                        |
  |                                                                    |
  |  General                                                           |
  |   +-- Kubernetes / Compute Resources / Cluster                     |
  |   +-- Kubernetes / Compute Resources / Namespace (Pods)            |
  |   +-- Kubernetes / Compute Resources / Namespace (Workloads)       |
  |   +-- Kubernetes / Compute Resources / Node (Pods)                 |
  |   +-- Kubernetes / Compute Resources / Pod                         |
  |   +-- Kubernetes / Compute Resources / Workload                    |
  |   +-- Kubernetes / Networking / Cluster                            |
  |   +-- Kubernetes / Networking / Namespace                          |
  |   +-- Kubernetes / Networking / Pod                                |
  |   +-- Node Exporter / Nodes                                        |
  |   +-- Prometheus / Overview                                        |
  |   +-- Alertmanager / Overview                                      |
  |   +-- ... more dashboards ...                                      |
  +-------------------------------------------------------------------+
```

#### 3.1 — Explore "Kubernetes / Compute Resources / Cluster"

Click on this dashboard. It shows cluster-wide CPU and memory usage:

- CPU usage by namespace
- Memory usage by namespace
- CPU quota vs actual usage
- Running Pods count

#### 3.2 — Explore "Kubernetes / Compute Resources / Pod"

Select a namespace and Pod from the dropdowns. This dashboard shows:

- CPU usage for a specific Pod
- Memory usage and working set
- Network bandwidth (receive/transmit)
- Rate of received/transmitted packets

#### 3.3 — Explore "Node Exporter / Nodes"

This dashboard shows node-level hardware metrics:

- CPU busy percentage
- System load
- RAM used
- Disk I/O
- Network traffic

### Step 4 — Explore Data Sources

Navigate to **Configuration > Data Sources** (or the gear icon > Data Sources).

You will see **Prometheus** pre-configured as a data source:

```
  +-------------------------------------------------------------------+
  |  Data Sources                                                      |
  |                                                                    |
  |  +-- Prometheus                                                    |
  |      URL: http://prometheus-kube-prometheus-prometheus:9090         |
  |      Type: Prometheus                                              |
  |      Status: Working (green)                                       |
  +-------------------------------------------------------------------+
```

This is how Grafana queries Prometheus — via an internal Kubernetes Service URL.

### Step 5 — Create a Simple Custom Dashboard

#### 5.1 — Create a New Dashboard

1. Click **+ (Create)** > **Dashboard** > **Add a new panel**

#### 5.2 — Panel 1: CPU Usage by Namespace

In the query editor at the bottom:

1. Make sure the data source is **Prometheus**
2. Enter this PromQL query:

```promql
sum by (namespace) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))
```

3. On the right side panel:
   - Title: `CPU Usage by Namespace`
   - Visualization type: `Time series`

4. Click **Apply**

#### 5.3 — Panel 2: Pod Count by Namespace

1. Click **Add panel** > **Add a new panel**
2. Enter this query:

```promql
count by (namespace) (kube_pod_info)
```

3. On the right side panel:
   - Title: `Pod Count by Namespace`
   - Visualization type: `Bar gauge`

4. Click **Apply**

#### 5.4 — Panel 3: Memory Usage (Cluster-wide)

1. Click **Add panel** > **Add a new panel**
2. Enter this query:

```promql
sum(container_memory_working_set_bytes{container!=""}) / sum(machine_memory_bytes) * 100
```

3. On the right side panel:
   - Title: `Cluster Memory Usage %`
   - Visualization type: `Gauge`
   - Under **Standard options**, set Unit to `Percent (0-100)`
   - Under **Thresholds**, set green < 70, orange < 85, red >= 85

4. Click **Apply**

#### 5.5 — Save the Dashboard

1. Click the **Save** icon (floppy disk) at the top
2. Name it: `My Kubernetes Overview`
3. Click **Save**

Your custom dashboard should look something like:

```
  +-------------------------------------------------------------------+
  |  My Kubernetes Overview                                            |
  |                                                                    |
  |  +--------------------+  +------------------+  +-----------+       |
  |  | CPU Usage by       |  | Pod Count by     |  | Cluster   |       |
  |  | Namespace          |  | Namespace        |  | Memory    |       |
  |  |                    |  |                  |  | Usage %   |       |
  |  | [time series graph]|  | default:    12   |  |           |       |
  |  | showing lines per  |  | kube-system: 8   |  |  [gauge]  |       |
  |  | namespace          |  | monitoring: 6    |  |   45%     |       |
  |  +--------------------+  +------------------+  +-----------+       |
  |                                                                    |
  +-------------------------------------------------------------------+
```

---

## Part 2 — Fluent Bit Logging

### Step 6 — Deploy a Sample Application (Log Source)

First, deploy a simple application that generates logs:

```bash
# Create a namespace for our logging demo
kubectl create namespace logging-demo

# Deploy a Pod that writes logs to stdout
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-generator
  namespace: logging-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: log-generator
  template:
    metadata:
      labels:
        app: log-generator
    spec:
      containers:
      - name: logger
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
        - |
          while true; do
            echo "{\"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"level\":\"INFO\",\"service\":\"log-generator\",\"message\":\"Processing order $(shuf -i 1000-9999 -n 1)\"}"
            sleep 5
            if [ $((RANDOM % 5)) -eq 0 ]; then
              echo "{\"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"level\":\"ERROR\",\"service\":\"log-generator\",\"message\":\"Failed to connect to payment service\"}"
            fi
          done
EOF
```

Verify the Pod is running and generating logs:

```bash
kubectl get pods -n logging-demo
kubectl logs -n logging-demo -l app=log-generator --tail=5
```

### Step 7 — Add the Fluent Helm Repository

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

### Step 8 — Install Fluent Bit as a DaemonSet

```bash
helm install fluent-bit fluent/fluent-bit \
  --namespace monitoring \
  --set config.outputs=|
    [OUTPUT]
        Name   stdout
        Match  *
        Format json_lines
```

Alternatively, install with default settings and we will inspect the configuration:

```bash
helm install fluent-bit fluent/fluent-bit --namespace monitoring
```

Verify Fluent Bit is running as a DaemonSet:

```bash
# Check the DaemonSet
kubectl get daemonset -n monitoring fluent-bit

# Check that a Fluent Bit Pod is running on each node
kubectl get pods -n monitoring -l app.kubernetes.io/name=fluent-bit -o wide
```

Expected output:

```
NAME                READY   STATUS    RESTARTS   AGE   NODE
fluent-bit-xxxxx    1/1     Running   0          1m    minikube
```

### Step 9 — Understand the Fluent Bit Configuration

View the ConfigMap that defines the Fluent Bit pipeline:

```bash
kubectl get configmap -n monitoring fluent-bit -o yaml
```

The configuration defines the log pipeline:

```
  +-------------------------------------------------------------------+
  |  Fluent Bit Pipeline (from ConfigMap)                              |
  |                                                                    |
  |  [INPUT]                                                           |
  |    Name              tail                                          |
  |    Path              /var/log/containers/*.log                     |
  |    Tag               kube.*                                        |
  |    (reads container log files from the node)                       |
  |                                                                    |
  |  [FILTER]                                                          |
  |    Name              kubernetes                                    |
  |    Match             kube.*                                        |
  |    (enriches logs with Kubernetes metadata:                        |
  |     pod name, namespace, labels, etc.)                             |
  |                                                                    |
  |  [OUTPUT]                                                          |
  |    Name              es / stdout / forward / ...                   |
  |    Match             *                                             |
  |    (sends logs to the configured destination)                      |
  +-------------------------------------------------------------------+
```

### Step 10 — View Collected Logs

Check Fluent Bit's own logs to see it collecting and processing container logs:

```bash
# View Fluent Bit Pod logs
kubectl logs -n monitoring -l app.kubernetes.io/name=fluent-bit --tail=20
```

You should see Fluent Bit processing logs from your `log-generator` Pod and other cluster Pods.

### Step 11 — Inspect How Fluent Bit Collects Logs

Fluent Bit runs on each node and reads log files that the container runtime writes:

```bash
# Exec into the Fluent Bit Pod to see the log files it reads
FLUENT_POD=$(kubectl get pods -n monitoring -l app.kubernetes.io/name=fluent-bit -o jsonpath='{.items[0].metadata.name}')

# List container log files on the node (mounted into the Fluent Bit Pod)
kubectl exec -n monitoring $FLUENT_POD -- ls /var/log/containers/ | head -10
```

You will see files like:

```
log-generator-xxxxxxxxxx-xxxxx_logging-demo_logger-xxxxxxxxxx.log
prometheus-grafana-xxxxxxxxxx-xxxxx_monitoring_grafana-xxxxxxxxxx.log
coredns-xxxxxxxxxx-xxxxx_kube-system_coredns-xxxxxxxxxx.log
```

Each file is a symlink to the actual log file, and Fluent Bit tails all of them.

### Step 12 — Modify Fluent Bit to Output to stdout (for Debugging)

If Fluent Bit is not configured to output to stdout by default, you can upgrade the Helm release:

```bash
helm upgrade fluent-bit fluent/fluent-bit \
  --namespace monitoring \
  --set config.outputs="[OUTPUT]\n    Name stdout\n    Match *\n    Format json_lines"
```

Then view the logs again:

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=fluent-bit --tail=30
```

You should see JSON-formatted log lines from all containers in the cluster, enriched with Kubernetes metadata (pod name, namespace, container name, labels).

---

## Step 13 — Clean Up

```bash
# Remove Fluent Bit
helm uninstall fluent-bit -n monitoring

# Remove the log generator
kubectl delete namespace logging-demo

# (Optional) Remove the entire monitoring stack
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring
```

---

## Key Takeaways

1. **Grafana is a visualization layer**: It does not store data — it queries data sources like Prometheus and displays the results in dashboards.

2. **kube-prometheus-stack** comes with dozens of pre-built Kubernetes dashboards covering cluster, namespace, node, and Pod-level metrics.

3. **Custom dashboards** are built by writing PromQL queries and choosing visualization types (time series, gauge, bar, table, etc.).

4. **Fluent Bit runs as a DaemonSet**: One Pod per node, reading container logs from `/var/log/containers/*.log`.

5. **Fluent Bit pipeline**: INPUT (collect from files) -> FILTER (enrich with Kubernetes metadata) -> OUTPUT (send to Elasticsearch, Loki, stdout, etc.).

6. **Kubernetes container logging**: Containers write to stdout/stderr, the container runtime captures this to log files on the node, and a log collector (Fluent Bit) tails those files and forwards them.

7. **Grafana data sources**: Grafana connects to backends like Prometheus, Elasticsearch, Loki, and Jaeger using internal Kubernetes Service DNS names.
