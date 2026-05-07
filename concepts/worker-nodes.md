# Worker Nodes

**Worker nodes are the machines in a Kubernetes cluster that run your application workloads, each hosting a kubelet agent, kube-proxy, and a container runtime to manage and network Pods.**

---

## Table of Contents

- [Overview](#overview)
- [kubelet](#kubelet)
- [kube-proxy](#kube-proxy)
- [Container Runtime](#container-runtime)
- [How a Pod Gets Scheduled and Started](#how-a-pod-gets-scheduled-and-started)
- [Node Status and Conditions](#node-status-and-conditions)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## Overview

```
+================================================================+
|                        WORKER NODE                             |
|                                                                |
|  +-------------------+  +----------------+  +--------------+  |
|  |     kubelet        |  |   kube-proxy   |  |  Container   |  |
|  |                    |  |                |  |  Runtime     |  |
|  |  - Pod lifecycle   |  | - Service      |  |  (containerd)|  |
|  |  - Health checks   |  |   routing      |  |              |  |
|  |  - Node register   |  | - iptables/    |  | - CRI iface  |  |
|  |  - Status reports   |  |   IPVS rules   |  | - Pull imgs  |  |
|  +-------------------+  +----------------+  | - Run cntrs  |  |
|                                              +--------------+  |
|                                                                |
|  +--------------------+  +--------------------+                |
|  |      Pod A         |  |      Pod B         |                |
|  |  +---------+       |  |  +---------+       |                |
|  |  | cntr-1  |       |  |  | cntr-1  |       |                |
|  |  +---------+       |  |  +---------+       |                |
|  |  | volumes |       |  |  | volumes |       |                |
|  +--------------------+  +--------------------+                |
+================================================================+
```

---

## kubelet

The kubelet is the **primary node agent** that runs on every worker node. It ensures that containers described in Pod specs are running and healthy.

### Responsibilities

1. **Node Registration** — Registers the node with the API server when it starts
2. **Pod Lifecycle Management** — Watches the API server for Pods assigned to its node, then creates/updates/deletes containers accordingly
3. **Health Monitoring** — Runs liveness, readiness, and startup probes to check container health
4. **Status Reporting** — Reports node status and Pod status back to the API server
5. **Resource Management** — Manages CPU, memory, and storage resources on the node
6. **Volume Mounting** — Mounts volumes specified in Pod specs
7. **Container Image Management** — Works with the container runtime to pull images

### How kubelet Manages Pods

```
  API Server
      |
      |  Watch: "What Pods are assigned to this node?"
      v
  +----------+
  |  kubelet  |
  +----+-----+
       |
       |  For each Pod:
       |
       +---> Create sandbox (pause container)
       +---> Pull container image(s)
       +---> Create and start container(s)
       +---> Set up volumes
       +---> Configure networking
       +---> Run health probes
       +---> Report status back to API server
```

### Health Probes

The kubelet runs three types of probes to monitor container health:

| Probe Type       | Purpose                                    | On Failure                    |
|------------------|--------------------------------------------|-------------------------------|
| **Liveness**     | Is the container alive?                    | Restart the container         |
| **Readiness**    | Is the container ready to serve traffic?   | Remove from Service endpoints |
| **Startup**      | Has the container started successfully?    | Keep checking (delay others)  |

#### Probe Mechanisms

```
  +------- httpGet --------+   GET /healthz on port 8080
  |                        |   Success: 200-399 status code
  +------------------------+

  +------- tcpSocket ------+   Open TCP connection to port 3306
  |                        |   Success: port is open
  +------------------------+

  +------- exec -----------+   Run command inside container
  |                        |   Success: exit code 0
  +------------------------+

  +------- grpc -----------+   Send gRPC health check
  |                        |   Success: SERVING status
  +------------------------+
```

### kubelet Configuration

- Runs as a **systemd service** on the node (not as a Pod)
- Configuration file: `/var/lib/kubelet/config.yaml`
- Kubeconfig: `/etc/kubernetes/kubelet.conf`
- Communicates with API server over HTTPS using client certificates

---

## kube-proxy

kube-proxy runs on every node and maintains **network rules** that allow Pods to communicate with Services.

### What It Does

- Watches the API server for Service and Endpoint changes
- Programs networking rules so that traffic to a Service ClusterIP reaches the correct backend Pods
- Handles load balancing across Pod replicas behind a Service

### Modes of Operation

#### 1. iptables Mode (Default)

```
  Client Pod                     kube-proxy
      |                              |
      | Traffic to Service IP        | Watches API server for
      | (10.96.0.10:80)              | Service/Endpoint changes
      v                              v
  +-------+                   Programs iptables rules
  | iptables rules on node |
  +-------+
      |
      |  DNAT (Destination NAT)
      |  Randomly selects a backend Pod
      |
      +----> Pod-1 (10.244.1.5:8080)
      +----> Pod-2 (10.244.2.8:8080)   <-- randomly chosen
      +----> Pod-3 (10.244.3.2:8080)
```

- Uses kernel-level iptables rules for packet forwarding
- **Random selection** among backends (no true round-robin)
- Cannot detect if a Pod is unresponsive (relies on readiness probes)
- Lower overhead for small numbers of Services

#### 2. IPVS Mode

```
  Client Pod
      |
      | Traffic to Service IP
      | (10.96.0.10:80)
      v
  +-------+
  | IPVS virtual server |
  +-------+
      |
      |  Multiple load balancing algorithms available:
      |  - Round Robin (rr)
      |  - Least Connections (lc)
      |  - Destination Hashing (dh)
      |  - Source Hashing (sh)
      |  - Shortest Expected Delay (sed)
      |
      +----> Pod-1 (10.244.1.5:8080)
      +----> Pod-2 (10.244.2.8:8080)
      +----> Pod-3 (10.244.3.2:8080)
```

- Uses kernel IPVS (IP Virtual Server) module
- **Better performance** for large numbers of Services (O(1) vs O(n) for iptables)
- Supports **multiple load balancing algorithms**
- More efficient with thousands of Services/Endpoints

#### Comparison Table

| Feature           | iptables (default)        | IPVS                          |
|-------------------|---------------------------|-------------------------------|
| Performance       | Good for < 1000 Services | Better for large clusters     |
| Load Balancing    | Random                   | Round-robin, least conn, etc. |
| Complexity        | Simpler                  | Requires IPVS kernel module   |
| Scalability       | O(n) rule lookup         | O(1) lookup via hash table    |
| Default           | Yes                      | No                            |

---

## Container Runtime

The container runtime is the software responsible for **pulling images and running containers** on the node.

### Container Runtime Interface (CRI)

Kubernetes does not run containers directly. It communicates with the container runtime through the **Container Runtime Interface (CRI)**, a standardized gRPC API.

```
  +----------+          CRI (gRPC)          +-----------------+
  |  kubelet  | --------------------------> | Container       |
  |           |                             | Runtime         |
  |           |  - PullImage()              | (containerd,    |
  |           |  - CreateContainer()        |  CRI-O)         |
  |           |  - StartContainer()         |                 |
  |           |  - StopContainer()          |                 |
  |           |  - RemoveContainer()        |                 |
  +----------+                             +-----------------+
                                                   |
                                                   v
                                            +-----------------+
                                            | OCI Runtime     |
                                            | (runc)          |
                                            |                 |
                                            | Actually creates|
                                            | Linux processes |
                                            | with cgroups &  |
                                            | namespaces      |
                                            +-----------------+
```

### Common Container Runtimes

| Runtime        | Description                                                  |
|----------------|--------------------------------------------------------------|
| **containerd** | Industry standard, default for most distros, CNCF graduated |
| **CRI-O**      | Lightweight, designed specifically for Kubernetes            |

> **Note**: Docker (dockershim) was removed as a runtime in Kubernetes v1.24. However, images built with Docker still work because they follow the OCI (Open Container Initiative) image spec.

### Runtime Layers

```
  kubelet
    |
    v
  CRI (Container Runtime Interface)
    |
    v
  containerd / CRI-O        <-- High-level runtime
    |
    v
  runc                       <-- Low-level OCI runtime
    |
    v
  Linux kernel               <-- cgroups + namespaces
  (actual isolation)
```

---

## How a Pod Gets Scheduled and Started

This is the complete end-to-end flow from creating a Pod to it running on a worker node:

```
  Step 1: User submits Pod spec
  =============================
  kubectl apply -f pod.yaml
         |
         v

  Step 2: API Server processes request
  =====================================
  +------------------+
  |  kube-apiserver   |
  |                   |
  |  1. Authenticate  |
  |  2. Authorize     |
  |  3. Admit         |
  |  4. Validate      |
  |  5. Store in etcd |
  +--------+---------+
           |
           v

  Step 3: Scheduler assigns a Node
  ==================================
  +------------------+
  |  kube-scheduler   |
  |                   |
  |  1. Watch sees    |
  |     unscheduled   |
  |     Pod           |
  |  2. Filter nodes  |
  |  3. Score nodes   |
  |  4. Pick best     |
  |  5. Write binding |
  |     (nodeName)    |
  +--------+---------+
           |
           v

  Step 4: kubelet creates the Pod
  =================================
  +---------------------------------------------+
  |  kubelet (on the assigned worker node)       |
  |                                              |
  |  1. Watch sees Pod assigned to this node     |
  |  2. Pull container image(s)                  |
  |  3. Create sandbox (pause container)         |
  |     - Sets up network namespace              |
  |     - Gets Pod IP from CNI plugin            |
  |  4. Create and start init containers (if any)|
  |     - Wait for each to complete              |
  |  5. Create and start app containers          |
  |  6. Set up volumes and mounts                |
  |  7. Start health probes                      |
  |  8. Report Pod status = Running              |
  +---------------------------------------------+
           |
           v

  Step 5: Pod is Running
  ========================
  +--------------------+
  |      Pod           |
  |  Status: Running   |
  |  IP: 10.244.1.5    |
  |  Node: worker-1    |
  |                    |
  |  +----------+     |
  |  | container |     |
  |  | (nginx)   |     |
  |  +----------+     |
  +--------------------+

  Step 6: Ongoing Management
  ============================
  kubelet continuously:
    - Runs liveness probes (restart if fails)
    - Runs readiness probes (remove from Service if fails)
    - Reports container status changes
    - Manages resource limits (cgroups)
    - Watches for Pod spec updates
```

---

## Node Status and Conditions

When you run `kubectl describe node`, you see several status fields:

### Node Conditions

| Condition         | Meaning                                          |
|-------------------|--------------------------------------------------|
| Ready             | Node is healthy and ready to accept Pods         |
| MemoryPressure    | Node is running low on memory                    |
| DiskPressure      | Node is running low on disk space                |
| PIDPressure       | Too many processes on the node                   |
| NetworkUnavailable| Node networking is not configured correctly      |

### Node Capacity vs Allocatable

```
  +------------------------------------------+
  |           Node Total Capacity            |
  |                                          |
  |  +------------------------------------+  |
  |  |    System Reserved (OS, kubelet)   |  |
  |  +------------------------------------+  |
  |  +------------------------------------+  |
  |  |    Kube Reserved (k8s daemons)     |  |
  |  +------------------------------------+  |
  |  +------------------------------------+  |
  |  |    Eviction Threshold              |  |
  |  +------------------------------------+  |
  |  +------------------------------------+  |
  |  |    ALLOCATABLE                     |  |
  |  |    (available for Pods)            |  |
  |  +------------------------------------+  |
  +------------------------------------------+
```

---

## What to Remember for the Exam

1. **kubelet** is the primary agent on every worker node. It manages Pod lifecycle, runs health probes, and reports status. It is NOT a Pod — it runs as a system service.

2. **Three types of health probes**: Liveness (restart on fail), Readiness (remove from Service on fail), Startup (give slow containers time to start).

3. **Probe mechanisms**: httpGet, tcpSocket, exec command, gRPC.

4. **kube-proxy** maintains network rules for Services on each node. It operates in either iptables mode (default) or IPVS mode.

5. **iptables vs IPVS**: iptables is default and simpler; IPVS is better for large clusters with many Services and supports multiple load balancing algorithms.

6. **CRI (Container Runtime Interface)** is the standard API between kubelet and the container runtime. containerd and CRI-O are the common runtimes.

7. **Docker removal**: Dockershim was removed in Kubernetes 1.24, but Docker-built images still work because they use the OCI image format.

8. **Pod startup order**: sandbox (pause container) first, then init containers (sequentially), then app containers (in parallel).

9. **Node conditions**: Ready, MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable. A node must be Ready to accept Pods.

10. **kubelet communicates with the API server** over HTTPS using client certificates. It watches for Pods assigned to its node and reports status back.
