# Serverless Computing

> Run code without managing servers — the platform handles provisioning, scaling, and infrastructure automatically.

---

## What Is Serverless?

Serverless computing is a cloud execution model where:

- The **cloud provider** (or platform) manages all infrastructure.
- You deploy **functions or applications** without provisioning or managing servers.
- Resources are allocated **on demand** and released when not in use.
- You are billed based on **actual usage** (pay-per-invocation or pay-per-request), not reserved capacity.

Despite the name, **servers still exist** — you just do not manage them.

---

## Key Characteristics

```
  ┌──────────────────────────────────────────────────────┐
  │              Serverless Characteristics                │
  │                                                        │
  │  ┌────────────────────┐   ┌────────────────────┐      │
  │  │  No Server Mgmt   │   │   Event-Driven     │      │
  │  │                    │   │                    │      │
  │  │  No provisioning,  │   │  Functions run in  │      │
  │  │  patching, or      │   │  response to       │      │
  │  │  capacity planning │   │  events/triggers   │      │
  │  └────────────────────┘   └────────────────────┘      │
  │                                                        │
  │  ┌────────────────────┐   ┌────────────────────┐      │
  │  │  Pay-Per-Use       │   │  Auto-Scale to     │      │
  │  │                    │   │  Zero              │      │
  │  │  Billed only for   │   │                    │      │
  │  │  actual execution  │   │  No traffic = no   │      │
  │  │  time / requests   │   │  running instances │      │
  │  └────────────────────┘   └────────────────────┘      │
  │                                                        │
  └──────────────────────────────────────────────────────┘
```

### 1. No Server Management
- No need to provision, patch, or maintain servers.
- The platform handles OS updates, security patches, and hardware.

### 2. Event-Driven
- Functions execute in response to triggers: HTTP requests, database changes, file uploads, messages in a queue, scheduled timers.
- When there are no events, nothing runs.

### 3. Pay-Per-Use
- Charged per invocation, per request, or per execution duration.
- No charges when idle (unlike VMs or always-running containers).

### 4. Auto-Scaling to Zero
- Automatically scales up when requests arrive.
- Scales back to **zero instances** when there is no traffic.
- This is the key differentiator from traditional container deployments.

---

## Function as a Service (FaaS)

FaaS is the most common serverless model. You write a function; the platform runs it.

```
  ┌──────────────────────────────────────────────────────────┐
  │                  FaaS Execution Flow                      │
  │                                                            │
  │  Event                Function              Result         │
  │  ┌─────────┐         ┌──────────┐         ┌──────────┐   │
  │  │ HTTP    │────────►│          │────────►│ Response │   │
  │  │ Request │         │ Your     │         │ / Side   │   │
  │  └─────────┘         │ Function │         │ Effect   │   │
  │  ┌─────────┐         │          │         └──────────┘   │
  │  │ Queue   │────────►│ (Node.js,│                        │
  │  │ Message │         │  Python, │                        │
  │  └─────────┘         │  Go,     │                        │
  │  ┌─────────┐         │  Java)   │                        │
  │  │ DB      │────────►│          │                        │
  │  │ Change  │         └──────────┘                        │
  │  └─────────┘              ▲                              │
  │                           │                              │
  │              Platform provisions container,              │
  │              runs function, then can remove it           │
  │                                                            │
  └──────────────────────────────────────────────────────────┘
```

### Popular FaaS Platforms

| Platform | Provider |
|----------|----------|
| AWS Lambda | Amazon |
| Azure Functions | Microsoft |
| Google Cloud Functions | Google |
| IBM Cloud Functions | IBM (Apache OpenWhisk) |
| Knative | Open source (Kubernetes) |
| OpenFaaS | Open source (Kubernetes) |

---

## Serverless Architecture

