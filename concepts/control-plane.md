# Control Plane Components

**The control plane is the brain of a Kubernetes cluster, consisting of the API server, etcd, scheduler, and controller manager — all working together to maintain the desired state of your workloads.**

---

## Table of Contents

- [Overview](#overview)
- [kube-apiserver](#kube-apiserver)
- [etcd](#etcd)
- [kube-scheduler](#kube-scheduler)
- [kube-controller-manager](#kube-controller-manager)
- [cloud-controller-manager](#cloud-controller-manager)
- [Request Flow Diagrams](#request-flow-diagrams)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## Overview

```
+================================================================+
|                     CONTROL PLANE                              |
|                                                                |
|  +------------------+       +----------------------------+     |
|  |  kube-apiserver   | <---> |           etcd             |     |
|  |  (Central Hub)    |       |   (Cluster State Store)    |     |
|  +--------+---------+       +----------------------------+     |
|           |                                                    |
|     +-----+------+                                             |
|     |            |                                             |
|     v            v                                             |
|  +----------+ +---------------------------+                    |
|  |  kube-   | | kube-controller-manager   |                    |
|  | scheduler| |  (Reconciliation Loops)   |                    |
|  +----------+ +---------------------------+                    |
|                                                                |
|  +----------------------------+                                |
|  | cloud-controller-manager   |  (optional, cloud only)       |
|  +----------------------------+                                |
+================================================================+
```

---

## kube-apiserver

The API server is the **front door** to the Kubernetes cluster. Every interaction — from `kubectl` commands to internal component communication — goes through it.

### What It Does

- Exposes a **RESTful API** over HTTPS (default port 6443)
- **Validates** incoming requests (is this a well-formed Pod spec?)
- **Authenticates** who is making the request (certificates, tokens, OIDC)
- **Authorizes** whether that user/service can perform the action (RBAC)
- Runs **Admission Controllers** that can mutate or validate requests
- **Persists** the resource to etcd after all checks pass
- **Notifies** watchers (scheduler, controllers, kubelets) of changes

### Request Processing Pipeline

```
  Incoming Request (kubectl, kubelet, controller, etc.)
         |
         v
  +------+-------+
  | Authentication |   Who are you?
  | (AuthN)        |   (certs, tokens, OIDC)
  +------+-------+
         |
         v
  +------+-------+
  | Authorization  |   Are you allowed to do this?
  | (AuthZ - RBAC) |   (roles, rolebindings)
  +------+-------+
         |
         v
  +------+-----------+
  | Admission Control |   Should we allow/modify this?
  | (Mutating +       |   (defaults, webhooks, policies)
  |  Validating)      |
  +------+-----------+
         |
         v
  +------+-------+
  | Validation     |   Is this a valid resource spec?
  +------+-------+
         |
         v
  +------+-------+
  | Persist to     |   Write to etcd
  | etcd           |
  +------+-------+
         |
         v
  +------+-------+
  | Notify         |   Alert watchers (scheduler,
  | Watchers       |   controllers, kubelets)
  +------+-------+
```

### Key Facts About the API Server

- It is the **only** component that talks to etcd directly
- It is **stateless** — all state lives in etcd
- It can be **horizontally scaled** by running multiple instances behind a load balancer
- It uses a **watch mechanism** (long-polling/streaming) so other components can react to changes in real time
- API requests follow standard HTTP methods: GET, POST, PUT, PATCH, DELETE

### API Server Endpoints Example

```
GET    /api/v1/namespaces/default/pods          List all pods in default namespace
GET    /api/v1/namespaces/default/pods/nginx     Get a specific pod
POST   /api/v1/namespaces/default/pods           Create a new pod
PUT    /api/v1/namespaces/default/pods/nginx     Update a pod
DELETE /api/v1/namespaces/default/pods/nginx     Delete a pod
```

---

## etcd

etcd is a **distributed, reliable key-value store** that serves as the single source of truth for all cluster data.

### What It Stores

- All Kubernetes objects (Pods, Services, Deployments, ConfigMaps, Secrets, etc.)
- Cluster configuration
- Current state of all resources
- Lease/leader election information

### How It Works

```
+------------------+       +------------------+       +------------------+
|    etcd Node 1   |<----->|    etcd Node 2   |<----->|    etcd Node 3   |
|    (Leader)      |       |    (Follower)    |       |    (Follower)    |
+------------------+       +------------------+       +------------------+
        ^
        |  Only the API server reads/writes
        |
+------------------+
|  kube-apiserver   |
+------------------+
```

### Key Characteristics

| Feature              | Description                                               |
|----------------------|-----------------------------------------------------------|
| Consistency          | Uses **Raft consensus** algorithm — strongly consistent   |
| Leader Election      | One node is leader; all writes go through the leader      |
| Quorum               | Needs majority of nodes to agree (e.g., 2 of 3)          |
| Recommended Count    | Odd number of nodes (1, 3, 5) to avoid split-brain       |
| Data Format          | Key-value pairs; keys are hierarchical (like file paths)  |
| Port                 | 2379 (client), 2380 (peer communication)                  |

### etcd Key Structure

```
/registry/pods/default/nginx
/registry/services/default/my-service
/registry/deployments/default/my-app
/registry/secrets/default/my-secret
```

### Raft Consensus

```
  Client Write Request
         |
         v
  +------+------+
  |  etcd Leader |
  +------+------+
         |
    +----+----+          Replicates to followers
    |         |          before acknowledging
    v         v
+--------+ +--------+
|Follower| |Follower|
+--------+ +--------+
         |
         v
  Write acknowledged to API server
```

For a write to succeed, a **majority (quorum)** of etcd nodes must acknowledge it. With 3 nodes, the cluster tolerates 1 failure. With 5 nodes, it tolerates 2.

---

## kube-scheduler

The scheduler watches for newly created Pods that have no node assigned and selects the best node for each Pod.

### Scheduling Process: Two Phases

```
  New Pod (no nodeName set)
         |
         v
  +------+-------+
  | PHASE 1:       |
  | FILTERING      |   Which nodes CAN run this Pod?
  | (Predicates)   |   - Enough CPU/memory?
  |                |   - Does it match nodeSelector?
  |                |   - Any taints blocking it?
  |                |   - Port conflicts?
  +------+-------+
         |
         v
  Feasible Nodes: [node-1, node-3, node-5]
         |
         v
  +------+-------+
  | PHASE 2:       |
  | SCORING        |   Which node is BEST for this Pod?
  | (Priorities)   |   - Least requested resources
  |                |   - Spreading across zones
  |                |   - Affinity/anti-affinity
  |                |   - Data locality
  +------+-------+
         |
         v
  Best Node: node-3  (highest score)
         |
         v
  +------+-------+
  | BINDING        |   Write nodeName to Pod spec
  |                |   via API server
  +------+-------+
```

### Filtering Criteria (Predicates)

- **PodFitsResources** — Does the node have enough CPU and memory?
- **PodFitsHostPorts** — Are the required host ports available?
- **NodeSelector** — Does the node match the Pod's nodeSelector labels?
- **Taints/Tolerations** — Does the Pod tolerate the node's taints?
- **PodAffinity/AntiAffinity** — Does placing here satisfy affinity rules?
- **VolumeZone** — Is the required PV in the same zone?

### Scoring Criteria (Priorities)

- **LeastRequestedPriority** — Prefer nodes with more free resources
- **BalancedResourceAllocation** — Prefer nodes where CPU/memory usage is balanced
- **InterPodAffinity** — Prefer nodes matching pod affinity preferences
- **NodeAffinity** — Score based on preferred node affinity rules
- **ImageLocality** — Prefer nodes that already have the container image

### What If No Node Fits?

The Pod stays in **Pending** state until a suitable node becomes available (e.g., node scaled up, resources freed, taint removed).

---

## kube-controller-manager

The controller manager runs a collection of **controller loops** that watch the cluster state and work to move the actual state towards the desired state.

### The Reconciliation Loop

```
  +---------------------------------------------+
  |           RECONCILIATION LOOP                |
  |                                              |
  |   +-----------+       +-----------+          |
  |   | Desired   |       | Actual    |          |
  |   | State     | -?->  | State     |          |
  |   | (spec)    |       | (status)  |          |
  |   +-----------+       +-----------+          |
  |        |                    |                |
  |        +--------+-----------+                |
  |                 |                            |
  |                 v                            |
  |        +--------+--------+                   |
  |        | Are they equal?  |                  |
  |        +--------+--------+                   |
  |          YES |    | NO                       |
  |              |    v                          |
  |         Do   | Take corrective action        |
  |        nothing| (create/delete/update)        |
  |              |    |                          |
  |              +----+                          |
  |                 |                            |
  |            Wait & Repeat                     |
  +---------------------------------------------+
```

### Key Controllers

| Controller              | What It Reconciles                                          |
|--------------------------|-------------------------------------------------------------|
| **ReplicaSet Controller**| Ensures the desired number of Pod replicas are running      |
| **Deployment Controller**| Manages ReplicaSets for rolling updates and rollbacks       |
| **Node Controller**      | Monitors node health; marks nodes as NotReady               |
| **Endpoint Controller**  | Populates Endpoints objects (links Services to Pods)        |
| **Job Controller**       | Creates Pods for Jobs; tracks completions                   |
| **ServiceAccount Controller** | Creates default ServiceAccounts in new namespaces     |
| **Namespace Controller** | Cleans up resources when a namespace is deleted             |
| **PV Binder Controller** | Binds PersistentVolumeClaims to PersistentVolumes           |

### Example: ReplicaSet Controller in Action

```
  Desired: 3 Pods          Actual: 2 Pods Running
       |                          |
       +----------+---------------+
                  |
                  v
       Controller sees: 3 desired, 2 actual
       Action: Create 1 more Pod
                  |
                  v
       POST /api/v1/namespaces/default/pods  -->  API Server  -->  etcd
                  |
                  v
       Scheduler assigns new Pod to a node
                  |
                  v
       Kubelet starts the container
                  |
                  v
       Actual: 3 Pods Running  ==  Desired: 3 Pods
```

---

## cloud-controller-manager

The cloud-controller-manager separates cloud-specific logic from core Kubernetes logic. It runs only when Kubernetes is deployed on a cloud provider (AWS, GCP, Azure, etc.).

### Controllers It Runs

| Controller        | What It Does                                              |
|--------------------|-----------------------------------------------------------|
| Node Controller    | Checks with cloud API if a node still exists              |
| Route Controller   | Sets up network routes in the cloud infrastructure        |
| Service Controller | Creates cloud load balancers for LoadBalancer Services    |

### Why It Exists

```
  Before (monolithic):
  +-------------------------------------------+
  | kube-controller-manager                   |
  |  - ReplicaSet controller                  |
  |  - Node controller (+ cloud logic)        |
  |  - Service controller (+ cloud logic)     |
  |  - Route controller (+ cloud logic)       |
  +-------------------------------------------+

  After (separated):
  +----------------------------+   +----------------------------+
  | kube-controller-manager    |   | cloud-controller-manager   |
  |  - ReplicaSet controller   |   |  - Node controller (cloud) |
  |  - Node controller (core)  |   |  - Service controller      |
  |  - Endpoint controller     |   |  - Route controller        |
  +----------------------------+   +----------------------------+
```

This separation allows cloud providers to develop at their own pace without modifying core Kubernetes code.

---

## Request Flow Diagrams

### Creating a Deployment (Full Flow)

```
  kubectl apply -f deployment.yaml
         |
         v
  (1) kube-apiserver
      - Authenticates the user
      - Authorizes via RBAC
      - Validates the Deployment spec
      - Stores Deployment in etcd
      - Notifies watchers
         |
         v
  (2) Deployment Controller (in kube-controller-manager)
      - Sees new Deployment
      - Creates a ReplicaSet
      - Writes ReplicaSet to API server --> etcd
         |
         v
  (3) ReplicaSet Controller
      - Sees new ReplicaSet with desired=3
      - Creates 3 Pod objects
      - Writes Pods to API server --> etcd
         |
         v
  (4) kube-scheduler
      - Sees 3 Pods with no nodeName
      - Runs filtering + scoring for each
      - Assigns each Pod to a node (writes binding)
         |
         v
  (5) kubelet (on each assigned node)
      - Sees Pod assigned to its node
      - Pulls container image
      - Starts container via container runtime
      - Reports Pod status back to API server
         |
         v
  (6) Pod is Running!
```

### Deleting a Pod (What Happens Behind the Scenes)

```
  kubectl delete pod nginx
         |
         v
  (1) API server marks Pod for deletion (sets deletionTimestamp)
         |
         v
  (2) kubelet sees the update
      - Sends SIGTERM to container
      - Waits for graceful shutdown (default 30s)
      - Sends SIGKILL if still running
      - Reports Pod as terminated
         |
         v
  (3) API server removes Pod from etcd
         |
         v
  (4) ReplicaSet Controller notices actual < desired
      - Creates a new Pod to replace it
```

---

## What to Remember for the Exam

1. **kube-apiserver** is the central communication hub. All components communicate through it. It is the only component that directly accesses etcd.

2. **etcd** stores all cluster state. It uses the Raft consensus algorithm, needs an odd number of nodes for quorum, and a majority must agree for writes to succeed.

3. **kube-scheduler** uses a two-phase process: **filtering** (which nodes can run this Pod?) then **scoring** (which node is best?). If no node fits, the Pod stays Pending.

4. **kube-controller-manager** runs multiple reconciliation loops. Each controller watches for a specific resource type and ensures actual state matches desired state.

5. **cloud-controller-manager** is optional and only needed when running on a cloud provider. It handles cloud-specific operations like load balancers and node lifecycle.

6. **Request processing order**: Authentication --> Authorization (RBAC) --> Admission Control --> Validation --> Persist to etcd.

7. All control plane components are **stateless except etcd**. They can be restarted without data loss because state is in etcd.

8. The control plane components communicate via the API server using a **watch mechanism** — they don't poll; they receive streaming updates.

9. **Controllers don't talk to each other** — they all interact independently through the API server.

10. In production, the control plane is typically run with **multiple replicas** for high availability, with etcd running as a 3- or 5-node cluster.
