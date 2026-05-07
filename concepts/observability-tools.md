# Observability Tools

**The cloud native observability ecosystem includes Grafana for visualization, Fluentd/Fluent Bit for log collection, and Jaeger/Zipkin for distributed tracing — each tool specializes in one aspect and they are commonly combined into integrated stacks.**

---

## Table of Contents

- [Grafana — Visualization and Dashboards](#grafana--visualization-and-dashboards)
- [Log Collection — Fluentd and Fluent Bit](#log-collection--fluentd-and-fluent-bit)
- [Log Stacks — EFK and ELK](#log-stacks--efk-and-elk)
- [Distributed Tracing — Jaeger and Zipkin](#distributed-tracing--jaeger-and-zipkin)
- [Tool Comparison Table](#tool-comparison-table)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## Grafana — Visualization and Dashboards

Grafana is an **open-source visualization and analytics platform** that turns metrics, logs, and traces from multiple data sources into rich, interactive dashboards.

### Key Features

- **Multi-data-source support**: Connects to Prometheus, Elasticsearch, Loki, Jaeger, InfluxDB, CloudWatch, and many more
- **Pre-built dashboards**: Thousands of community dashboards available (e.g., Kubernetes cluster monitoring)
- **Alerting**: Can define alert rules and send notifications (Slack, PagerDuty, email)
- **Explore mode**: Ad-hoc querying of data sources for investigation
- **CNCF Graduated** project

### How Grafana Fits In

Grafana does **not collect or store data** — it is purely a visualization layer:

```
  Data Sources                          Grafana                  Users
  +------------+                    +---------------+
  | Prometheus | ---- metrics ----> |               |
  +------------+                    |  Dashboards   |        +---------+
  +------------+                    |  Graphs       | -----> | Browser |
  | Loki       | ---- logs -------> |  Panels       |        | (UI)    |
  +------------+                    |  Alerts       |        +---------+
  +------------+                    |               |
  | Jaeger     | ---- traces -----> |  Explore mode |
  +------------+                    +---------------+
```

### Common Grafana Dashboard Panels

```
  +-----------------------------------------------------------------------+
  |  Kubernetes Cluster Dashboard                                          |
  |                                                                        |
  |  +------------------+  +------------------+  +------------------+      |
  |  | CPU Usage        |  | Memory Usage     |  | Pod Count        |      |
  |  | [line graph]     |  | [gauge: 67%]     |  | [stat: 42]       |      |
  |  +------------------+  +------------------+  +------------------+      |
  |                                                                        |
  |  +------------------+  +------------------+  +------------------+      |
  |  | Network I/O      |  | Disk Usage       |  | HTTP Error Rate  |      |
  |  | [area chart]     |  | [bar chart]      |  | [stat: 0.2%]     |      |
  |  +------------------+  +------------------+  +------------------+      |
  |                                                                        |
  |  +-------------------------------------------------------------------+ |
  |  | Pod Status Table                                                    | |
  |  | Name          | Namespace  | CPU  | Memory | Status   | Restarts | | |
  |  | nginx-abc     | default    | 12m  | 64Mi   | Running  | 0        | | |
  |  | redis-xyz     | cache      | 8m   | 128Mi  | Running  | 1        | | |
  |  +-------------------------------------------------------------------+ |
  +-----------------------------------------------------------------------+
```

---

## Log Collection — Fluentd and Fluent Bit

In Kubernetes, containers write logs to **stdout/stderr**, and these logs need to be collected from every node and forwarded to a central store. That is the job of Fluentd and Fluent Bit.

### How Container Logging Works in Kubernetes

```
  Pod (container)
  +----------------+
  | App writes to  |
  | stdout/stderr  |
  +-------+--------+
          |
          v
  Container Runtime captures output
          |
          v
  /var/log/containers/*.log  (on the Node)
          |
          v
  +-------------------+         +------------------+
  | Fluent Bit or     |  --->   | Central Log Store |
  | Fluentd (DaemonSet)|        | (Elasticsearch,   |
  | running on each   |         |  Loki, S3, etc.)  |
  | node              |         +------------------+
  +-------------------+
```

### Fluentd vs Fluent Bit

| Feature | Fluentd | Fluent Bit |
|---------|---------|------------|
| Written in | C + Ruby | C only |
| Memory footprint | ~60 MB | ~1 MB |
| Plugin ecosystem | 1000+ plugins | ~100 built-in plugins |
| Best for | Heavy processing, complex routing | Lightweight collection, edge/IoT |
| CNCF Status | Graduated | Graduated (sub-project of Fluentd) |
| Deployment in K8s | Deployment or DaemonSet | DaemonSet (typical) |
| Use case | Aggregation layer | Node-level log forwarder |

### Common Architecture: Fluent Bit + Fluentd

A popular pattern combines both — Fluent Bit collects on each node and forwards to a central Fluentd for processing:

```
  Node 1                Node 2                Node 3
  +-------------+       +-------------+       +-------------+
  | Fluent Bit  |       | Fluent Bit  |       | Fluent Bit  |
  | (DaemonSet) |       | (DaemonSet) |       | (DaemonSet) |
  +------+------+       +------+------+       +------+------+
         |                     |                     |
         +----------+----------+----------+----------+
                    |                     |
                    v                     v
              +------------------------------+
              |          Fluentd             |
              |   (Aggregation layer)        |
              |   - Parse, filter, enrich    |
              +-------------+----------------+
                            |
                +-----------+-----------+
                |                       |
                v                       v
         +-----------+          +------------+
         |Elasticsearch|        |  S3 / GCS  |
         +-----------+          +------------+
```

### Log Pipeline Concepts

Both tools use a pipeline model:

```
  INPUT  --->  PARSER  --->  FILTER  --->  OUTPUT

  Collect       Parse         Transform     Send to
  logs from     structured    (add fields,  destination
  files,        format from   remove noise, (Elasticsearch,
  stdout,       raw text      redact PII)   Loki, S3, etc.)
  TCP, etc.
```

---

## Log Stacks — EFK and ELK

These are popular combinations for centralized logging:

### EFK Stack (Kubernetes-native)

```
  E = Elasticsearch  (store & index logs)
  F = Fluentd        (collect & forward logs)
  K = Kibana         (visualize & search logs)

  +-------------+       +---------------+       +----------+
  | Fluentd /   |  -->  | Elasticsearch |  -->  | Kibana   |
  | Fluent Bit  |       |               |       |          |
  | (collect)   |       | (store/index) |       | (search/ |
  +-------------+       +---------------+       | visualize)|
                                                  +----------+
```

### ELK Stack (traditional)

```
  E = Elasticsearch  (store & index logs)
  L = Logstash       (collect & process logs)
  K = Kibana         (visualize & search logs)
```

The difference is **Fluentd vs Logstash** as the collection layer. In Kubernetes environments, **EFK is more common** because Fluentd/Fluent Bit is CNCF-native and lighter weight.

### Alternative: Grafana Loki

A newer, lightweight alternative to Elasticsearch for log aggregation:

- Designed by the Grafana team
- Does not index log content (only labels) — much cheaper to run
- Uses the same label model as Prometheus
- Queries use LogQL (similar to PromQL)
- Integrates natively with Grafana

---

## Distributed Tracing — Jaeger and Zipkin

Distributed tracing follows a request as it flows through multiple services, capturing timing and relationship data.

### Core Concepts

**Trace**: The entire journey of a request across all services (identified by a unique trace ID).

**Span**: A single operation within a trace (e.g., one service processing the request). Each span has:
- Operation name
- Start time and duration
- Parent span ID (to build the tree)
- Tags/labels (metadata)

**Context Propagation**: The mechanism by which trace context (trace ID, span ID) is passed between services, typically via HTTP headers.

```
  Trace: "User places an order" (trace-id: abc-123)

  +-- api-gateway (span 1) ----------------------------------------+
  |   duration: 120ms                                               |
  |                                                                 |
  |   +-- auth-service (span 2) ----+                               |
  |   |   duration: 15ms            |                               |
  |   +-----------------------------+                               |
  |                                                                 |
  |   +-- order-service (span 3) ----------------------------+      |
  |   |   duration: 95ms                                     |      |
  |   |                                                      |      |
  |   |   +-- inventory-check (span 4) ----+                 |      |
  |   |   |   duration: 10ms               |                 |      |
  |   |   +--------------------------------+                 |      |
  |   |                                                      |      |
  |   |   +-- payment-service (span 5) ----+                 |      |
  |   |   |   duration: 70ms               |  <-- slow!      |      |
  |   |   |                                |                 |      |
  |   |   |   +-- db-write (span 6) ---+   |                 |      |
  |   |   |   |   duration: 60ms       |   |  <-- root cause |      |
  |   |   |   +------------------------+   |                 |      |
  |   |   +--------------------------------+                 |      |
  |   +------------------------------------------------------+      |
  +-----------------------------------------------------------------+
```

### How Context Propagation Works

```
  Service A                     Service B                  Service C
  +----------+                  +----------+               +----------+
  |          |  HTTP request    |          |  HTTP request  |          |
  |  Create  |  + headers:      |  Extract |  + headers:    |  Extract |
  |  trace   |  traceparent:   |  context |  traceparent:  |  context |
  |  context |  abc-123/span1  |  Create  |  abc-123/span2 |  Create  |
  |          | ---------------> |  child   | --------------> |  child   |
  |          |                  |  span    |                |  span    |
  +----------+                  +----------+               +----------+

  All spans share the same trace-id (abc-123),
  forming a tree when reassembled.
```

### Jaeger

- Originally built by Uber, donated to CNCF
- **CNCF Graduated** project
- Full-featured distributed tracing platform
- Supports OpenTelemetry natively
- Components: Agent, Collector, Query UI, Storage (Elasticsearch/Cassandra)
- Provides dependency graphs showing service relationships

### Zipkin

- Originally built by Twitter (inspired by Google's Dapper paper)
- Open-source (not a CNCF project)
- Simpler architecture — single binary deployment
- Supports many languages and frameworks
- Storage: in-memory, MySQL, Cassandra, Elasticsearch

### Jaeger vs Zipkin

| Feature | Jaeger | Zipkin |
|---------|--------|--------|
| Origin | Uber | Twitter |
| CNCF Status | Graduated | Not a CNCF project |
| Architecture | Distributed (agent + collector + query) | Single binary or distributed |
| UI | Feature-rich | Simpler |
| OpenTelemetry | Native support | Supported via exporters |
| Storage backends | Elasticsearch, Cassandra, Kafka | In-memory, MySQL, Cassandra, Elasticsearch |
| Best for | Production-scale K8s environments | Simpler setups |

---

## Tool Comparison Table

| Tool | Category | CNCF Status | Primary Function |
|------|----------|-------------|------------------|
| **Prometheus** | Metrics | Graduated | Scrape, store, and query metrics |
| **Grafana** | Visualization | Graduated | Dashboards for metrics, logs, traces |
| **Alertmanager** | Alerting | Part of Prometheus | Route and manage alerts |
| **Fluentd** | Logging | Graduated | Log collection, processing, forwarding |
| **Fluent Bit** | Logging | Graduated | Lightweight log collection/forwarding |
| **Elasticsearch** | Log Storage | Not CNCF | Index and search log data |
| **Kibana** | Log Visualization | Not CNCF | Search and visualize Elasticsearch data |
| **Loki** | Log Storage | Not CNCF (Grafana Labs) | Lightweight log aggregation |
| **Jaeger** | Tracing | Graduated | Distributed tracing backend and UI |
| **Zipkin** | Tracing | Not CNCF | Distributed tracing backend and UI |
| **OpenTelemetry** | Collection | Incubating | Vendor-neutral telemetry collection |

### Common Tool Combinations

```
  METRICS STACK           LOGGING STACK           TRACING STACK
  +-------------+        +-------------+         +-------------+
  | Prometheus  |        | Fluent Bit  |         | OpenTelemetry|
  | (collect)   |        | (collect)   |         | SDK (collect)|
  +------+------+        +------+------+         +------+------+
         |                      |                        |
         v                      v                        v
  +-------------+        +-------------+         +-------------+
  | Prometheus  |        |Elasticsearch|         | Jaeger      |
  | TSDB (store)|        | (store)     |         | (store)     |
  +------+------+        +------+------+         +------+------+
         |                      |                        |
         v                      v                        v
  +-------------+        +-------------+         +-------------+
  | Grafana     |        | Kibana or   |         | Jaeger UI   |
  | (visualize) |        | Grafana     |         | or Grafana  |
  +-------------+        +-------------+         +-------------+
```

---

## What to Remember for the Exam

1. **Grafana** is a visualization layer — it does not collect or store data. It queries data sources like Prometheus, Elasticsearch, Loki, and Jaeger to build dashboards. CNCF Graduated.

2. **Fluentd vs Fluent Bit**: Both collect and forward logs. Fluentd is full-featured (~60 MB, 1000+ plugins). Fluent Bit is lightweight (~1 MB, written in C). Both are CNCF Graduated. In Kubernetes, Fluent Bit runs as a DaemonSet on every node.

3. **EFK stack**: Elasticsearch (store) + Fluentd/Fluent Bit (collect) + Kibana (visualize). This is the Kubernetes-native logging stack. ELK uses Logstash instead of Fluentd.

4. **Distributed tracing**: A trace tracks a request across services. A trace is composed of spans. Context propagation passes trace IDs between services via HTTP headers.

5. **Jaeger** is CNCF Graduated; Zipkin is not a CNCF project. Both serve the same purpose (distributed tracing).

6. **Spans and traces**: A span represents one operation in one service. A trace is the collection of all spans for one end-to-end request, forming a tree structure.

7. **Grafana Loki** is a lightweight alternative to Elasticsearch — it indexes labels only (not log content), making it cheaper to run.

8. **Context propagation** is how trace context moves between services — typically via HTTP headers (e.g., `traceparent` in W3C Trace Context standard).
