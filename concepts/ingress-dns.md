# Ingress and DNS

> Ingress provides HTTP/HTTPS routing from outside the cluster to Services inside. CoreDNS provides service discovery within the cluster using DNS names.

---

## The Problem Ingress Solves

Without Ingress, exposing multiple HTTP services requires multiple LoadBalancer Services (each gets its own external IP and costs money in the cloud). Ingress consolidates routing behind a single entry point.

```
  Without Ingress                    With Ingress
  (one LB per service)               (one LB, routing rules)

  Internet                           Internet
     │                                  │
     ├──► LB 1 ──► Service A            │
     │                                  ▼
     ├──► LB 2 ──► Service B      ┌──────────┐
     │                             │ Ingress  │
     └──► LB 3 ──► Service C      │Controller│
                                   └────┬─────┘
     3 external IPs                     │
     3 LB costs                    ┌────┼────┐
                                   ▼    ▼    ▼
                                  Svc  Svc  Svc
                                   A    B    C

                                   1 external IP
                                   1 LB cost
```

---

## Ingress Architecture

Ingress has two components:

### 1. Ingress Resource

A Kubernetes API object that defines routing rules (YAML). It is **just configuration** — it does nothing by itself.

### 2. Ingress Controller

A pod running a reverse proxy (NGINX, Traefik, HAProxy, etc.) that **reads Ingress resources** and configures itself to route traffic accordingly.

```
  Ingress Architecture

  ┌─────────────────────────────────────────────────────────────┐
  │                      Kubernetes Cluster                      │
  │                                                              │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │              Ingress Controller (Pod)                 │   │
  │  │         (e.g., NGINX Ingress Controller)              │   │
  │  │                                                       │   │
  │  │  Watches ──► Ingress Resources (API)                  │   │
  │  │  Generates ──► nginx.conf (or equivalent)             │   │
  │  │  Listens on ──► ports 80 and 443                      │   │
  │  └──────────────────────┬───────────────────────────────┘   │
  │                          │                                   │
  │          ┌───────────────┼───────────────┐                   │
  │          ▼               ▼               ▼                   │
  │    ┌──────────┐    ┌──────────┐    ┌──────────┐             │
  │    │ Service  │    │ Service  │    │ Service  │             │
  │    │  app-a   │    │  app-b   │    │  app-c   │             │
  │    └────┬─────┘    └────┬─────┘    └────┬─────┘             │
  │         ▼               ▼               ▼                   │
  │    ┌────────┐      ┌────────┐      ┌────────┐              │
  │    │ Pods   │      │ Pods   │      │ Pods   │              │
  │    └────────┘      └────────┘      └────────┘              │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

**Important**: An Ingress resource without an Ingress Controller does nothing. You must install a controller first.

---

## Ingress Routing Rules

### Host-Based Routing

Route traffic based on the hostname in the HTTP request:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  rules:
  - host: app-a.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-a-service
            port:
              number: 80
  - host: app-b.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-b-service
            port:
              number: 80
```

```
  Host-Based Routing

  app-a.example.com ──► Ingress Controller ──► app-a-service ──► Pods
  app-b.example.com ──► Ingress Controller ──► app-b-service ──► Pods
```

### Path-Based Routing

Route traffic based on the URL path:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

```
  Path-Based Routing

  example.com/api  ──► Ingress Controller ──► api-service ──► API Pods
  example.com/web  ──► Ingress Controller ──► web-service ──► Web Pods
```

### Path Types

| Type | Behavior |
|------|----------|
| **Prefix** | Matches the URL path prefix (e.g., `/api` matches `/api`, `/api/v1`, `/api/users`) |
| **Exact** | Matches the exact URL path only (e.g., `/api` matches only `/api`, not `/api/v1`) |
| **ImplementationSpecific** | Matching depends on the Ingress Controller |

---

## TLS Termination

Ingress can terminate TLS (HTTPS) using a Kubernetes Secret containing the certificate and key:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: example-tls-secret   # Contains tls.crt and tls.key
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

```
  TLS Termination

  Client ──(HTTPS)──► Ingress Controller ──(HTTP)──► Service ──► Pods
                       ▲
                       │ TLS terminated here
                       │ Certificate from K8s Secret
```

---

## Popular Ingress Controllers

| Controller | Maintained By | Notes |
|-----------|--------------|-------|
| **NGINX Ingress Controller** | Kubernetes community | Most widely used |
| **Traefik** | Traefik Labs | Auto-discovery, Let's Encrypt integration |
| **HAProxy Ingress** | HAProxy Technologies | High performance |
| **Contour** | VMware/CNCF | Uses Envoy proxy under the hood |
| **AWS ALB Ingress** | AWS | Provisions AWS ALB automatically |

For the exam, know that **NGINX** is the most common Ingress Controller.

---

## Ingress vs Gateway API

The **Gateway API** is the successor to Ingress, offering more features:

