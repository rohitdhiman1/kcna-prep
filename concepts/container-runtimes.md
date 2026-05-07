# Container Runtimes

> The container runtime is the software that actually runs containers. Kubernetes uses the Container Runtime Interface (CRI) to talk to any compatible runtime.

---

## The Container Runtime Stack

Kubernetes does not run containers directly. It delegates to a container runtime through a well-defined interface. There are multiple layers involved:

```
  Container Runtime Stack

  ┌──────────────────────────────────────────────────────┐
  │                     kubelet                          │
  │           (node agent, part of Kubernetes)            │
  └───────────────────────┬──────────────────────────────┘
                          │ CRI (gRPC API)
                          ▼
  ┌──────────────────────────────────────────────────────┐
  │              High-Level Runtime                      │
  │         (containerd  OR  CRI-O)                      │
  │                                                      │
  │  Responsibilities:                                   │
  │  - Image pulling and storage                         │
  │  - Container lifecycle management                    │
  │  - Snapshot management                               │
  │  - Networking setup (calls CNI)                      │
  └───────────────────────┬──────────────────────────────┘
                          │ OCI Runtime Spec
                          ▼
  ┌──────────────────────────────────────────────────────┐
  │              Low-Level Runtime                       │
  │             (runc  OR  crun)                         │
  │                                                      │
  │  Responsibilities:                                   │
  │  - Create Linux namespaces                           │
  │  - Set up cgroups                                    │
  │  - Execute the container process                     │
  └───────────────────────┬──────────────────────────────┘
                          │
                          ▼
  ┌──────────────────────────────────────────────────────┐
  │               Linux Kernel                           │
  │     (namespaces, cgroups, seccomp, AppArmor)         │
  └──────────────────────────────────────────────────────┘
```

---

## Container Runtime Interface (CRI)

CRI is a **gRPC-based API** that defines how the kubelet communicates with the container runtime. It was introduced to decouple Kubernetes from any specific runtime.

### What CRI Defines

CRI has two main services:

```
  CRI API Structure

  ┌─────────────────────────────────────────────┐
  │                   CRI                        │
  │                                              │
  │  ┌───────────────────┐  ┌────────────────┐  │
  │  │ RuntimeService    │  │ ImageService   │  │
  │  │                   │  │                │  │
  │  │ - RunPodSandbox   │  │ - PullImage    │  │
  │  │ - StopPodSandbox  │  │ - ListImages   │  │
  │  │ - CreateContainer │  │ - RemoveImage  │  │
  │  │ - StartContainer  │  │ - ImageStatus  │  │
  │  │ - StopContainer   │  │                │  │
  │  │ - RemoveContainer │  │                │  │
  │  │ - ListContainers  │  │                │  │
  │  │ - ExecSync        │  │                │  │
  │  └───────────────────┘  └────────────────┘  │
  │                                              │
  └─────────────────────────────────────────────┘
```

### Why CRI Matters

Before CRI, Docker was hardcoded into Kubernetes (the "dockershim"). CRI made the runtime **pluggable**:

```
  Evolution of Container Runtimes in Kubernetes

  Before CRI (K8s < 1.5):
    kubelet ──(hardcoded)──► Docker ──► containerd ──► runc

  With CRI (K8s 1.5+):
    kubelet ──(CRI)──► containerd ──► runc
    kubelet ──(CRI)──► CRI-O ──► runc

  Docker removed (K8s 1.24+):
    kubelet ──(CRI)──► containerd ──► runc    (most common)
    kubelet ──(CRI)──► CRI-O ──► runc         (OpenShift default)
```

**Docker was removed from Kubernetes in v1.24** (dockershim removal). This does not mean Docker images stopped working — Docker images are OCI-compliant and work with any CRI runtime.

---

## High-Level Runtimes

### containerd

- Originally part of Docker, now a **standalone CNCF graduated project**.
- The **most widely used** container runtime in Kubernetes.
- Manages the full container lifecycle: image pull, container creation, networking, storage.
- Uses runc (or another OCI runtime) under the hood to actually create containers.
- Used by: GKE, EKS, AKS, kind, k3s, most managed Kubernetes services.
- CLI tool: `ctr` (low-level), `nerdctl` (Docker-compatible).

### CRI-O

- Purpose-built for Kubernetes — does **nothing more** than what CRI requires.
- Lightweight alternative to containerd.
- **Default runtime for Red Hat OpenShift**.
- Also uses runc under the hood.
- Tightly follows Kubernetes release cycles.
- CLI tool: `crictl` (same tool works with containerd too).

```
  containerd vs CRI-O

  ┌──────────────────────┐    ┌──────────────────────┐
  │     containerd       │    │       CRI-O          │
  │                      │    │                      │
  │ - General purpose    │    │ - Kubernetes-only    │
  │ - CNCF Graduated     │    │ - CNCF Incubating   │
  │ - Supports non-K8s   │    │ - Minimal footprint  │
  │   use cases too      │    │ - Tracks K8s releases│
  │ - Used by most       │    │ - Used by OpenShift  │
  │   managed K8s        │    │                      │
  │ - Plugin system      │    │ - No plugin system   │
  └──────────────────────┘    └──────────────────────┘
```

