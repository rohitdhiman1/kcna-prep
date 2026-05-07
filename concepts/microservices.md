# Microservices Architecture

> Breaking applications into small, independently deployable services — the dominant architecture pattern in cloud native systems.

---

## Monolithic vs Microservices

### Monolithic Architecture

A monolith is a single deployable unit where all components (UI, business logic, data access) are packaged together.

```
  ┌──────────────────────────────────────────┐
  │            Monolithic Application          │
  │                                            │
  │  ┌────────┐  ┌────────┐  ┌────────────┐  │
  │  │  User  │  │ Order  │  │  Payment   │  │
  │  │ Module │  │ Module │  │  Module    │  │
  │  └───┬────┘  └───┬────┘  └─────┬──────┘  │
  │      │           │             │          │
  │      └───────────┼─────────────┘          │
  │                  │                         │
  │          ┌───────┴───────┐                │
  │          │  Shared DB    │                │
  │          └───────────────┘                │
  │                                            │
  │  Single process, single deployment unit    │
  └──────────────────────────────────────────┘
```

### Microservices Architecture

Each service is a small, independent unit that owns its own data and communicates over the network.

```
  ┌───────────────────────────────────────────────────────────┐
  │                 Microservices Architecture                  │
  │                                                             │
  │            ┌──────────────────────────┐                    │
  │            │      API Gateway         │                    │
  │            └─────┬──────┬──────┬──────┘                    │
  │                  │      │      │                            │
  │         ┌────────┘      │      └────────┐                  │
  │         ▼               ▼               ▼                  │
  │   ┌──────────┐   ┌──────────┐   ┌──────────┐             │
  │   │  User    │   │  Order   │   │ Payment  │             │
  │   │ Service  │   │ Service  │   │ Service  │             │
  │   │          │   │          │   │          │             │
  │   │  Go      │   │  Java   │   │  Python  │  <-- tech   │
  │   └────┬─────┘   └────┬─────┘   └────┬─────┘    diversity│
  │        │              │              │                    │
  │   ┌────┴─────┐   ┌────┴─────┐   ┌────┴─────┐            │
  │   │ User DB  │   │ Order DB │   │ Pay DB   │            │
  │   │ (Postgres)│   │ (MongoDB)│   │ (MySQL)  │            │
  │   └──────────┘   └──────────┘   └──────────┘            │
  │                                                             │
  │   Each service: own repo, own DB, own deploy cycle         │
  └───────────────────────────────────────────────────────────┘
```

### Side-by-Side Comparison

```
    Monolith                          Microservices
    ────────                          ──────────────

    ┌──────────────┐                 ┌────┐ ┌────┐ ┌────┐
    │              │                 │ A  │ │ B  │ │ C  │
    │  Everything  │      vs.       │    │ │    │ │    │
    │  in one box  │                 └─┬──┘ └─┬──┘ └─┬──┘
    │              │                   │      │      │
    └──────┬───────┘                 ┌─┴──┐ ┌─┴──┐ ┌─┴──┐
           │                         │DB A│ │DB B│ │DB C│
    ┌──────┴───────┐                 └────┘ └────┘ └────┘
    │  Single DB   │
    └──────────────┘                 Independent units
    One deploy = everything          Each deploys alone
```

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| Deployment | All-or-nothing | Independent per service |
| Scaling | Entire application | Individual services |
| Technology | Single stack | Polyglot (mix languages/frameworks) |
| Team structure | One large team | Small, autonomous teams per service |
| Data | Shared database | Database per service |
| Failure blast radius | Whole application | Single service |
| Complexity | In the code | In the network and infrastructure |
| Testing | Simpler (one unit) | Harder (distributed integration tests) |
| Deployment speed | Slower (bigger releases) | Faster (small, frequent releases) |

---

## Benefits of Microservices

### 1. Independent Deployment
- Each service has its own CI/CD pipeline.
- Teams can release without coordinating with other teams.
- Smaller deployments mean lower risk.

### 2. Technology Diversity
- Each service can use the best language, framework, or database for its job.
- Example: User service in Go, ML service in Python, frontend in Node.js.
- Teams are free to experiment and adopt new technologies.

### 3. Team Autonomy
- Small teams (often called "two-pizza teams") own a service end-to-end.
- Teams can make decisions about architecture, technology, and release schedule independently.
- Aligns with **Conway's Law**: system structure mirrors organization structure.

### 4. Fault Isolation
- A bug or crash in one service does not bring down the entire application.
- Circuit breakers and bulkheads prevent cascading failures.
- Services can degrade gracefully.

### 5. Independent Scaling
- Scale only the services that need it (e.g., the search service during high traffic) instead of the entire application.
- More cost-efficient resource usage.

---

## Challenges of Microservices