```
  ┌────────────────────────────────────────────────────────────┐
  │              Serverless Application Architecture            │
  │                                                              │
  │   Client                                                     │
  │   ┌────────┐                                                 │
  │   │Browser/│                                                 │
  │   │Mobile  │                                                 │
  │   └───┬────┘                                                 │
  │       │                                                      │
  │       ▼                                                      │
  │   ┌────────────┐                                             │
  │   │API Gateway │  <-- Routes requests to functions           │
  │   └──┬────┬──┬─┘                                             │
  │      │    │  │                                               │
  │      ▼    │  ▼                                               │
  │  ┌──────┐ │ ┌──────┐                                        │
  │  │Func A│ │ │Func C│   <-- Stateless functions              │
  │  │(auth)│ │ │(pay) │       Scale independently              │
  │  └──┬───┘ │ └──┬───┘       Scale to zero when idle          │
  │     │     │    │                                             │
  │     │     ▼    │                                             │
  │     │  ┌──────┐│                                             │
  │     │  │Func B││                                             │
  │     │  │(order││                                             │
  │     │  └──┬───┘│                                             │
  │     │     │    │                                             │
  │     ▼     ▼    ▼                                             │
  │   ┌─────┐ ┌─────┐ ┌──────────┐                              │
  │   │Auth │ │Order│ │ Payment  │  <-- Managed backing         │
  │   │ DB  │ │ DB  │ │ Service  │      services (also          │
  │   └─────┘ └─────┘ └──────────┘      serverless/managed)     │
  │                                                              │
  │   ┌──────────┐    ┌──────────┐                               │
  │   │ Event    │───►│ Func D   │  <-- Event-driven            │
  │   │ Stream   │    │(process) │      processing              │
  │   │ (Kafka)  │    └──────────┘                               │
  │   └──────────┘                                               │
  │                                                              │
  └────────────────────────────────────────────────────────────┘
```

---

## Knative

**Knative** is an open-source platform that runs serverless workloads on **Kubernetes**. It is a CNCF Incubating project.

Knative has two main components:

### Knative Serving

Provides request-driven compute that can scale to zero.

```
  Request arrives           Knative Serving                  Scale to zero
  ──────────────           ───────────────                  ──────────────

  ┌────────┐    ┌──────────────┐    ┌───────────┐
  │  HTTP  │───►│  Activator   │───►│  Pod(s)   │         No traffic
  │Request │    │              │    │           │         for configured
  └────────┘    │ If 0 pods,   │    │  Your app │         period
                │ buffers req  │    │  running  │              │
                │ and triggers │    └───────────┘              ▼
                │ scale-up     │                         ┌───────────┐
                └──────────────┘                         │  0 pods   │
                                                         │  (idle)   │
                                                         └───────────┘
```

Key features:
- **Scale to zero**: Pods are removed when there is no traffic.
- **Scale from zero**: The Activator buffers incoming requests while pods start up.
- **Revision management**: Each deployment creates a revision; traffic can be split across revisions (canary, blue-green).
- **Automatic TLS**: Can auto-provision HTTPS certificates.

### Knative Eventing

Provides event-driven architecture for Kubernetes.

```
  ┌──────────┐    ┌─────────────┐    ┌──────────────┐    ┌──────────┐
  │  Event   │───►│   Broker    │───►│   Trigger    │───►│  Service │
  │  Source  │    │             │    │  (filter)    │    │  (Sink)  │
  │          │    │  Receives & │    │              │    │          │
  │ - GitHub │    │  distributes│    │  Routes by   │    │  Your    │
  │ - Kafka  │    │  events     │    │  event type  │    │  function │
  │ - Cron   │    │             │    │              │    │          │
  └──────────┘    └─────────────┘    └──────────────┘    └──────────┘
```

Key features:
- **Event sources**: GitHub, Kafka, Cron, API server, custom.
- **Broker and Trigger model**: Broker receives events; Triggers filter and route them to services.
- **CloudEvents**: Uses the CloudEvents specification for event format interoperability.
- **Loose coupling**: Event producers do not know about consumers.

---

## Use Cases for Serverless

| Use Case | Example |
|----------|---------|
| **API backends** | REST APIs with variable traffic |
| **Event processing** | Process file uploads, database changes |
| **Scheduled tasks** | Periodic data cleanup, report generation |
| **Webhooks** | GitHub webhooks, Slack bots |
| **Data pipelines** | ETL processing triggered by new data |
| **IoT backends** | Process sensor data from devices |

