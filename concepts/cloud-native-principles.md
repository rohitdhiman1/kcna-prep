# Cloud Native Principles

> The foundational philosophy behind cloud native: build applications that fully exploit the advantages of the cloud computing model.

---

## What Is Cloud Native?

The **Cloud Native Computing Foundation (CNCF)** defines cloud native as:

> "Cloud native technologies empower organizations to build and run scalable applications in modern, dynamic environments such as public, private, and hybrid clouds. Containers, service meshes, microservices, immutable infrastructure, and declarative APIs exemplify this approach."

In short, cloud native is not just "running in the cloud." It is a set of **practices, patterns, and technologies** designed to take full advantage of distributed, scalable infrastructure.

---

## Key Characteristics

Cloud native applications share four core characteristics:

```
  ┌──────────────────────────────────────────────────┐
  │           Cloud Native Characteristics            │
  │                                                    │
  │   ┌────────────┐         ┌────────────┐           │
  │   │  Scalable  │         │  Resilient │           │
  │   │            │         │            │           │
  │   │ Scale out/ │         │ Self-heal, │           │
  │   │ in on      │         │ tolerate   │           │
  │   │ demand     │         │ failures   │           │
  │   └────────────┘         └────────────┘           │
  │                                                    │
  │   ┌────────────┐         ┌─────────────┐          │
  │   │ Manageable │         │ Observable  │          │
  │   │            │         │             │          │
  │   │ Declarative│         │ Metrics,    │          │
  │   │ config,    │         │ logs,       │          │
  │   │ automation │         │ traces      │          │
  │   └────────────┘         └─────────────┘          │
  │                                                    │
  └──────────────────────────────────────────────────┘
```

### 1. Scalable
- Applications can **scale horizontally** (add more instances) rather than just vertically (bigger machines).
- Scaling is elastic: resources grow and shrink with demand.

### 2. Resilient
- Applications are designed to **tolerate failures** gracefully.
- Self-healing mechanisms detect and recover from failures automatically.
- No single point of failure.

### 3. Manageable
- Infrastructure and applications are managed through **declarative configuration** (YAML, not manual steps).
- Automation handles provisioning, deployments, and operations.
- **Immutable infrastructure**: replace, don't patch.

### 4. Observable
- Applications expose **metrics, logs, and traces** so operators can understand system state.
- Proactive alerting before users are affected.

---

## Cloud Native vs Traditional Architecture

```
  Traditional Architecture              Cloud Native Architecture
  ========================              =========================

  ┌───────────────────────┐            ┌─────┐ ┌─────┐ ┌─────┐
  │                       │            │ Svc │ │ Svc │ │ Svc │
  │   Monolithic App      │            │  A  │ │  B  │ │  C  │
  │                       │            └──┬──┘ └──┬──┘ └──┬──┘
  │  ┌─────┐ ┌─────┐     │               │       │       │
  │  │ UI  │ │Logic│     │            ┌──┴───────┴───────┴──┐
  │  └─────┘ └─────┘     │            │   Service Mesh /    │
  │  ┌─────┐ ┌─────┐     │            │   API Gateway       │
  │  │ DB  │ │Auth │     │            └──┬───────┬───────┬──┘
  │  └─────┘ └─────┘     │               │       │       │
  │                       │            ┌──┴──┐ ┌──┴──┐ ┌──┴──┐
  └───────────┬───────────┘            │DB A │ │DB B │ │DB C │
              │                        └─────┘ └─────┘ └─────┘
     ┌────────┴────────┐
     │  Single Server   │             Containers on Kubernetes
     │  (scale UP only) │             (scale OUT, self-heal)
     └─────────────────┘

  - Tightly coupled               - Loosely coupled microservices
  - Scale vertically               - Scale horizontally
  - Manual deployments             - CI/CD automated pipelines
  - Long release cycles            - Frequent, small releases
  - Mutable servers (patch)        - Immutable containers (replace)
  - Ops team manages servers       - Platform manages infra
  - Failures cascade               - Failures are isolated
```

| Aspect | Traditional | Cloud Native |
|--------|-------------|--------------|
| Deployment unit | WAR/JAR on app server | Container image |
| Scaling | Vertical (bigger server) | Horizontal (more replicas) |
| State | Stateful servers | Stateless services + external state |
| Updates | Big-bang releases | Rolling / canary / blue-green |
| Infrastructure | Mutable (patched in-place) | Immutable (replaced) |
| Configuration | Embedded in app | Externalized (env vars, ConfigMaps) |
| Recovery | Manual intervention | Self-healing (orchestrator restarts) |

---

## The 12-Factor App Methodology

The **12-Factor App** is a set of best practices for building modern, cloud native applications. Originally published by Heroku engineers, these principles are highly relevant to the KCNA exam.

### All 12 Factors

