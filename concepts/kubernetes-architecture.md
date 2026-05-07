# Kubernetes Architecture

**Kubernetes is a container orchestration platform with a master/worker architecture where a control plane manages the desired state of workloads running across a fleet of worker nodes.**

---

## Table of Contents

- [High-Level Architecture](#high-level-architecture)
- [Control Plane vs Worker Nodes](#control-plane-vs-worker-nodes)
- [Complete Architecture Diagram](#complete-architecture-diagram)
- [Communication Patterns](#communication-patterns)
- [Key Concepts](#key-concepts)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## High-Level Architecture

A Kubernetes cluster consists of two main types of machines:

1. **Control Plane Nodes** (formerly called "masters") — Run the brains of the cluster
2. **Worker Nodes** — Run the actual application workloads (Pods)

```
+---------------------------------------------------------------------+
|                        KUBERNETES CLUSTER                           |
|                                                                     |
|  +-----------------------------+  +------------------------------+  |
|  |      CONTROL PLANE          |  |        WORKER NODES          |  |
|  |                             |  |                              |  |
|  |  Manages desired state      |  |  Run application workloads  |  |
|  |  Makes scheduling decisions |  |  Execute containers         |  |
|  |  Stores cluster data        |  |  Report status back          |  |
|  +-----------------------------+  +------------------------------+  |
+---------------------------------------------------------------------+
```

---

## Control Plane vs Worker Nodes

### Control Plane Components

| Component                  | Role                                              |
|----------------------------|----------------------------------------------------|
| kube-apiserver             | Central API gateway; all communication goes here   |
| etcd                       | Distributed key-value store for all cluster data   |
| kube-scheduler             | Assigns Pods to Nodes based on resource needs      |
| kube-controller-manager    | Runs reconciliation loops to maintain desired state|
| cloud-controller-manager   | Integrates with cloud provider APIs (optional)     |

### Worker Node Components

| Component        | Role                                                     |
|------------------|----------------------------------------------------------|
| kubelet          | Agent on each node; manages Pod lifecycle                |
| kube-proxy       | Handles networking rules for Service traffic             |
| Container Runtime| Runs containers (containerd, CRI-O)                     |

---

## Complete Architecture Diagram

```
+===========================================================================+
||                        KUBERNETES CLUSTER                               ||
||                                                                         ||
||  +-------------------------------------------------------------------+  ||
||  |                      CONTROL PLANE NODE(S)                        |  ||
||  |                                                                   |  ||
||  |  +---------------------+         +---------------------------+   |  ||
||  |  |   kube-apiserver    |<------->|          etcd             |   |  ||
||  |  |                     |         |  (key-value store)        |   |  ||
||  |  |  - REST API         |         |  - cluster state          |   |  ||
||  |  |  - AuthN / AuthZ    |         |  - configs, secrets       |   |  ||
||  |  |  - Admission Ctrl   |         |  - leader election data   |   |  ||
||  |  +----------+----------+         +---------------------------+   |  ||
||  |             |                                                    |  ||
||  |             | watches & updates                                  |  ||
||  |    +--------+--------+                                           |  ||
||  |    |                 |                                           |  ||
||  |    v                 v                                           |  ||
||  |  +-------------+  +-------------------------+                    |  ||
||  |  |   kube-     |  | kube-controller-manager |                    |  ||
||  |  |  scheduler  |  |                         |                    |  ||
||  |  |             |  | - ReplicaSet controller |                    |  ||
||  |  | - filtering |  | - Node controller       |                    |  ||
||  |  | - scoring   |  | - Endpoint controller   |                    |  ||
||  |  | - binding   |  | - SA & Token controller |                    |  ||
||  |  +-------------+  +-------------------------+                    |  ||
||  |                                                                   |  ||
||  |  +---------------------------+                                    |  ||
||  |  | cloud-controller-manager |  (if running on cloud)             |  ||
||  |  | - Node lifecycle         |                                    |  ||
||  |  | - Route controller       |                                    |  ||
||  |  | - Service (LB) controller|                                    |  ||
||  |  +---------------------------+                                    |  ||
||  +-------------------------------------------------------------------+  ||
||                              |                                          ||
||                    HTTPS (port 6443)                                     ||
||                              |                                          ||
||  +-------------------------------------------------------------------+  ||
||  |                       WORKER NODE 1                               |  ||
||  |                                                                   |  ||
||  |  +-------------------+  +----------------+  +-----------------+  |  ||
||  |  |     kubelet       |  |   kube-proxy   |  | Container       |  |  ||
||  |  |                   |  |                |  | Runtime         |  |  ||
||  |  | - Pod lifecycle   |  | - iptables     |  | (containerd)    |  |  ||
||  |  | - Node registr.   |  |   or IPVS      |  |                 |  |  ||
||  |  | - Health checks   |  | - Service      |  | - Pulls images  |  |  ||
||  |  | - Reports status  |  |   routing      |  | - Runs cntrs    |  |  ||
||  |  +-------------------+  +----------------+  +-----------------+  |  ||
||  |                                                                   |  ||
||  |  +--------------------+ +--------------------+                    |  ||
||  |  |    Pod A           | |    Pod B           |                    |  ||
||  |  | +-----+ +-------+ | | +-------+          |                    |  ||
||  |  | |cntr1| |cntr2  | | | |cntr1  |          |                    |  ||
||  |  | +-----+ +-------+ | | +-------+          |                    |  ||
||  |  +--------------------+ +--------------------+                    |  ||
||  +-------------------------------------------------------------------+  ||
||                                                                         ||
||  +-------------------------------------------------------------------+  ||
||  |                       WORKER NODE 2                               |  ||
||  |                                                                   |  ||
||  |  +-------------------+  +----------------+  +-----------------+  |  ||
||  |  |     kubelet       |  |   kube-proxy   |  | Container       |  |  ||
||  |  |                   |  |                |  | Runtime         |  |  ||
||  |  +-------------------+  +----------------+  +-----------------+  |  ||
||  |                                                                   |  ||
||  |  +--------------------+ +--------------------+                    |  ||
||  |  |    Pod C           | |    Pod D           |                    |  ||
||  |  | +-------+         | | +-------+          |                    |  ||
||  |  | |cntr1  |         | | |cntr1  |          |                    |  ||
||  |  | +-------+         | | +-------+          |                    |  ||
||  |  +--------------------+ +--------------------+                    |  ||
||  +-------------------------------------------------------------------+  ||
+===========================================================================+
```

---

## Communication Patterns

### 1. kubectl to API Server

```
  User
   |
   | kubectl apply -f deployment.yaml
   v
+------------------+
|  kube-apiserver   |  <-- All external communication enters here
+------------------+
```

All `kubectl` commands go to the API server over HTTPS (typically port 6443).

### 2. API Server to etcd

```
+------------------+         +--------+
|  kube-apiserver   | ------> |  etcd  |
+------------------+  write  +--------+
                      read
```

**Only the API server talks to etcd directly.** No other component communicates with etcd.

### 3. Scheduler to API Server

```
+------------------+         +------------------+
|  kube-scheduler   | <-----> |  kube-apiserver   |
+------------------+  watch  +------------------+
                      for unscheduled pods
                      then write binding
```

The scheduler watches for new Pods with no `nodeName` set, then writes a binding to assign the Pod to a node.

### 4. Kubelet to API Server

```
+------------------+         +------------------+
|     kubelet       | <-----> |  kube-apiserver   |
+------------------+  report +------------------+
                      node status
                      watch for assigned pods
```

The kubelet watches the API server for Pods assigned to its node, then manages their lifecycle.

### 5. Full Request Flow: Deploying a Pod

```
Step 1:  kubectl apply -f pod.yaml
              |
              v
Step 2:  kube-apiserver (validates, stores in etcd)
              |
              v
Step 3:  kube-scheduler (watches API, sees unscheduled Pod)
              |
              v
Step 4:  kube-scheduler writes binding (Pod -> Node)
              |
              v
Step 5:  kubelet on target Node (watches API, sees new Pod assigned)
              |
              v
Step 6:  kubelet tells Container Runtime to pull image & start container
              |
              v
Step 7:  Container is running! kubelet reports status back to API server
```

---

## Key Concepts

### Declarative Model

Kubernetes uses a **declarative model**: you describe the **desired state** (e.g., "I want 3 replicas of nginx"), and the system continuously works to make the **actual state** match.

```
  Desired State          Reconciliation Loop         Actual State
+----------------+     +---------------------+     +----------------+
| 3 nginx Pods   | --> | Controller compares  | --> | 3 nginx Pods   |
| running        |     | desired vs actual    |     | running        |
+----------------+     | and takes action     |     +----------------+
                        +---------------------+
```

### Everything Talks Through the API Server

```
                    +------------------+
                    |  kube-apiserver   |
                    +--------+---------+
                             |
         +-------------------+-------------------+
         |          |           |          |      |
         v          v           v          v      v
      etcd     scheduler   controllers  kubelet  kube-proxy
```

The API server is the **single point of communication** in the cluster. No component talks directly to another — everything goes through the API server.

### Cluster Networking

- Every Pod gets its own IP address
- Pods can communicate with each other across nodes without NAT
- Services provide stable endpoints for a set of Pods
- kube-proxy configures networking rules on each node

---

## What to Remember for the Exam

1. **Two types of nodes**: Control plane nodes run cluster management components; worker nodes run application Pods.

2. **API server is the central hub**: All communication flows through the kube-apiserver. It is the only component that talks to etcd.

3. **etcd stores all cluster state**: It is the single source of truth for the entire cluster.

4. **Scheduler assigns Pods to Nodes**: It uses filtering (which nodes can run this Pod?) and scoring (which node is the best fit?).

5. **Controller Manager runs reconciliation loops**: Controllers constantly compare desired state to actual state and take corrective action.

6. **kubelet runs on every worker node**: It manages Pod lifecycle and reports node status.

7. **kube-proxy handles Service networking**: It programs iptables or IPVS rules for routing traffic to Pods.

8. **Declarative model**: Kubernetes works by reconciling desired state vs actual state, not by executing imperative commands.

9. **Container Runtime Interface (CRI)**: Kubernetes does not run containers directly — it delegates to a CRI-compliant runtime like containerd.

10. **Cloud Controller Manager**: Optional component that integrates Kubernetes with cloud provider APIs (load balancers, routes, node lifecycle).