| Feature | Ingress | Gateway API |
|---------|---------|-------------|
| Protocol support | HTTP/HTTPS only | HTTP, HTTPS, TCP, UDP, gRPC |
| Role separation | Single resource | GatewayClass, Gateway, HTTPRoute (separated) |
| Header-based routing | Annotation-dependent | Native support |
| Traffic splitting | Not native | Native support |
| Status | Stable, widely used | GA since K8s 1.26, growing adoption |

For the KCNA exam, **Ingress** is the primary focus. Gateway API is good to know exists.

---

## DNS in Kubernetes

### CoreDNS

CoreDNS is the default cluster DNS server. It:
- Runs as a Deployment in the `kube-system` namespace.
- Watches the Kubernetes API for new Services and Endpoints.
- Creates DNS records automatically.
- Is configured via a ConfigMap (`coredns` in `kube-system`).

### Service DNS Format

```
  Full DNS name:
  <service-name>.<namespace>.svc.cluster.local

  ┌──────────┐  ┌───────────┐  ┌─────┐  ┌───────────────┐
  │ service  │  │ namespace │  │ svc │  │ cluster.local │
  │  name    │  │   name    │  │     │  │ (cluster      │
  │          │  │           │  │     │  │  domain)      │
  └──────────┘  └───────────┘  └─────┘  └───────────────┘

  Examples:
  ┌────────────────────────────────────────────────────────┐
  │ my-app.default.svc.cluster.local        → ClusterIP   │
  │ postgres.database.svc.cluster.local     → ClusterIP   │
  │ my-app                                  → same ns     │
  │ my-app.database                         → cross-ns    │
  └────────────────────────────────────────────────────────┘
```

### Pod DNS Records

Pods also get DNS records (but less commonly used):

```
  Pod DNS format:
  <pod-ip-with-dashes>.<namespace>.pod.cluster.local

  Example:
  10-244-1-5.default.pod.cluster.local
```

### Headless Services

A **headless Service** has `clusterIP: None`. Instead of returning a single ClusterIP, DNS returns the **individual pod IPs**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headless-service
spec:
  clusterIP: None          # <-- This makes it headless
  selector:
    app: my-app
  ports:
  - port: 80
```

```
  Regular Service DNS:
  my-service.default.svc.cluster.local ──► 10.96.45.12 (ClusterIP)

  Headless Service DNS:
  my-headless.default.svc.cluster.local ──► 10.244.1.5 (Pod IP)
                                         ──► 10.244.2.8 (Pod IP)
                                         ──► 10.244.1.9 (Pod IP)

  Returns all pod IPs directly (A records for each pod)
```

Use cases for headless services:
- **StatefulSets** — each pod needs a stable, unique DNS name
- **Service discovery** — client needs to know all endpoints
- **Database clusters** — connect to specific replicas

---

## Full Traffic Flow: External to Pod

```
  External Traffic Flow

  User's Browser
       │
       │  DNS: app.example.com ──► External DNS ──► LoadBalancer IP
       │
       ▼
  ┌──────────────────┐
  │  Cloud Load      │
  │  Balancer        │  (or NodePort)
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │  Ingress         │  Reads Ingress rules
  │  Controller      │  Matches host + path
  │  (NGINX pod)     │  TLS termination
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │  Service         │  kube-proxy rules
  │  (ClusterIP)     │  load balance to pods
  └────────┬─────────┘
           │
     ┌─────┼─────┐
     ▼     ▼     ▼
  ┌─────┐┌─────┐┌─────┐
  │Pod 1││Pod 2││Pod 3│
  └─────┘└─────┘└─────┘
```

---

## Key Exam Points

- **Ingress** = routing rules (YAML). **Ingress Controller** = the software that implements those rules (NGINX, Traefik).
- Ingress supports **host-based** and **path-based** routing.
- **TLS termination** is handled by the Ingress Controller using TLS Secrets.
- **An Ingress resource does nothing without an Ingress Controller installed.**
- CoreDNS provides automatic DNS for Services: `service.namespace.svc.cluster.local`.
- **Headless Services** (`clusterIP: None`) return pod IPs directly instead of a virtual IP.
- Gateway API is the modern successor to Ingress but Ingress is still the exam focus.

---

## What to Remember for the Exam

1. **Two parts**: Ingress Resource (rules) + Ingress Controller (implementation). Both are needed.
2. **Routing types**: Host-based (different domains) and path-based (different URL paths). Know both.
3. **NGINX** is the most common Ingress Controller.
4. **DNS format**: `service.namespace.svc.cluster.local`. Be able to construct this from a service name and namespace.
5. **Headless services**: `clusterIP: None` returns pod IPs directly. Used with StatefulSets.
6. **CoreDNS**: Runs in `kube-system`, watches API for Services, creates DNS records automatically.
7. **Path types**: Prefix (matches path and sub-paths) vs Exact (matches only the exact path).
