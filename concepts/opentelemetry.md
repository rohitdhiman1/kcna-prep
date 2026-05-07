# OpenTelemetry (OTel)

**OpenTelemetry is a vendor-neutral, open-source framework for generating, collecting, and exporting telemetry data (metrics, logs, and traces) — it is the CNCF standard for instrumenting cloud native applications and was formed by merging OpenTracing and OpenCensus.**

---

## Table of Contents

- [What is OpenTelemetry](#what-is-opentelemetry)
- [History — OpenTracing + OpenCensus](#history--opentracing--opencensus)
- [CNCF Status](#cncf-status)
- [Core Components](#core-components)
- [The OTel Collector](#the-otel-collector)
- [OTLP — OpenTelemetry Protocol](#otlp--opentelemetry-protocol)
- [Integration with Other Tools](#integration-with-other-tools)
- [OTel in Kubernetes](#otel-in-kubernetes)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## What is OpenTelemetry

OpenTelemetry (OTel) provides a **single, unified standard** for collecting all three types of telemetry data:

```
  Without OpenTelemetry                 With OpenTelemetry
  +----------+                          +----------+
  | App Code |                          | App Code |
  +----+-----+                          +----+-----+
       |                                     |
       +--- Prometheus client lib            +--- OTel SDK (one library)
       +--- Jaeger client lib                     |
       +--- Fluentd log lib                       | generates all three:
       +--- Vendor X SDK                          | - metrics
       |                                          | - logs
   4 libraries,                                   | - traces
   4 formats,                                     |
   4 things to maintain                      1 library,
                                              1 standard format,
                                              vendor-neutral
```

**Key idea**: OpenTelemetry separates **instrumentation** (generating telemetry in your code) from **backend** (where telemetry is stored and analyzed). You instrument once with OTel, then send data to any backend.

---

## History — OpenTracing + OpenCensus

OpenTelemetry was created in 2019 by merging two earlier CNCF projects:

```
  +------------------+       +------------------+
  | OpenTracing      |       | OpenCensus       |
  |                  |       |                  |
  | - Tracing only   |       | - Traces + Metrics|
  | - API standard   |       | - API + SDK + Agent|
  | - No SDK/agent   |       | - Google-originated|
  | - CNCF project   |       | - CNCF project   |
  +--------+---------+       +--------+---------+
           |                          |
           +----------+  +-----------+
                      |  |
                      v  v
              +------------------+
              | OpenTelemetry    |
              |                  |
              | - Metrics + Logs |
              |   + Traces       |
              | - API + SDK +    |
              |   Collector      |
              | - Vendor-neutral |
              | - CNCF Incubating|
              +------------------+
                 (2019 - present)
```

Both predecessor projects are now **archived** — OpenTelemetry is the successor and the future standard.

---

## CNCF Status

- **CNCF Incubating** project
- One of the most active CNCF projects by contributor count
- Supported by major vendors: Google, Microsoft, AWS, Datadog, Splunk, New Relic, and many others
- Language SDKs available for: Go, Java, Python, JavaScript/Node.js, .NET, Rust, C++, and more

---

## Core Components

OpenTelemetry consists of several key components:

```
+===========================================================================+
|                      OPENTELEMETRY COMPONENTS                             |
|                                                                           |
|  +-------------------------------------------------------------------+   |
|  |  1. SPECIFICATION                                                  |   |
|  |     - Defines the API and SDK interfaces                           |   |
|  |     - Language-agnostic contracts                                  |   |
|  +-------------------------------------------------------------------+   |
|                                                                           |
|  +-------------------------------------------------------------------+   |
|  |  2. API                                                            |   |
|  |     - Interfaces for instrumenting code                            |   |
|  |     - TracerProvider, MeterProvider, LoggerProvider                 |   |
|  |     - No-op by default (safe to ship in libraries)                 |   |
|  +-------------------------------------------------------------------+   |
|                                                                           |
|  +-------------------------------------------------------------------+   |
|  |  3. SDK                                                            |   |
|  |     - Implementation of the API                                    |   |
|  |     - Sampling, batching, export configuration                     |   |
|  |     - One SDK per language (Go, Java, Python, etc.)                |   |
|  +-------------------------------------------------------------------+   |
|                                                                           |
|  +-------------------------------------------------------------------+   |
|  |  4. COLLECTOR                                                      |   |
|  |     - Standalone binary for receiving, processing, exporting data  |   |
|  |     - Vendor-agnostic pipeline                                     |   |
|  |     - Can replace multiple vendor agents                           |   |
|  +-------------------------------------------------------------------+   |
|                                                                           |
|  +-------------------------------------------------------------------+   |
|  |  5. EXPORTERS                                                      |   |
|  |     - Plugins that send data to specific backends                  |   |
|  |     - Prometheus, Jaeger, Zipkin, OTLP, vendor-specific            |   |
|  +-------------------------------------------------------------------+   |
|                                                                           |
|  +-------------------------------------------------------------------+   |
|  |  6. AUTO-INSTRUMENTATION                                          |   |
|  |     - Automatic instrumentation for popular frameworks             |   |
|  |     - Zero-code-change telemetry for HTTP, gRPC, DB calls          |   |
|  +-------------------------------------------------------------------+   |
+===========================================================================+
```

---

## The OTel Collector

The Collector is the central component of an OpenTelemetry deployment. It is a **vendor-agnostic proxy** that receives, processes, and exports telemetry data.

### Collector Pipeline Architecture

```
+=======================================================================+
|                       OTEL COLLECTOR                                   |
|                                                                        |
|  RECEIVERS           PROCESSORS           EXPORTERS                    |
|  (input)             (transform)          (output)                     |
|                                                                        |
|  +-----------+       +-----------+       +-------------+               |
|  | OTLP      |       | Batch     |       | Prometheus  |               |
|  | (gRPC/HTTP)|---+   | (group    |   +-->| (remote     |               |
|  +-----------+   |   | before    |   |   | write)      |               |
|                  |   | sending)  |   |   +-------------+               |
|  +-----------+   |   +-----------+   |                                 |
|  | Prometheus|   |                   |   +-------------+               |
|  | (scrape   |---+--->  Pipeline  ---+--->| Jaeger      |               |
|  | /metrics) |   |                   |   | (traces)    |               |
|  +-----------+   |   +-----------+   |   +-------------+               |
|                  |   | Filter    |   |                                 |
|  +-----------+   |   | (drop     |   |   +-------------+               |
|  | Jaeger    |   |   | unwanted  |   +-->| OTLP        |               |
|  | (traces)  |---+   | data)     |   |   | (to another |               |
|  +-----------+   |   +-----------+   |   | collector)  |               |
|                  |                   |   +-------------+               |
|  +-----------+   |   +-----------+   |                                 |
|  | Fluent    |   |   | Attributes|   |   +-------------+               |
|  | Forward   |---+   | (add/     |   +-->| Elasticsearch|              |
|  | (logs)    |       | remove    |       | (logs)      |               |
|  +-----------+       | labels)   |       +-------------+               |
|                      +-----------+                                     |
|                                                                        |
|   Receive data        Transform data        Send data to               |
|   from any source     (enrich, filter,       any backend               |
|                        sample, batch)                                  |
+=======================================================================+
```

### Receiver → Processor → Exporter

| Stage | Purpose | Examples |
|-------|---------|----------|
| **Receivers** | Accept data from various sources | OTLP, Prometheus, Jaeger, Zipkin, Fluent Forward |
| **Processors** | Transform data in the pipeline | Batch, filter, attributes, sampling, memory limiter |
| **Exporters** | Send data to backends | Prometheus, Jaeger, OTLP, Elasticsearch, Loki, vendor-specific |

### Deployment Modes

The Collector can be deployed in two patterns:

```
  AGENT MODE                              GATEWAY MODE

  Node 1        Node 2                    Node 1        Node 2
  +--------+    +--------+               +--------+    +--------+
  |App  OTel|    |App  OTel|               |App     |    |App     |
  |Pod  Coll|    |Pod  Coll|               |Pod     |    |Pod     |
  +---+----+    +---+----+               +---+----+    +---+----+
      |              |                        |              |
      v              v                        +------+-------+
  +--------+    +--------+                           |
  | Backend|    | Backend|                           v
  +--------+    +--------+                    +------------+
                                              | OTel       |
  Each node has its own                       | Collector  |
  collector (DaemonSet)                       | Gateway    |
                                              +------+-----+
                                                     |
                                                     v
                                              +------------+
                                              | Backend    |
                                              +------------+

                                              Central collector
                                              (Deployment)
```

---

## OTLP — OpenTelemetry Protocol

OTLP (OpenTelemetry Protocol) is the **native wire protocol** for OpenTelemetry:

- Supports **metrics, logs, and traces** in a single protocol
- Available over **gRPC** (default, port 4317) and **HTTP/protobuf** (port 4318)
- Designed to be efficient with binary encoding (protobuf)
- Becoming the industry standard for telemetry transport

```
  Application           Network              Collector/Backend
  +-----------+         (OTLP)              +-----------+
  | OTel SDK  |  ---- gRPC:4317 -------->  | OTel      |
  |           |  or   HTTP:4318             | Collector |
  |           |                             |           |
  | Generates |  protobuf-encoded           | Receives  |
  | metrics,  |  telemetry data             | decodes,  |
  | logs,     |                             | processes |
  | traces    |                             |           |
  +-----------+                             +-----------+
```

---

## Integration with Other Tools

OpenTelemetry is designed to work with existing observability tools, not replace them:

```
+===========================================================================+
|                                                                           |
|  APPLICATION LAYER                                                        |
|  +----------+  +----------+  +----------+                                |
|  | Service A|  | Service B|  | Service C|                                |
|  | (OTel SDK|  | (OTel SDK|  | (OTel SDK|                                |
|  |  traces, |  |  traces, |  |  traces, |                                |
|  |  metrics)|  |  metrics)|  |  metrics)|                                |
|  +----+-----+  +----+-----+  +----+-----+                                |
|       |              |              |                                     |
|       +--------------+--------------+                                     |
|                      |                                                    |
|                      v  (OTLP)                                            |
|               +-------------+                                             |
|               | OTel        |                                             |
|               | Collector   |                                             |
|               +------+------+                                             |
|                      |                                                    |
|         +------------+------------+                                       |
|         |            |            |                                        |
|         v            v            v                                        |
|  +------------+ +----------+ +----------+                                 |
|  | Prometheus | | Jaeger   | | Grafana  |                                 |
|  | (metrics)  | | (traces) | | Loki     |                                 |
|  +------+-----+ +----+-----+ | (logs)   |                                 |
|         |             |       +----+-----+                                 |
|         +-------------+-----------+                                       |
|                       |                                                   |
|                       v                                                   |
|                +------------+                                             |
|                |  Grafana   |                                             |
|                | Dashboards |                                             |
|                +------------+                                             |
+===========================================================================+
```

### Key Integrations

| Backend | How OTel Integrates |
|---------|-------------------|
| **Prometheus** | OTel Collector can scrape Prometheus targets or receive OTLP metrics and remote-write to Prometheus |
| **Jaeger** | OTel Collector exports traces directly to Jaeger; Jaeger natively supports OTLP |
| **Grafana** | Grafana queries backends that OTel feeds; Grafana Tempo accepts OTLP traces directly |
| **Elasticsearch** | OTel Collector exports logs to Elasticsearch |
| **Loki** | OTel Collector exports logs to Grafana Loki |

---

## OTel in Kubernetes

In Kubernetes, the OTel Collector is typically deployed as:

1. **DaemonSet** (agent mode) — one Collector per node for collecting node-level telemetry
2. **Deployment** (gateway mode) — central Collector for aggregation and export
3. **Sidecar** — one Collector per Pod for application-specific processing

Additionally, the **OpenTelemetry Operator** for Kubernetes provides:

- Auto-injection of OTel SDKs into application Pods
- Automatic instrumentation (no code changes needed for supported frameworks)
- Custom Resource Definitions (CRDs) for managing Collectors

```
  Kubernetes Cluster
  +---------------------------------------------------------------+
  |                                                               |
  |  +-------------------+   +-------------------+                |
  |  | Pod: my-app       |   | Pod: my-api       |                |
  |  | +------+ +------+ |   | +------+ +------+ |                |
  |  | |app   | |OTel  | |   | |app   | |OTel  | |                |
  |  | |cntr  | |sidecar| |   | |cntr  | |sidecar| |               |
  |  | +------+ +--+---+ |   | +------+ +--+---+ |                |
  |  +-------------|-----+   +-------------|-----+                |
  |                |                       |                       |
  |                +----------+------------+                       |
  |                           |                                    |
  |                           v                                    |
  |              +------------------------+                        |
  |              | OTel Collector Gateway |                        |
  |              | (Deployment)           |                        |
  |              +----------+-------------+                        |
  |                         |                                      |
  +-------------------------|--------------------------------------+
                            |
               +------------+------------+
               |            |            |
               v            v            v
          Prometheus     Jaeger       Loki
```

---

## What to Remember for the Exam

1. **OpenTelemetry (OTel)** is a vendor-neutral, open-source framework for generating, collecting, and exporting telemetry data — metrics, logs, and traces.

2. **Formed by merging OpenTracing + OpenCensus** in 2019. Both predecessor projects are now archived.

3. **CNCF Incubating** project — one of the most active CNCF projects by contributors.

4. **Core components**: Specification, API, SDK, Collector, Exporters, Auto-instrumentation.

5. **OTel Collector pipeline**: Receivers (input) -> Processors (transform) -> Exporters (output). This is the key architecture to know.

6. **OTLP** (OpenTelemetry Protocol) is the native protocol — supports metrics, logs, and traces over gRPC (port 4317) or HTTP (port 4318).

7. **OTel does not replace backends** — it replaces vendor-specific instrumentation libraries. You still need Prometheus, Jaeger, Grafana, etc. for storage and visualization.

8. **Instrument once, export anywhere**: The main value proposition is that you instrument your code once with the OTel SDK, then configure the Collector to send data to any combination of backends.

9. **Collector deployment modes**: Agent (DaemonSet, one per node) or Gateway (Deployment, central aggregation point).

10. **Supports all three pillars**: Unlike its predecessors, OpenTelemetry covers metrics, logs, AND traces in a single framework.