### When Serverless Is NOT Ideal

- **Long-running processes**: Functions have execution time limits (e.g., 15 min on AWS Lambda).
- **Low-latency requirements**: Cold starts can add latency (100ms to several seconds).
- **Stateful applications**: Functions are stateless; state must be externalized.
- **High-throughput constant load**: If traffic is constant, always-running containers may be cheaper.

---

## Serverless vs Containers vs VMs

```
  ┌────────────────────────────────────────────────────────────────┐
  │                   Abstraction Levels                            │
  │                                                                  │
  │   More control                              Less control        │
  │   More management                           Less management     │
  │   ◄──────────────────────────────────────────────────────►      │
  │                                                                  │
  │   ┌──────────┐      ┌──────────┐      ┌──────────────┐         │
  │   │   VMs    │      │Containers│      │  Serverless  │         │
  │   │          │      │          │      │              │         │
  │   │ You      │      │ You      │      │ You manage   │         │
  │   │ manage:  │      │ manage:  │      │ only:        │         │
  │   │ - OS     │      │ - App    │      │ - Code       │         │
  │   │ - Runtime│      │ - Runtime│      │              │         │
  │   │ - App    │      │ - Deps   │      │ Platform     │         │
  │   │ - Deps   │      │          │      │ manages      │         │
  │   │ - Scaling│      │ Platform │      │ everything   │         │
  │   │          │      │ manages: │      │ else         │         │
  │   │          │      │ - OS     │      │              │         │
  │   │          │      │ - Scaling│      │              │         │
  │   └──────────┘      └──────────┘      └──────────────┘         │
  │                                                                  │
  └────────────────────────────────────────────────────────────────┘
```

| Aspect | VMs | Containers | Serverless |
|--------|-----|------------|------------|
| **Startup time** | Minutes | Seconds | Milliseconds* |
| **Scaling** | Manual or slow auto | Fast auto (HPA) | Instant, to zero |
| **Billing** | Per hour/reserved | Per second/resource | Per invocation |
| **State** | Stateful | Can be stateful | Stateless |
| **Isolation** | Hardware-level | OS-level (namespaces) | Function-level |
| **Ops overhead** | High | Medium | Low |
| **Vendor lock-in** | Medium | Low (OCI standard) | High (unless Knative) |
| **Max execution** | Unlimited | Unlimited | Time-limited |

*After warm-up; cold starts can take seconds.

---

## Cold Starts

A **cold start** happens when a serverless function is invoked after being idle (scaled to zero). The platform must:

1. Provision a container/runtime.
2. Load the function code.
3. Initialize dependencies.
4. Execute the function.

This adds latency (100ms to several seconds depending on runtime, package size, and provider).

**Mitigation strategies:**
- Keep function packages small.
- Use lightweight runtimes (Go, Rust over Java, .NET).
- Use provisioned concurrency (cloud provider feature to keep instances warm).
- Knative's Activator buffers requests during cold start.

---

## What to Remember for the Exam

1. **Serverless** = no server management, event-driven, pay-per-use, auto-scale to zero.
2. **FaaS (Function as a Service)** is the primary serverless model — deploy functions, not applications.
3. **Knative** is the open-source serverless platform for Kubernetes (CNCF Incubating project). Two components:
   - **Serving**: request-driven compute, scale to/from zero, revision management.
   - **Eventing**: event-driven architecture, Broker/Trigger model, CloudEvents.
4. Serverless is best for: variable traffic, event processing, APIs, webhooks.
5. Serverless is NOT ideal for: long-running tasks, low-latency requirements, constant high throughput.
6. **Cold starts** are a key trade-off — idle functions take time to start.
7. Know the progression: **VMs → Containers → Serverless** (increasing abstraction, decreasing ops overhead).
8. Knative runs **on top of Kubernetes** — it is not a replacement for Kubernetes.
9. Serverless functions are **stateless** by design; state must be stored externally.
