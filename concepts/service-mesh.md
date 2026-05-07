# Service Mesh

> A service mesh is a dedicated infrastructure layer for managing service-to-service communication, providing mTLS, observability, and traffic management without changing application code.

---

## Why Service Mesh Exists

In a microservices architecture, services communicate over the network constantly. As the number of services grows, managing this communication becomes complex:

```
  The Problem: Microservice Communication at Scale

  Without Service Mesh                With Service Mesh
  (each service handles everything)   (mesh handles cross-cutting concerns)

  ┌───────┐    ┌───────┐             ┌───────┐    ┌───────┐
  │Svc A  │───►│Svc B  │             │Svc A  │───►│Svc B  │
  │       │    │       │             │(just  │    │(just  │
  │+retry │    │+retry │             │ biz   │    │ biz   │
  │+TLS   │    │+TLS   │             │ logic)│    │ logic)│
  │+metric│    │+metric│             └───┬───┘    └───┬───┘
  │+trace │    │+trace │                 │            │
  │+auth  │    │+auth  │             ┌───┴───┐    ┌───┴───┐
  └───────┘    └───────┘             │Proxy  │───►│Proxy  │
                                     │(sidecar)   │(sidecar)
  Every service must implement       │+retry │    │+retry │
  the same cross-cutting concerns    │+mTLS  │    │+mTLS  │
  in every language                  │+metric│    │+metric│
                                     └───────┘    └───────┘
                                     Mesh handles it uniformly
```

### What a Service Mesh Provides

| Capability | Description |
|-----------|-------------|
| **mTLS (Mutual TLS)** | Automatic encryption and identity verification between services |
| **Traffic management** | Canary deployments, traffic splitting, retries, timeouts |
| **Observability** | Metrics, distributed tracing, and access logs without code changes |
| **Retries and timeouts** | Automatic retry on failure, configurable timeouts |
| **Circuit breaking** | Stop sending traffic to unhealthy services |
| **Load balancing** | Advanced algorithms beyond round-robin |
| **Access control** | Service-to-service authorization policies |

---

## The Sidecar Proxy Pattern

The sidecar pattern is the foundation of most service meshes. A **proxy container** is injected alongside every application container in a pod.

```
  Sidecar Proxy Pattern

  ┌────────────────────── Pod ──────────────────────────┐
  │                                                      │
  │  ┌──────────────────┐    ┌──────────────────────┐   │
  │  │   Application    │    │   Sidecar Proxy      │   │
  │  │   Container      │    │   (e.g., Envoy)      │   │
  │  │                  │    │                      │   │
  │  │   Your code      │◄──►│   Intercepts all     │   │
  │  │   (no mesh       │    │   network traffic    │   │
  │  │    awareness)    │    │                      │   │
  │  │                  │    │   - mTLS             │   │
  │  │   Listens on     │    │   - Metrics          │   │
  │  │   localhost:8080 │    │   - Retries          │   │
  │  │                  │    │   - Load balancing   │   │
  │  └──────────────────┘    └──────────────────────┘   │
  │                                                      │
  └──────────────────────────────────────────────────────┘

  All inbound and outbound traffic flows through the proxy.
  The application is unaware of the proxy.
  iptables rules redirect traffic automatically.
```

The most common sidecar proxy is **Envoy**, originally built by Lyft and now a CNCF Graduated project.

---

## Data Plane vs Control Plane

A service mesh has two distinct layers:

