# Prometheus

**Prometheus is a CNCF Graduated open-source monitoring and alerting system that uses a pull-based model to scrape numeric metrics from target endpoints, stores them as time-series data, and provides a powerful query language (PromQL) for analysis and alerting.**

---

## Table of Contents

- [Why Prometheus](#why-prometheus)
- [Pull-Based Model](#pull-based-model)
- [Prometheus Architecture](#prometheus-architecture)
- [Components in Detail](#components-in-detail)
- [Metric Types](#metric-types)
- [PromQL Basics](#promql-basics)
- [Service Discovery in Kubernetes](#service-discovery-in-kubernetes)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## Why Prometheus

Prometheus is the **de facto standard** for metrics in the Kubernetes ecosystem:

- **CNCF Graduated** project (second project to graduate after Kubernetes itself)
- Built specifically for dynamic, cloud native environments
- Native Kubernetes integration with service discovery
- Dimensional data model with key-value labels
- Powerful query language (PromQL)
- Built-in alerting via Alertmanager

---

## Pull-Based Model

Prometheus uses a **pull-based** (scrape) model, which is the opposite of traditional push-based monitoring:

```
  PUSH-BASED (traditional)              PULL-BASED (Prometheus)

  +-------+                             +-------+
  | App A |---push--->+            +--->| App A | /metrics
  +-------+           |            |    +-------+
                       v            |
  +-------+     +----------+     +-----------+
  | App B |---->| Monitoring|     | Prometheus|---scrape every 15s--->
  +-------+     | Server   |     | Server    |
                +----------+     +-----------+
  +-------+           ^            |    +-------+
  | App C |---push--->+            +--->| App C | /metrics
  +-------+                             +-------+

  Apps must know where to send        Prometheus decides what to scrape.
  data. If monitoring is down,        Apps just expose /metrics endpoint.
  data is lost.                       Simple, reliable, discoverable.
```

**Advantages of pull-based:**
- Prometheus controls the scrape frequency
- Easy to detect if a target is down (scrape fails)
- Targets don't need to know about Prometheus
- Works naturally with Kubernetes service discovery

---

## Prometheus Architecture

```
+===========================================================================+
|                        PROMETHEUS ECOSYSTEM                               |
|                                                                           |
|                        Service Discovery                                  |
|                     (Kubernetes API, DNS, etc.)                            |
|                              |                                            |
|                              | discover targets                           |
|                              v                                            |
|  +---------------------------------------------------------------+       |
|  |                    PROMETHEUS SERVER                            |       |
|  |                                                                |       |
|  |  +------------------+  +--------------+  +-----------------+   |       |
|  |  |    Retrieval      |  |    TSDB      |  |   HTTP Server  |   |       |
|  |  |    (Scraper)      |  | (Time Series |  |                |   |       |
|  |  |                   |  |  Database)   |  |  /api/v1/query |   |       |
|  |  | - pulls /metrics  |  |              |  |  /graph        |   |       |
|  |  |   from targets    |->| - stores     |->|                |   |       |
|  |  | - every 15s       |  |   samples    |  | Serves PromQL  |   |       |
|  |  |   (configurable)  |  | - local disk |  | queries        |   |       |
|  |  +------------------+  +--------------+  +-----------------+   |       |
|  |                                                                |       |
|  +-----+----------------------------------------------+-----------+       |
|        |                                              |                   |
|        | pushes alerts                                | queries           |
|        v                                              v                   |
|  +---------------+                            +--------------+            |
|  | ALERTMANAGER  |                            |   GRAFANA    |            |
|  |               |                            |              |            |
|  | - dedup       |                            | - dashboards |            |
|  | - grouping    |                            | - graphs     |            |
|  | - routing     |                            | - panels     |            |
|  | - silencing   |                            |              |            |
|  +-------+-------+                            +--------------+            |
|          |                                                                |
|          v                                                                |
|  +---------------+                                                        |
|  | Notifications |                                                        |
|  | - Slack       |                                                        |
|  | - PagerDuty   |                                                        |
|  | - Email       |                                                        |
|  +---------------+                                                        |
|                                                                           |
|                                                                           |
|  TARGETS (things being scraped):                                          |
|                                                                           |
|  +----------+  +----------+  +----------+  +--------------+              |
|  | App Pod  |  | App Pod  |  | App Pod  |  | PUSHGATEWAY  |              |
|  | /metrics |  | /metrics |  | /metrics |  |              |              |
|  +----------+  +----------+  +----------+  | For short-   |              |
|                                             | lived jobs   |              |
|  +----------+  +----------+  +----------+  | that push    |              |
|  | node-    |  | kube-    |  | custom   |  | metrics here |              |
|  | exporter |  | state-   |  | exporter |  +--------------+              |
|  | /metrics |  | metrics  |  | /metrics |                                |
|  +----------+  +----------+  +----------+                                |
|                                                                           |
|   EXPORTERS: expose metrics for systems that don't natively               |
|   have a /metrics endpoint (databases, hardware, etc.)                    |
+===========================================================================+
```

---

## Components in Detail

### Prometheus Server

The core component. It does three things:

1. **Scrapes** metrics from configured targets at regular intervals
2. **Stores** time-series data in its built-in TSDB (Time Series Database) on local disk
3. **Serves** PromQL queries via HTTP API

### Alertmanager

Handles alerts generated by Prometheus alerting rules:

- **Deduplication** — Avoids sending the same alert multiple times
- **Grouping** — Combines related alerts into a single notification
- **Routing** — Sends alerts to the right team/channel based on labels
- **Silencing** — Temporarily mutes alerts (e.g., during maintenance)
- **Inhibition** — Suppresses certain alerts when others are already firing

```
  Prometheus                  Alertmanager              Notification
  +-----------+              +---------------+          +----------+
  | Alert     |   fires      | Dedup         |  routes  | Slack    |
  | Rules     | -----------> | Group         | -------> | PagerDuty|
  |           |              | Route         |          | Email    |
  +-----------+              | Silence       |          +----------+
                              +---------------+
```

### Pushgateway

A special intermediary for **short-lived jobs** (batch jobs, cron jobs) that may finish before Prometheus can scrape them:

```
  Short-lived Job                    Pushgateway              Prometheus
  +-------------+    push metrics    +------------+   scrape  +----------+
  | Batch Job   | -----------------> | Stores     | <-------- | Server   |
  | (exits in   |                    | last push  |           |          |
  |  30 seconds)|                    +------------+           +----------+
  +-------------+
```

**Note**: The Pushgateway is not for turning Prometheus into a push-based system. It is only for short-lived jobs.

### Exporters

Exporters are adapters that **expose metrics in Prometheus format** for third-party systems that do not natively have a `/metrics` endpoint:

| Exporter | Purpose |
|----------|---------|
| node-exporter | Hardware/OS metrics (CPU, memory, disk, network) |
| kube-state-metrics | Kubernetes object state (Deployments, Pods, Nodes) |
| mysqld-exporter | MySQL database metrics |
| blackbox-exporter | Probe endpoints (HTTP, DNS, TCP, ICMP) |

---

## Metric Types

Prometheus defines four core metric types:

### 1. Counter

A **monotonically increasing** value that only goes up (or resets to zero on restart).

- Use for: Total requests, errors, bytes sent
- Example: `http_requests_total`

```
  Value
   500 |                                          *
   400 |                                    *
   300 |                              *
   200 |                   *    *
   100 |        *    *
     0 +---*----+----+----+----+----+----+----> Time

   Only goes UP. Never decreases.
   (Resets to 0 only on process restart)
```

### 2. Gauge

A value that can **go up and down**.

- Use for: Temperature, memory usage, queue size, active connections
- Example: `node_memory_available_bytes`

```
  Value
   80% |        *         *
   60% |   *         *         *
   40% |                            *    *
   20% |                                      *
    0% +----+----+----+----+----+----+----+----> Time

   Goes UP and DOWN freely.
```

### 3. Histogram

Records observations and puts them into **configurable buckets**. Useful for measuring request durations or response sizes.

- Use for: Request latency, response size
- Example: `http_request_duration_seconds`
- Automatically provides: `_bucket`, `_sum`, `_count`

```
  Buckets for http_request_duration_seconds:

  Bucket (le)     Count
  +---------+     +-----+
  | <= 0.01 | --> |  50 |  50 requests under 10ms
  | <= 0.05 | --> | 180 |  180 requests under 50ms (cumulative)
  | <= 0.1  | --> | 250 |  250 requests under 100ms
  | <= 0.5  | --> | 290 |  290 requests under 500ms
  | <= 1.0  | --> | 298 |  298 requests under 1s
  | <= +Inf | --> | 300 |  300 total requests
  +---------+     +-----+
```

### 4. Summary

Similar to histogram but calculates **quantiles** (percentiles) on the client side.

- Use for: Pre-calculated percentiles when you know which quantiles you need
- Example: `go_gc_duration_seconds{quantile="0.99"}`

```
  Summary: go_gc_duration_seconds

  Quantile    Value
  +--------+  +---------+
  | 0.5    |  | 0.0003s |  Median (50th percentile)
  | 0.9    |  | 0.0008s |  90th percentile
  | 0.99   |  | 0.0021s |  99th percentile
  +--------+  +---------+
  _count = 15000
  _sum   = 4.5s
```

### Histogram vs Summary

| Feature | Histogram | Summary |
|---------|-----------|---------|
| Bucket/quantile config | Server-side (Prometheus) | Client-side (application) |
| Aggregatable across instances | Yes | No |
| CPU cost | Cheap on client | Expensive on client |
| Recommended for | Most use cases | Pre-defined quantiles |

---

## PromQL Basics

PromQL (Prometheus Query Language) is the query language used to select and aggregate time-series data. For the KCNA exam, you need **awareness-level** understanding.

### Selecting Metrics

```promql
# Select all time series for a metric
http_requests_total

# Filter by label
http_requests_total{method="GET", status="200"}

# Regex match on label
http_requests_total{method=~"GET|POST"}
```

### Common Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `rate()` | Per-second rate of increase for counters | `rate(http_requests_total[5m])` |
| `sum()` | Sum across all matching series | `sum(rate(http_requests_total[5m]))` |
| `avg()` | Average across all matching series | `avg(node_cpu_seconds_total)` |
| `increase()` | Total increase over a time range | `increase(http_requests_total[1h])` |
| `histogram_quantile()` | Calculate quantile from histogram | `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))` |

### Practical Examples

```promql
# Request rate per second (last 5 minutes)
rate(http_requests_total[5m])

# Total request rate across all pods
sum(rate(http_requests_total[5m]))

# Request rate broken down by status code
sum by (status) (rate(http_requests_total[5m]))

# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

---

## Service Discovery in Kubernetes

Prometheus natively integrates with the Kubernetes API to **automatically discover scrape targets**. No manual configuration of target IPs is needed.

```
  +-------------------+           +-----------------------+
  |  Kubernetes API   |           |   Prometheus Server    |
  |                   |  watches  |                        |
  |  Pods             |<----------|  Service Discovery     |
  |  Services         |           |  module queries the    |
  |  Endpoints        |           |  Kubernetes API for    |
  |  Nodes            |           |  targets with specific |
  |                   |           |  annotations           |
  +-------------------+           +-----------------------+
                                           |
                                           | scrapes discovered targets
                                           v
                                  +---------+---------+
                                  |   |   |   |   |   |
                                  v   v   v   v   v   v
                                 Pod Pod Pod Pod Pod Pod
                                 /metrics endpoints
```

### How It Works

1. Prometheus queries the Kubernetes API for Pods, Services, Endpoints, or Nodes
2. It looks for specific **annotations** on Pods/Services:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"     # Enable scraping
    prometheus.io/port: "8080"       # Port to scrape
    prometheus.io/path: "/metrics"   # Path to scrape (default: /metrics)
```

3. Prometheus automatically updates its target list as Pods are created/destroyed
4. No restart or config reload needed — fully dynamic

### Discovery Roles

| Role | Discovers |
|------|-----------|
| `node` | Each cluster node (kubelet metrics) |
| `pod` | All Pods (application metrics) |
| `service` | Services (service-level metrics) |
| `endpoints` | Endpoints behind Services |
| `ingress` | Ingress objects (for blackbox probing) |

---

## What to Remember for the Exam

1. **Pull-based model**: Prometheus **scrapes** (pulls) metrics from targets' `/metrics` endpoints at regular intervals. This is opposite to push-based systems.

2. **CNCF Graduated**: Prometheus was the second project to graduate from the CNCF (after Kubernetes).

3. **Four components**: Prometheus Server (scrape + store + query), Alertmanager (alert routing/dedup), Pushgateway (for short-lived jobs), Exporters (adapt third-party systems).

4. **Four metric types**:
   - **Counter** — only goes up (requests, errors)
   - **Gauge** — goes up and down (temperature, memory)
   - **Histogram** — bucketed observations (latency distribution)
   - **Summary** — client-side quantiles (percentiles)

5. **PromQL**: Know that `rate()` calculates per-second rate for counters, `sum()` aggregates across series, and `avg()` computes averages.

6. **Service discovery**: Prometheus auto-discovers Kubernetes targets via the Kubernetes API. Pods annotated with `prometheus.io/scrape: "true"` are automatically scraped.

7. **Alertmanager**: Handles deduplication, grouping, routing, and silencing of alerts. Prometheus fires alerts; Alertmanager manages notifications.

8. **Time-series database**: Prometheus stores data in its own efficient TSDB on local disk. It is not designed for long-term storage (typically 15-30 days retention).

9. **Exporters**: node-exporter (hardware metrics) and kube-state-metrics (Kubernetes object state) are the two most common exporters in a Kubernetes cluster.

10. **Pushgateway exception**: The only time metrics are pushed to Prometheus is via the Pushgateway, used exclusively for short-lived batch jobs.
