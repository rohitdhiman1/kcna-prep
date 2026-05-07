# Services

**A Service is a stable network endpoint that provides load balancing and discovery for a set of Pods, using label selectors to dynamically route traffic to healthy backend Pods.**

---

## Table of Contents

- [Why Services?](#why-services)
- [How Services Work](#how-services-work)
- [Service Types](#service-types)
- [Service Discovery](#service-discovery)
- [Endpoints and EndpointSlices](#endpoints-and-endpointslices)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## Why Services?

Pods are **ephemeral** — they come and go, and each gets a different IP address. You cannot rely on Pod IPs directly. A Service provides:

- A **stable IP address** (ClusterIP) that does not change
- A **stable DNS name** (e.g., `my-service.default.svc.cluster.local`)
- **Load balancing** across matching Pods
- **Automatic updates** when Pods are added or removed

```
  WITHOUT a Service:                  WITH a Service:

  Client                              Client
    |                                   |
    +--> Pod IP 10.244.1.5 (dead!)      +--> Service IP 10.96.0.10
    +--> Pod IP 10.244.2.8 (maybe?)           (stable, never changes)
    +--> Pod IP ??? (new pod)                  |
                                          +----+----+
                                          |    |    |
                                          v    v    v
                                        Pod  Pod  Pod
                                        (IPs don't matter)
```

---

## How Services Work

Services use **label selectors** to find their backend Pods:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web          # <-- Matches Pods with label app=web
  ports:
  - port: 80          # Service port (what clients connect to)
    targetPort: 8080   # Container port (where traffic is sent)
  type: ClusterIP      # Default type
```

```
  Service: web-service (10.96.0.10:80)
  Selector: app=web
       |
       |  Routes traffic to all Pods matching app=web
       |
       +-------+-------+-------+
       |       |       |       |
       v       v       v       v
    +------+ +------+ +------+ +------+
    | Pod  | | Pod  | | Pod  | | Pod  |  (NOT selected,
    | app= | | app= | | app= | | app= |   different label)
    | web  | | web  | | web  | | api  |
    +------+ +------+ +------+ +------+
    Selected  Selected  Selected  Skipped
```

---

## Service Types

### 1. ClusterIP (Default)

Exposes the Service on an **internal cluster IP**. Only accessible from within the cluster.

```
  +------------------------------------------------------+
  |  CLUSTER                                             |
  |                                                      |
  |  Client Pod                Service (ClusterIP)       |
  |  +----------+             +------------------+       |
  |  | app-pod  | ----------> | web-service      |       |
  |  +----------+             | 10.96.0.10:80    |       |
  |                            +--------+---------+       |
  |                                     |                |
  |                              +------+------+         |
  |                              |      |      |         |
  |                              v      v      v         |
  |                           +----+ +----+ +----+       |
  |                           |Pod | |Pod | |Pod |       |
  |                           +----+ +----+ +----+       |
  +------------------------------------------------------+
  |  OUTSIDE CLUSTER — Cannot reach ClusterIP            |
  +------------------------------------------------------+
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP       # Default, can be omitted
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

**Use case**: Internal communication between microservices.

### 2. NodePort

Exposes the Service on a **static port on every node** in the cluster. Accessible from outside via `<NodeIP>:<NodePort>`.

```
  +------------------------------------------------------+
  |  OUTSIDE CLUSTER                                     |
  |                                                      |
  |  External Client                                     |
  |  http://NODE_IP:30080                                |
  +-----------|------------------------------------------+
              |
  +-----------v------------------------------------------+
  |  CLUSTER                                             |
  |                                                      |
  |  +----------+    NodePort       +------------------+ |
  |  | Node 1   |----- 30080 ----->| web-service      | |
  |  +----------+                  | ClusterIP:80     | |
  |  +----------+    NodePort      | NodePort:30080   | |
  |  | Node 2   |----- 30080 ----->+--------+---------+ |
  |  +----------+                           |            |
  |  +----------+    NodePort        +------+------+     |
  |  | Node 3   |----- 30080 ----->  |      |      |    |
  |  +----------+                    v      v      v    |
  |                               +----+ +----+ +----+  |
  |                               |Pod | |Pod | |Pod |  |
  |                               +----+ +----+ +----+  |
  +------------------------------------------------------+
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80             # ClusterIP port
    targetPort: 8080      # Container port
    nodePort: 30080       # Node port (30000-32767 range)
```

- Port range: **30000-32767** (configurable)
- A NodePort Service also creates a ClusterIP automatically
- Traffic to any node on that port gets forwarded to the Service

**Use case**: Development/testing, direct access without a load balancer.

### 3. LoadBalancer

Provisions an **external load balancer** from the cloud provider and assigns a public IP.

```
  +------------------------------------------------------+
  |  INTERNET                                            |
  |                                                      |
  |  External Client                                     |
  |  http://203.0.113.50:80                              |
  +-----------|------------------------------------------+
              |
  +-----------v------------------------------------------+
  |  CLOUD LOAD BALANCER                                 |
  |  External IP: 203.0.113.50                           |
  |  Distributes traffic to nodes                        |
  +-----------|---------|---------|-----------------------+
              |         |         |
  +-----------v---------v---------v----------------------+
  |  CLUSTER                                             |
  |  +----------+  +----------+  +----------+            |
  |  | Node 1   |  | Node 2   |  | Node 3   |            |
  |  +----------+  +----------+  +----------+            |
  |       |              |             |                 |
  |       +-------- Service (LB) -----+                  |
  |                     |                                |
  |              +------+------+                         |
  |              |      |      |                         |
  |              v      v      v                         |
  |           +----+ +----+ +----+                       |
  |           |Pod | |Pod | |Pod |                       |
  |           +----+ +----+ +----+                       |
  +------------------------------------------------------+
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

- Builds on top of NodePort (which builds on ClusterIP)
- Requires a cloud provider or MetalLB for bare metal
- Gets an external IP provisioned automatically

**Use case**: Production services that need to be accessible from the internet.

### 4. ExternalName

Maps a Service to an **external DNS name**. No proxying — just a CNAME redirect.

```
  +------------------------------------------------------+
  |  CLUSTER                                             |
  |                                                      |
  |  Client Pod                                          |
  |  +----------+                                        |
  |  | nslookup |     Service: db-service                |
  |  | db-svc   | --> (ExternalName)                     |
  |  +----------+     Returns CNAME:                     |
  |                    db.example.com                     |
  +------------------------------------------------------+
              |
              v
  +------------------------------------------------------+
  |  EXTERNAL                                            |
  |  db.example.com (actual database)                    |
  +------------------------------------------------------+
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-service
spec:
  type: ExternalName
  externalName: db.example.com     # No selector, no ClusterIP
```

- No selector, no ClusterIP, no proxying
- Just creates a DNS CNAME record
- Useful for referencing external services by a stable internal name

**Use case**: Integrating with external services (external database, third-party APIs).

### Service Types Comparison

| Type            | ClusterIP | Accessible From       | External IP | Use Case                    |
|-----------------|-----------|----------------------|-------------|-----------------------------|
| ClusterIP       | Yes       | Inside cluster only  | No          | Internal microservices      |
| NodePort        | Yes       | NodeIP:Port          | No          | Dev/testing                 |
| LoadBalancer    | Yes       | External LB IP       | Yes         | Production external access  |
| ExternalName    | No        | DNS CNAME redirect   | No          | External service reference  |

### Layering

```
  LoadBalancer
       |
       | includes
       v
  NodePort
       |
       | includes
       v
  ClusterIP (base)
```

---

## Service Discovery

Pods can discover Services in two ways:

### 1. DNS (Preferred)

Kubernetes runs a DNS service (CoreDNS) that automatically creates DNS records for Services:

```
  Service DNS Format:
  <service-name>.<namespace>.svc.cluster.local

  Examples:
  web-service.default.svc.cluster.local
  web-service.default.svc
  web-service.default
  web-service          (within same namespace)

  From a Pod in the "default" namespace:
  +----------+
  | curl     |  curl http://web-service:80
  | client   |  (resolves to ClusterIP 10.96.0.10)
  +----------+
```

### 2. Environment Variables

When a Pod starts, Kubernetes injects environment variables for each Service that exists at that time:

```
  WEB_SERVICE_SERVICE_HOST=10.96.0.10
  WEB_SERVICE_SERVICE_PORT=80
  WEB_SERVICE_PORT=tcp://10.96.0.10:80
  WEB_SERVICE_PORT_80_TCP=tcp://10.96.0.10:80
  WEB_SERVICE_PORT_80_TCP_PROTO=tcp
  WEB_SERVICE_PORT_80_TCP_PORT=80
  WEB_SERVICE_PORT_80_TCP_ADDR=10.96.0.10
```

> **Limitation**: Environment variables only include Services that existed BEFORE the Pod started. DNS does not have this limitation and is the preferred method.

---

## Endpoints and EndpointSlices

When a Service is created, Kubernetes automatically creates an **Endpoints** object (and **EndpointSlices** in newer versions) that tracks the IPs of matching Pods.

```
  Service: web-service                  Endpoints: web-service
  selector: app=web                     addresses:
  ClusterIP: 10.96.0.10                  - 10.244.1.5 (Pod A)
       |                                 - 10.244.2.8 (Pod B)
       |                                 - 10.244.3.2 (Pod C)
       +--- managed by --->
       Endpoint Controller

  When Pod B fails readiness probe:
  Endpoints: web-service
  addresses:
    - 10.244.1.5 (Pod A)
    - 10.244.3.2 (Pod C)           <-- Pod B removed
```

---

## What to Remember for the Exam

1. **Services provide stable endpoints** for ephemeral Pods. They use label selectors to find backend Pods.

2. **Four Service types**: ClusterIP (internal, default), NodePort (external via node ports 30000-32767), LoadBalancer (external via cloud LB), ExternalName (DNS CNAME).

3. **ClusterIP is the default** and is included in NodePort and LoadBalancer types (they layer on top).

4. **Service discovery**: DNS is preferred (`<service>.<namespace>.svc.cluster.local`). Environment variables are injected but only for pre-existing Services.

5. **Port mapping**: `port` is the Service port (what clients connect to), `targetPort` is the container port (where traffic goes).

6. **Label selectors** connect Services to Pods. A Pod must have matching labels to receive traffic from a Service.

7. **Endpoints** are automatically maintained by the Endpoint controller. When a Pod fails its readiness probe, it is removed from Endpoints.

8. **NodePort range** is 30000-32767 by default.

9. **LoadBalancer requires** a cloud provider or an implementation like MetalLB.

10. **ExternalName** has no selector, no ClusterIP, and no proxying — it just returns a DNS CNAME record.