```
  Service Mesh Architecture

  ┌─────────────────────────────────────────────────────────────┐
  │                     CONTROL PLANE                            │
  │                                                              │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │                    istiod / linkerd-control-plane      │   │
  │  │                                                       │   │
  │  │  - Certificate authority (issues mTLS certs)          │   │
  │  │  - Configuration API (traffic rules, policies)        │   │
  │  │  - Service discovery (knows all endpoints)            │   │
  │  │  - Pushes config to all sidecar proxies               │   │
  │  └──────────┬──────────────────┬──────────────┬─────────┘   │
  │             │ config           │ config       │ config      │
  └─────────────┼──────────────────┼──────────────┼─────────────┘
                ▼                  ▼              ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                      DATA PLANE                              │
  │                                                              │
  │  Pod 1              Pod 2              Pod 3                 │
  │  ┌──────┬──────┐   ┌──────┬──────┐   ┌──────┬──────┐      │
  │  │ App  │Proxy │◄─►│ App  │Proxy │◄─►│ App  │Proxy │      │
  │  │      │(Envoy│   │      │(Envoy│   │      │(Envoy│      │
  │  └──────┴──────┘   └──────┴──────┘   └──────┴──────┘      │
  │                                                              │
  │  All service-to-service traffic flows through proxies.       │
  │  Proxies enforce mTLS, collect metrics, apply traffic rules. │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘
```

| Layer | Components | Responsibility |
|-------|-----------|---------------|
| **Control Plane** | istiod (Istio), linkerd-control-plane (Linkerd) | Configuration, certificate management, policy distribution |
| **Data Plane** | Envoy sidecar proxies (or linkerd2-proxy) | Handles actual traffic, enforces rules, collects telemetry |

---

## Istio

Istio is the most feature-rich service mesh, a **CNCF Graduated project**.

### Architecture

```
  Istio Architecture

  ┌──────────────────── Control Plane ─────────────────────┐
  │                                                         │
  │  ┌──────────────────────────────────────────────────┐  │
  │  │                    istiod                         │  │
  │  │                                                   │  │
  │  │  ┌─────────┐  ┌──────────┐  ┌────────────────┐  │  │
  │  │  │  Pilot  │  │ Citadel  │  │    Galley      │  │  │
  │  │  │         │  │          │  │                │  │  │
  │  │  │ Service │  │ Cert     │  │ Config         │  │  │
  │  │  │ disc-   │  │ mgmt,   │  │ validation     │  │  │
  │  │  │ overy,  │  │ mTLS    │  │ and            │  │  │
  │  │  │ traffic │  │ identity│  │ distribution   │  │  │
  │  │  │ mgmt    │  │         │  │                │  │  │
  │  │  └─────────┘  └──────────┘  └────────────────┘  │  │
  │  │                                                   │  │
  │  │  (All consolidated into single istiod binary)     │  │
  │  └──────────────────────────────────────────────────┘  │
  │                                                         │
  └──────────────────────────┬──────────────────────────────┘
                             │ xDS API (config push)
                             ▼
  ┌──────────────────── Data Plane ─────────────────────────┐
  │                                                          │
  │  Pod                    Pod                              │
  │  ┌───────┬─────────┐   ┌───────┬─────────┐             │
  │  │  App  │  Envoy  │◄─►│  App  │  Envoy  │             │
  │  │       │  Proxy  │   │       │  Proxy  │             │
  │  └───────┴─────────┘   └───────┴─────────┘             │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

Key Istio features:
- **istiod** — single control plane binary (consolidates Pilot, Citadel, Galley)
- **Envoy** — the sidecar proxy (data plane)
- **VirtualService** — traffic routing rules
- **DestinationRule** — load balancing, connection pool settings
- **Gateway** — manages inbound/outbound mesh traffic
- **PeerAuthentication** — mTLS settings

---

## Linkerd

Linkerd is a **lightweight, simple** service mesh, also a **CNCF Graduated project**.

| Aspect | Istio | Linkerd |
|--------|-------|---------|
| **Complexity** | Feature-rich, complex | Simple, minimal |
| **Proxy** | Envoy (C++) | linkerd2-proxy (Rust, purpose-built) |
| **Resource usage** | Higher | Lower (smaller proxy) |
| **Learning curve** | Steep | Gentle |
| **mTLS** | Yes (configurable) | Yes (on by default) |
| **Traffic management** | Advanced (VirtualService, etc.) | Basic (TrafficSplit, etc.) |
| **Multi-cluster** | Yes | Yes |
| **CNCF status** | Graduated | Graduated |

Linkerd is often recommended for teams that want mesh benefits without the complexity of Istio.

---

## Service Mesh Interface (SMI)

SMI is a **standard API specification** for service meshes on Kubernetes. It defines common APIs so that tools can work with any mesh implementation.

```
  Service Mesh Interface (SMI)

  ┌────────────────────────────────────────────┐
  │                   SMI API                   │
  │                                             │
  │  ┌────────────────┐  ┌──────────────────┐  │
  │  │ Traffic Access  │  │ Traffic Specs    │  │
  │  │ Control         │  │                  │  │
  │  │ (who can talk   │  │ (define routes   │  │
  │  │  to whom)       │  │  and matches)    │  │
  │  └────────────────┘  └──────────────────┘  │
  │                                             │
  │  ┌────────────────┐  ┌──────────────────┐  │
  │  │ Traffic Split  │  │ Traffic Metrics  │  │
  │  │                │  │                  │  │
  │  │ (canary, A/B   │  │ (standard       │  │
  │  │  traffic       │  │  metrics API)   │  │
  │  │  splitting)    │  │                  │  │
  │  └────────────────┘  └──────────────────┘  │
  │                                             │
  └────────────────────────────────────────────┘

  Implementations: Linkerd, Consul Connect, Open Service Mesh
  (Istio has partial SMI support via adapter)