| # | Factor | Description |
|---|--------|-------------|
| 1 | **Codebase** | One codebase tracked in version control, many deploys (dev, staging, prod). |
| 2 | **Dependencies** | Explicitly declare and isolate dependencies (e.g., `package.json`, `requirements.txt`). Never rely on system-wide packages. |
| 3 | **Config** | Store config in the environment (env vars), not in code. Kubernetes uses ConfigMaps and Secrets for this. |
| 4 | **Backing Services** | Treat databases, message queues, caches as **attached resources** accessed via URL/config. Swappable without code changes. |
| 5 | **Build, Release, Run** | Strictly separate build (compile), release (combine build + config), and run (execute) stages. Container images embody the build; Kubernetes manifests add config. |
| 6 | **Processes** | Run the app as one or more **stateless** processes. Store shared state in external backing services (DB, Redis). |
| 7 | **Port Binding** | Export services via port binding. The app is self-contained and does not require an external web server. Containers expose ports; Kubernetes Services route traffic. |
| 8 | **Concurrency** | Scale out via the **process model** — run multiple instances rather than making one instance bigger. This is exactly what Kubernetes Deployments do with replicas. |
| 9 | **Disposability** | Processes start fast and shut down gracefully. Containers should handle SIGTERM. Enables rapid scaling and deployment. |
| 10 | **Dev/Prod Parity** | Keep development, staging, and production as **similar as possible**. Containers help achieve this — same image runs everywhere. |
| 11 | **Logs** | Treat logs as **event streams**. Write to stdout/stderr; let the platform (Fluentd, Fluent Bit) collect and route them. |
| 12 | **Admin Processes** | Run one-off admin tasks (migrations, scripts) as processes in the **same environment** as the app. In Kubernetes, use Jobs or exec into pods. |

### How 12-Factor Maps to Kubernetes

```
  12-Factor Principle          Kubernetes Concept
  ─────────────────────        ──────────────────
  Codebase                 --> Git repo + container registry
  Dependencies             --> Container image (all deps baked in)
  Config                   --> ConfigMaps, Secrets, env vars
  Backing Services         --> ExternalName Services, connection strings
  Build, Release, Run      --> CI/CD pipeline + image tags + Deployment
  Processes                --> Pods (stateless, ephemeral)
  Port Binding             --> containerPort + Service
  Concurrency              --> replicas in Deployment / HPA
  Disposability            --> Pod lifecycle, graceful shutdown
  Dev/Prod Parity          --> Same image across namespaces/clusters
  Logs                     --> stdout/stderr + Fluentd/Fluent Bit
  Admin Processes          --> Jobs, CronJobs, kubectl exec
```

---

## Benefits of Cloud Native

### Portability
- Containers run the same way on any cloud or on-premises environment.
- No vendor lock-in when using open standards (OCI, CRI, CNI, CSI).

### Elasticity
- Applications scale automatically based on demand.
- Pay only for what you use (especially with serverless and autoscaling).

### Agility
- Small, independent teams can deploy independently.
- Microservices allow different technology choices per service.
- Faster experimentation and iteration.

### Faster Time to Market
- CI/CD pipelines automate build, test, and deploy.
- Small, frequent releases reduce risk.
- Infrastructure provisioning is automated and fast.

---

## Design for Failure

A core cloud native principle: **assume everything will fail, and design accordingly**.

```
  ┌─────────────────────────────────────────────────┐
  │           Design for Failure Patterns            │
  │                                                   │
  │  ┌──────────────┐    ┌───────────────────┐       │
  │  │  Redundancy  │    │  Health Checks    │       │
  │  │  Run multiple│    │  Liveness probes  │       │
  │  │  replicas    │    │  Readiness probes │       │
  │  └──────────────┘    └───────────────────┘       │
  │                                                   │
  │  ┌──────────────┐    ┌───────────────────┐       │
  │  │  Circuit     │    │  Graceful         │       │
  │  │  Breakers    │    │  Degradation      │       │
  │  │  Stop calling│    │  Serve partial    │       │
  │  │  failing svc │    │  results          │       │
  │  └──────────────┘    └───────────────────┘       │
  │                                                   │
  │  ┌──────────────┐    ┌───────────────────┐       │
  │  │  Retries +   │    │  Chaos            │       │
  │  │  Timeouts    │    │  Engineering      │       │
  │  │  Backoff     │    │  Test failure     │       │
  │  │  strategies  │    │  proactively      │       │
  │  └──────────────┘    └───────────────────┘       │
  │                                                   │
  └─────────────────────────────────────────────────┘
```

Key patterns:
- **Redundancy**: Run multiple replicas across nodes and zones.
- **Health checks**: Kubernetes liveness and readiness probes detect and replace unhealthy pods.
- **Circuit breakers**: Stop sending requests to a failing service; let it recover.
- **Graceful degradation**: Return partial results instead of a full error.
- **Retries with backoff**: Retry failed requests with increasing delays.
- **Chaos engineering**: Intentionally inject failures to test resilience (e.g., Chaos Monkey, Litmus).

---

## What to Remember for the Exam

1. **Cloud native is NOT just "running in the cloud"** — it is a set of practices and patterns (containers, microservices, CI/CD, declarative config).
2. The four key characteristics: **Scalable, Resilient, Manageable, Observable**.
3. **12-Factor App** is the methodology for building cloud native apps. Know all 12 factors at a high level, especially Config (factor 3), Processes (factor 6), Disposability (factor 9), and Logs (factor 11).
4. **Immutable infrastructure**: replace, don't patch. Containers embody this.
5. **Design for failure**: assume components will fail; use redundancy, health checks, circuit breakers.
6. Cloud native benefits: **portability, elasticity, agility, faster time to market**.
7. The CNCF is the governing body for cloud native projects and defines the cloud native approach.
