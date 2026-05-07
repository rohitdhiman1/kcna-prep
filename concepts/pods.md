# Pods

**A Pod is the smallest deployable unit in Kubernetes -- a wrapper around one or more containers that share the same network namespace, IP address, and storage volumes.**

---

## Table of Contents

- [What Is a Pod?](#what-is-a-pod)
- [Pod Structure Diagram](#pod-structure-diagram)
- [Pod Spec](#pod-spec)
- [Single vs Multi-Container Pods](#single-vs-multi-container-pods)
- [Init Containers](#init-containers)
- [Sidecar Containers](#sidecar-containers)
- [Pod Lifecycle](#pod-lifecycle)
- [Restart Policies](#restart-policies)
- [Resource Requests and Limits](#resource-requests-and-limits)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## What Is a Pod?

A Pod is:
- The **smallest unit** you can deploy in Kubernetes (you cannot deploy a bare container)
- A **wrapper** around one or more containers
- Containers in a Pod share the **same network namespace** (same IP, same ports)
- Containers in a Pod can share **volumes**
- Typically you run **one container per Pod**, unless containers are tightly coupled

Think of a Pod as a **logical host** — containers in the same Pod are like processes on the same machine.

---

## Pod Structure Diagram

### Single-Container Pod (Most Common)

```
+------------------------------------------------------+
|  POD (IP: 10.244.1.5)                               |
|                                                      |
|  +------------------------------------------------+ |
|  |  Container: nginx                               | |
|  |  Image: nginx:1.25                              | |
|  |  Ports: 80                                      | |
|  |  Resources: 100m CPU, 128Mi memory              | |
|  +------------------------------------------------+ |
|                                                      |
|  +-------------------+                               |
|  | Volume: html-data |                               |
|  | (shared storage)  |                               |
|  +-------------------+                               |
|                                                      |
|  Network: localhost communication between containers |
|  DNS: Gets cluster DNS (/etc/resolv.conf)            |
+------------------------------------------------------+
```

### Multi-Container Pod

```
+------------------------------------------------------+
|  POD (IP: 10.244.1.5)                               |
|                                                      |
|  +---------------------+  +------------------------+ |
|  |  Container: app      |  |  Container: log-agent  | |
|  |  Image: my-app:v2    |  |  Image: fluentd:latest | |
|  |  Port: 8080          |  |  (no port exposed)     | |
|  |  Writes logs to      |  |  Reads logs from       | |
|  |  /var/log/app.log    |  |  /var/log/app.log      | |
|  +---------------------+  +------------------------+ |
|          |                          |                 |
|          +-----+  shared volume  +--+                 |
|                |                 |                     |
|         +------+-----------------+------+             |
|         | Volume: log-volume             |             |
|         | mountPath: /var/log            |             |
|         +--------------------------------+             |
|                                                      |
|  Containers share:                                   |
|    - Same IP address (10.244.1.5)                    |
|    - Same localhost (can reach each other on 127.0.0.1) |
|    - Same volumes (if mounted)                       |
|    - Same IPC namespace                              |
+------------------------------------------------------+
```

### Pod with Init Container

```
+------------------------------------------------------+
|  POD                                                 |
|                                                      |
|  PHASE 1: Init Containers (run sequentially)         |
|  +--------------------------------------------------+|
|  | Init Container: wait-for-db                       ||
|  | Image: busybox                                    ||
|  | Command: until nslookup db; do sleep 2; done      ||
|  | Status: Completed                                 ||
|  +--------------------------------------------------+|
|                     |                                 |
|                     v  (init completes, then...)      |
|                                                      |
|  PHASE 2: App Containers (run in parallel)           |
|  +--------------------------------------------------+|
|  | Container: web-app                                ||
|  | Image: my-app:v2                                  ||
|  | Port: 8080                                        ||
|  +--------------------------------------------------+|
+------------------------------------------------------+
```

---

## Pod Spec

A minimal Pod manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: web
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
```

### Key Fields

```
  Pod Manifest Structure
  =======================

  apiVersion: v1              <-- API version (core group)
  kind: Pod                   <-- Resource type
  metadata:
    name: nginx               <-- Pod name (unique in namespace)
    namespace: default        <-- Namespace (default if omitted)
    labels: {}                <-- Key-value pairs for selection
    annotations: {}           <-- Key-value pairs for metadata
  spec:
    containers: []            <-- List of app containers (required)
    initContainers: []        <-- List of init containers (optional)
    volumes: []               <-- Volumes available to containers
    restartPolicy: Always     <-- Always | OnFailure | Never
    nodeSelector: {}          <-- Schedule to nodes with these labels
    tolerations: []           <-- Tolerate node taints
    serviceAccountName: ""    <-- Service account to use
```

---

## Single vs Multi-Container Pods

### When to Use Single-Container Pods

Most Pods have **one container**. This is the default pattern:

- One microservice per Pod
- Scales independently
- Simplest to manage

### When to Use Multi-Container Pods

Multi-container Pods are used when containers are **tightly coupled** and need to share resources:

| Pattern     | Description                                     | Example                              |
|-------------|-------------------------------------------------|--------------------------------------|
| **Sidecar** | Enhances or extends the main container          | Log collector, proxy, config watcher |
| **Ambassador** | Proxies network traffic for the main container | localhost proxy to external DB      |
| **Adapter** | Transforms output of the main container         | Log format converter, metrics export |

```
  Sidecar Pattern:
  +------------------------------------+
  |  Pod                               |
  |  +----------+    +----------+      |
  |  | main-app | -> | log-ship |      |
  |  +----------+    +----------+      |
  |       |               |            |
  |       +---shared vol--+            |
  +------------------------------------+

  Ambassador Pattern:
  +------------------------------------+
  |  Pod                               |
  |  +----------+    +----------+      |
  |  | main-app | -> | proxy    | ---> External DB
  |  | (localhost|    | (localhost|     |
  |  |  :3306)  |    |  :3306)  |     |
  |  +----------+    +----------+      |
  +------------------------------------+

  Adapter Pattern:
  +------------------------------------+
  |  Pod                               |
  |  +----------+    +----------+      |
  |  | main-app | -> | adapter  | ---> Monitoring
  |  | (custom  |    | (converts|      (Prometheus
  |  |  logs)   |    |  format) |       format)
  |  +----------+    +----------+      |
  +------------------------------------+
```

---

## Init Containers

Init containers run **before** app containers start. They run **sequentially** (one at a time) and must complete successfully before the next one begins.

### Use Cases

- Wait for a dependency (database, service) to be available
- Run database migrations
- Clone a Git repo into a shared volume
- Generate configuration files
- Set up permissions on volumes

### Example: Wait for a Database

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nslookup db-service; do echo waiting for db; sleep 2; done']
  containers:
  - name: web-app
    image: my-app:v2
    ports:
    - containerPort: 8080
```

### Init Container Rules

- Run to completion before any app container starts
- If an init container fails, Kubernetes restarts the Pod (subject to restartPolicy)
- Each init container must complete successfully before the next one starts
- Init containers do NOT support liveness/readiness probes
- Init containers can have different images and security contexts than app containers

---

## Sidecar Containers

Starting with Kubernetes 1.28+, there is native sidecar container support using `restartPolicy: Always` on init containers. These start before the main containers and run alongside them for the Pod's entire lifetime.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-sidecar
spec:
  initContainers:
  - name: log-agent
    image: fluentd:latest
    restartPolicy: Always    # This makes it a native sidecar
  containers:
  - name: web
    image: nginx:1.25
```

Traditional sidecar pattern (multi-container Pod):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-sidecar
spec:
  containers:
  - name: web
    image: nginx:1.25
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  - name: log-agent
    image: fluentd:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  volumes:
  - name: logs
    emptyDir: {}
```

---

## Pod Lifecycle

```
  Pod Created
       |
       v
  +---------+     Init containers not        +---------+
  | Pending  | --> done, waiting for          | Pending |
  |          |     scheduling, image pull     |         |
  +---------+                                +---------+
       |
       | All containers started
       v
  +---------+
  | Running  |     At least one container
  |          |     is running
  +---------+
       |
       +------------------+------------------+
       |                  |                  |
       v                  v                  v
  +-----------+    +-----------+    +-----------+
  | Succeeded  |    |  Failed   |    |  Unknown   |
  |            |    |           |    |            |
  | All cntrs  |    | At least  |    | Node lost  |
  | exited 0   |    | one cntr  |    | contact    |
  |            |    | exited != 0|    |            |
  +-----------+    +-----------+    +-----------+
```

### Phase Details

| Phase       | Description                                                    |
|-------------|----------------------------------------------------------------|
| **Pending** | Pod accepted but not yet running. Could be scheduling, pulling images, or running init containers. |
| **Running** | Pod bound to a node, at least one container is running or starting. |
| **Succeeded** | All containers exited with code 0 and will not be restarted. |
| **Failed**  | All containers terminated, at least one exited with non-zero code. |
| **Unknown** | Pod status cannot be determined (usually node communication failure). |

### Container States

Within a running Pod, each container is in one of these states:

| State        | Description                                     |
|--------------|--------------------------------------------------|
| **Waiting**  | Not yet running (pulling image, etc.)            |
| **Running**  | Executing without issues                         |
| **Terminated** | Finished execution (exited)                    |

---

## Restart Policies

The `restartPolicy` field controls what happens when a container exits:

| Policy        | Behavior                                            | Used By           |
|---------------|-----------------------------------------------------|--------------------|
| **Always**    | Always restart the container (default)              | Deployments        |
| **OnFailure** | Restart only if container exits with non-zero code  | Jobs               |
| **Never**     | Never restart the container                         | One-off debug Pods |

```
  Container Exits
       |
       +-- restartPolicy: Always -----> Always restart
       |
       +-- restartPolicy: OnFailure --> Restart only if exit code != 0
       |
       +-- restartPolicy: Never ------> Do not restart
```

**Backoff delay**: When a container keeps crashing, Kubernetes applies an exponential backoff: 10s, 20s, 40s, ... up to 5 minutes. This is the "CrashLoopBackOff" state you see in `kubectl get pods`.

---

## Resource Requests and Limits

Every container can specify CPU and memory **requests** (guaranteed minimum) and **limits** (maximum allowed).

```
  +-----------------------------------------------+
  |              Node Resources                    |
  |                                                |
  |  Requests = "I need at least this much"        |
  |  Limits   = "I must not use more than this"    |
  |                                                |
  |  +----Pod A----+  +----Pod B----+              |
  |  | req: 100m   |  | req: 200m   |              |
  |  | lim: 500m   |  | lim: 400m   |              |
  |  +-------------+  +-------------+              |
  |                                                |
  |  Total requested: 300m CPU                     |
  |  Node allocatable: 2000m CPU                   |
  |  Remaining for scheduling: 1700m CPU           |
  +-----------------------------------------------+
```

### CPU

- Measured in **millicores** (m): `100m` = 0.1 CPU core
- `1000m` = 1 full CPU core
- CPU is **compressible** — if a container hits its limit, it gets throttled (not killed)

### Memory

- Measured in bytes: `128Mi` (mebibytes), `1Gi` (gibibytes)
- Memory is **not compressible** — if a container exceeds its memory limit, it gets **OOM killed**

### Example

```yaml
containers:
- name: app
  image: my-app:v2
  resources:
    requests:
      cpu: "100m"      # Guaranteed 0.1 CPU core
      memory: "128Mi"  # Guaranteed 128 MiB
    limits:
      cpu: "500m"      # Can burst up to 0.5 CPU core
      memory: "256Mi"  # Killed if exceeds 256 MiB
```

---

## What to Remember for the Exam

1. **Pod is the smallest deployable unit**. You cannot deploy a bare container in Kubernetes — it must be in a Pod.

2. **Containers in a Pod share**: the same IP address, localhost network, and can share volumes. They are co-scheduled on the same node.

3. **One container per Pod** is the most common pattern. Multi-container Pods are for tightly coupled containers (sidecar, ambassador, adapter patterns).

4. **Init containers** run sequentially before app containers. They must complete successfully. Used for setup tasks and dependency checks.

5. **Pod phases**: Pending, Running, Succeeded, Failed, Unknown. A Pod in Pending might be waiting for scheduling, image pull, or init containers.

6. **Restart policies**: Always (default, used by Deployments), OnFailure (used by Jobs), Never.

7. **CrashLoopBackOff** means a container keeps crashing and Kubernetes is applying exponential backoff before restarting.

8. **Resource requests** are used by the scheduler to find a suitable node. **Resource limits** are enforced by the kubelet (throttle CPU, OOM-kill for memory).

9. **CPU is compressible** (throttled at limit), **memory is incompressible** (killed at limit).

10. **Pods are ephemeral** — they are not rescheduled to another node if their node fails. Higher-level controllers (Deployments, ReplicaSets) handle that.

11. **Labels** on Pods are how Services, Deployments, and other controllers select and manage them.