```

SMI aims to be like CNI/CRI/CSI — a standard interface. In practice, adoption varies, and many meshes have their own native APIs.

---

## When to Use a Service Mesh

```
  Decision Guide

  Do you have many microservices (10+)?
  ├─ No ──► Probably don't need a mesh yet
  │
  ├─ Yes
  │   │
  │   ├─ Need mTLS between services?
  │   │   └─ Yes ──► Service mesh helps
  │   │
  │   ├─ Need detailed service-to-service metrics?
  │   │   └─ Yes ──► Service mesh helps
  │   │
  │   ├─ Need advanced traffic management (canary, retries)?
  │   │   └─ Yes ──► Service mesh helps
  │   │
  │   └─ Want to keep it simple?
  │       ├─ Yes ──► Linkerd
  │       └─ Need advanced features ──► Istio
  │
  └─ A mesh adds complexity. Use it when the benefits outweigh the overhead.
```

---

## Key Exam Points

- A service mesh handles **mTLS, observability, traffic management, retries, and circuit breaking** without code changes.
- **Sidecar pattern**: A proxy container is injected into every pod to intercept all traffic.
- **Envoy** is the most common sidecar proxy (used by Istio, also standalone).
- **Data plane** = sidecar proxies handling actual traffic. **Control plane** = management components (istiod, etc.).
- **Istio** = feature-rich, complex, uses Envoy, CNCF Graduated.
- **Linkerd** = lightweight, simple, uses linkerd2-proxy (Rust), CNCF Graduated.
- **SMI** (Service Mesh Interface) = standard API for service meshes, similar to CNI/CRI/CSI.
- A service mesh adds overhead — use it when you have enough microservices to justify it.

---

## What to Remember for the Exam

1. **Core capabilities**: mTLS, observability, traffic management, retries, circuit breaking. Know these by name.
2. **Sidecar proxy pattern**: proxy injected into each pod, intercepts all traffic. Application is unaware.
3. **Two planes**: Data plane (proxies) and Control plane (management). Be able to identify which components belong where.
4. **Istio architecture**: istiod (control plane) + Envoy sidecars (data plane).
5. **Linkerd vs Istio**: Linkerd is simpler and lighter. Istio is more feature-rich. Both are CNCF Graduated.
6. **Envoy**: Originally from Lyft, CNCF Graduated, most popular proxy for service meshes.
7. **SMI**: Standard API for service meshes. Know it exists and what it standardizes (traffic access, traffic split, traffic specs, traffic metrics).
8. **mTLS**: Mutual TLS — both client and server verify each other's identity. This is the killer feature of service meshes.