### 1. Distributed System Complexity
- Network calls replace function calls — more things can go wrong.
- Need to handle partial failures, retries, and timeouts.
- Debugging spans multiple services (requires distributed tracing).

### 2. Data Consistency
- No shared database means no simple transactions across services.
- Must use patterns like **eventual consistency**, **Saga pattern**, or **event sourcing**.
- Data duplication across services is common and expected.

### 3. Network Latency
- Every inter-service call has network overhead.
- "Chatty" services (too many calls) cause performance problems.
- Need to design APIs carefully (coarse-grained over fine-grained).

### 4. Operational Overhead
- More services = more things to deploy, monitor, and debug.
- Requires mature DevOps practices, CI/CD, observability stack.
- Service discovery, load balancing, and configuration management become critical.

### 5. Testing Complexity
- Integration testing across services is harder.
- Need contract testing, end-to-end testing in staging environments.
- Local development environment is more complex.

---

## When a Monolith Is Better

Microservices are NOT always the right answer. A monolith is often better when:

- **Small team**: Fewer than 5-10 developers. The overhead of microservices is not justified.
- **Simple application**: The app does not have clear domain boundaries to split on.
- **Early-stage product**: You do not yet know where the boundaries should be. Start monolithic, split later ("monolith first" approach).
- **Tight deadlines**: Microservices add infrastructure complexity that slows initial development.

**Martin Fowler's advice**: "Don't start with microservices. Start with a monolith, keep it modular, and extract microservices when you have a clear need."

---

## The API Gateway Pattern

An API Gateway is a single entry point for all client requests that routes them to the appropriate microservice.

```
                    ┌─────────┐
                    │ Client  │
                    │(Browser,│
                    │ Mobile) │
                    └────┬────┘
                         │
                         ▼
               ┌─────────────────┐
               │   API Gateway   │
               │                 │
               │ - Routing       │
               │ - Auth          │
               │ - Rate limiting │
               │ - SSL           │
               │ - Load balance  │
               └───┬─────┬───┬──┘
                   │     │   │
          ┌────────┘     │   └────────┐
          ▼              ▼            ▼
    ┌──────────┐  ┌──────────┐  ┌──────────┐
    │  User    │  │  Order   │  │ Product  │
    │ Service  │  │ Service  │  │ Service  │
    └──────────┘  └──────────┘  └──────────┘
```

**What the API Gateway does:**
- **Routing**: Directs `/users/*` to the User Service, `/orders/*` to the Order Service, etc.
- **Authentication/Authorization**: Validates tokens before forwarding requests.
- **Rate limiting**: Protects services from traffic spikes.
- **SSL termination**: Handles HTTPS at the edge.
- **Load balancing**: Distributes traffic across service replicas.
- **Request aggregation**: Combines responses from multiple services into one response for the client.

**In Kubernetes**, the API Gateway role is often filled by:
- **Ingress controllers** (NGINX, Traefik)
- **Service mesh ingress** (Istio Gateway)
- **Dedicated gateways** (Kong, Ambassador/Emissary)

---

## Communication Patterns

Microservices communicate in two main ways:

### Synchronous (Request/Response)
- REST over HTTP or gRPC.
- Caller waits for a response.
- Simpler but creates tight coupling.

### Asynchronous (Event-Driven)
- Services communicate via message brokers (Kafka, RabbitMQ, NATS).
- Producer publishes an event; consumer processes it later.
- Loose coupling, better resilience, but harder to debug.

```
  Synchronous                      Asynchronous

  ┌─────┐  HTTP/gRPC  ┌─────┐    ┌─────┐  event  ┌───────┐  event  ┌─────┐
  │ Svc ├────────────►│ Svc │    │ Svc ├────────►│ Broker├────────►│ Svc │
  │  A  │◄────────────┤  B  │    │  A  │         │(Kafka)│         │  B  │
  └─────┘  response   └─────┘    └─────┘         └───────┘         └─────┘

  A waits for B                    A fires and forgets
  Tight coupling                   Loose coupling
```

---

## What to Remember for the Exam

1. **Microservices** = small, independently deployable services, each with its own database.
2. **Monolith** = single deployable unit. Better for small teams and early-stage projects.
3. Key benefits of microservices: **independent deployment, technology diversity, team autonomy, fault isolation, independent scaling**.
4. Key challenges: **distributed complexity, data consistency, network latency, operational overhead**.
5. **API Gateway** provides a single entry point for clients and handles routing, auth, rate limiting.
6. **Communication**: synchronous (REST/gRPC) vs asynchronous (message broker/events).
7. Microservices align with cloud native principles: each service can be containerized, scaled independently, and deployed via CI/CD.
8. Know that microservices are NOT always better — small teams should start with a monolith.