---

## Low-Level Runtimes

### runc

- The **reference implementation** of the OCI Runtime Specification.
- Written in Go.
- Does the actual work of creating containers using Linux kernel features:
  - **Namespaces** — process, network, mount, user, IPC, UTS isolation
  - **Cgroups** — CPU, memory, I/O resource limits
  - **Seccomp** — system call filtering
  - **Capabilities** — fine-grained root privilege control

### crun

- An alternative OCI runtime written in **C** (faster, lower memory).
- Fully compatible with runc.
- Drop-in replacement — containerd and CRI-O can use either.
- Popular in resource-constrained environments.

---

## OCI — Open Container Initiative

The OCI defines two key specifications:

### 1. OCI Runtime Specification

Defines **how to run a container**:
- The filesystem bundle (rootfs + config.json)
- Lifecycle operations: create, start, kill, delete
- Linux-specific: namespaces, cgroups, mounts

### 2. OCI Image Specification

Defines **how container images are structured**:
- Image manifest (layers, config, media type)
- Image index (multi-architecture support)
- Layer format (tar+gzip)

### Why OCI Matters

```
  OCI ensures interoperability:

  Build with:          Store in:           Run with:
  ┌──────────┐        ┌──────────┐        ┌──────────┐
  │ Docker   │        │ Docker   │        │containerd│
  │ Podman   │──OCI──►│  Hub     │──OCI──►│ CRI-O   │
  │ Buildah  │ Image  │ Harbor   │ Image  │ Podman   │
  │ kaniko   │  Spec  │ ECR/GCR  │  Spec  │          │
  └──────────┘        └──────────┘        └──────────┘

  An image built with Docker runs on containerd or CRI-O.
  An image built with Buildah runs on Docker or containerd.
  OCI makes this possible.
```

---

## How kubelet Talks to the Runtime

Here is the full flow when Kubernetes needs to start a pod:

```
  Pod Creation Flow

  1. API Server receives pod spec
                │
                ▼
  2. Scheduler assigns pod to a node
                │
                ▼
  3. kubelet on that node picks up the assignment
                │
                ▼
  4. kubelet calls CRI RuntimeService.RunPodSandbox()
     ──► creates the pod's network namespace (calls CNI)
                │
                ▼
  5. kubelet calls CRI ImageService.PullImage()
     ──► pulls container image if not cached
                │
                ▼
  6. kubelet calls CRI RuntimeService.CreateContainer()
     ──► high-level runtime (containerd) prepares the container
                │
                ▼
  7. containerd invokes runc to create the container
     ──► runc sets up namespaces, cgroups, mounts
     ──► runc starts the container process
                │
                ▼
  8. Container is running. kubelet monitors it via CRI.
```

---

## Runtime Detection

You can check which runtime your cluster uses:

```bash
# Check node runtime
kubectl get nodes -o wide
# Look at the CONTAINER-RUNTIME column

# Check kubelet configuration
kubectl get node <node-name> -o jsonpath='{.status.nodeInfo.containerRuntimeVersion}'
# Output example: containerd://1.7.2

# Using crictl (works with any CRI runtime)
crictl info
crictl version
```

---

## Key Exam Points

- **CRI** is the standard API between kubelet and the container runtime (gRPC-based).
- **containerd** is the most common high-level runtime (CNCF Graduated).
- **CRI-O** is a lightweight, Kubernetes-only alternative (default in OpenShift).
- **runc** is the reference low-level OCI runtime. **crun** is a faster C alternative.
- **OCI** defines two specs: Runtime Spec (how to run) and Image Spec (how to package).
- Docker images work everywhere because they follow the OCI Image Spec.
- **dockershim was removed in Kubernetes 1.24** — Kubernetes no longer supports Docker as a runtime directly.
- The stack is: kubelet --> CRI --> containerd/CRI-O --> runc/crun --> Linux kernel.

---

## What to Remember for the Exam

1. **CRI is the interface** — it decouples Kubernetes from any specific runtime. Know the name "Container Runtime Interface."
2. **Two layers of runtime** — high-level (containerd, CRI-O) handles image management and lifecycle; low-level (runc, crun) creates the actual container using Linux kernel primitives.
3. **containerd vs CRI-O** — containerd is general-purpose and most popular; CRI-O is Kubernetes-specific and used by OpenShift.
4. **OCI has two specs** — Runtime Spec and Image Spec. These ensure images built anywhere run anywhere.
5. **Docker removal** — Kubernetes 1.24 removed dockershim. Docker-built images still work because they are OCI-compliant. The runtime changed, not the image format.
6. **runc** uses namespaces (isolation) and cgroups (resource limits) from the Linux kernel.
