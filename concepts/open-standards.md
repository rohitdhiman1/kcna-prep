# Open Standards in Cloud Native

> Open standards enable interoperability, prevent vendor lock-in, and allow components to be swapped without rewriting applications.

---

## Why Open Standards Matter

```
  Without Open Standards              With Open Standards
  ─────────────────────               ───────────────────

  ┌─────────┐                        ┌─────────┐
  │ App     │                        │ App     │
  │         │                        │         │
  │ Tied to │                        │ Uses    │
  │ Vendor  │                        │ standard│
  │ A's API │                        │ API     │
  └────┬────┘                        └────┬────┘
       │                                  │
       ▼                                  ▼
  ┌─────────┐                     ┌──────────────┐
  │ Vendor  │                     │   Standard   │
  │ A only  │                     │  Interface   │
  │         │                     └──┬───┬───┬───┘
  │ Locked  │                        │   │   │
  │ in!     │                        ▼   ▼   ▼
  └─────────┘                     ┌──┐ ┌──┐ ┌──┐
                                  │A │ │B │ │C │  Swap freely
                                  └──┘ └──┘ └──┘
```

**Key benefits:**
- **Vendor neutrality**: Not locked into a single provider.
- **Interoperability**: Components from different vendors work together.
- **Innovation**: Multiple implementations compete, driving improvement.
- **Portability**: Move workloads between clouds or on-premises.
- **Community-driven**: Standards evolve based on real-world needs.

---

## Where Standards Sit in the Kubernetes Stack

```
  ┌─────────────────────────────────────────────────────────────┐
  │                     Kubernetes Cluster                       │
  │                                                               │
  │  ┌───────────────────────────────────────────────────────┐   │
  │  │                   API Server                          │   │
  │  └────┬──────────────┬──────────────┬────────────────────┘   │
  │       │              │              │                         │
  │       ▼              ▼              ▼                         │
  │  ┌─────────┐   ┌──────────┐   ┌──────────┐                  │
  │  │   CPI   │   │ Scheduler│   │Controller│                  │
  │  │ Cloud   │   │          │   │ Manager  │                  │
  │  │Provider │   └──────────┘   └──────────┘                  │
  │  │Interface│                                                  │
  │  └─────────┘                                                  │
  │                                                               │
  │  ┌───────────────────────────────────────────────────────┐   │
  │  │                    Kubelet (Node)                      │   │
  │  │                                                         │   │
  │  │  ┌──────────┐    ┌──────────┐    ┌──────────┐         │   │
  │  │  │   CRI    │    │   CNI    │    │   CSI    │         │   │
  │  │  │Container │    │Container │    │Container │         │   │
  │  │  │ Runtime  │    │ Network  │    │ Storage  │         │   │
  │  │  │Interface │    │Interface │    │Interface │         │   │
  │  │  └────┬─────┘    └────┬─────┘    └────┬─────┘         │   │
  │  │       │              │              │                   │   │
  │  │       ▼              ▼              ▼                   │   │
  │  │  ┌────────┐    ┌────────┐    ┌────────┐               │   │
  │  │  │contain-│    │ Calico │    │  Ceph  │               │   │
  │  │  │  erd   │    │ Cilium │    │  EBS   │               │   │
  │  │  │ CRI-O  │    │Flannel │    │  NFS   │               │   │
  │  │  │  etc.  │    │ etc.   │    │  etc.  │               │   │
  │  │  └────────┘    └────────┘    └────────┘               │   │
  │  │                                                         │   │
  │  └───────────────────────────────────────────────────────┘   │
  │                                                               │
  │  ┌───────────────────────────────────────────────────────┐   │
  │  │               Service Mesh Layer                      │   │
  │  │                                                         │   │
  │  │  ┌──────────┐    ┌──────────────────┐                 │   │
  │  │  │   SMI    │    │    OCI Images    │                 │   │
  │  │  │ Service  │    │ (image spec,     │                 │   │
  │  │  │ Mesh     │    │  runtime spec,   │                 │   │
  │  │  │Interface │    │  distribution)   │                 │   │
  │  │  └────┬─────┘    └────────┬─────────┘                 │   │
  │  │       │                   │                            │   │
  │  │       ▼                   ▼                            │   │
  │  │  ┌────────┐         ┌────────┐                        │   │
  │  │  │ Istio  │         │Docker  │                        │   │
  │  │  │Linkerd │         │Hub,    │                        │   │
  │  │  │Consul  │         │Harbor, │                        │   │
  │  │  │ etc.   │         │ECR,etc.│                        │   │
  │  │  └────────┘         └────────┘                        │   │
  │  │                                                         │   │
  │  └───────────────────────────────────────────────────────┘   │
  │                                                               │
  └─────────────────────────────────────────────────────────────┘
```

---

## OCI (Open Container Initiative)

The **OCI** is a Linux Foundation project that defines open standards for containers.

### Three Specifications

| Spec | What It Standardizes | Why It Matters |
|------|---------------------|----------------|
| **Image Spec** | Format of container images (layers, manifest, config) | Any OCI-compliant image runs on any OCI-compliant runtime. Build with Docker, run with containerd or CRI-O. |
| **Runtime Spec** | How to run a container (filesystem bundle, lifecycle, environment) | Any OCI-compliant runtime (runc, crun, gVisor, Kata) can run an OCI image. |
| **Distribution Spec** | How to push and pull images from registries | Any OCI-compliant registry (Docker Hub, Harbor, GitHub Container Registry, ECR) can store and serve images. |

### Key Points
- **runc** is the reference implementation of the OCI runtime spec.
- Docker images and OCI images are compatible.
- OCI ensures you are **not locked into Docker** or any single container tool.
- Before OCI, container image formats and runtimes were Docker-specific.

---

## CRI (Container Runtime Interface)

The **CRI** is a Kubernetes-specific plugin API that allows kubelet to use different container runtimes without code changes.

### What It Standardizes
- How kubelet communicates with container runtimes (via gRPC).
- Operations: pull images, create/start/stop/delete containers, get container status.

### Implementations

| Runtime | Description |
|---------|-------------|
| **containerd** | Industry standard, used by Docker under the hood. CNCF Graduated. |
| **CRI-O** | Lightweight runtime built specifically for Kubernetes. |
| **Docker** | Was the original, but Kubernetes removed direct Docker support (dockershim) in v1.24. Docker images still work via containerd. |

```
  Before CRI                          With CRI
  ──────────                          ────────

  kubelet ──► dockershim ──► Docker   kubelet ──► CRI ──┬──► containerd
                                                         ├──► CRI-O
                                                         └──► (any CRI impl)
  Tightly coupled to Docker           Pluggable runtime
```

### Why CRI Matters
- Kubernetes is **not tied to Docker** or any single runtime.
- New runtimes (e.g., for security or performance) can be added without changing Kubernetes.
- **dockershim removal** (Kubernetes 1.24) was possible because CRI exists.

---

## CNI (Container Network Interface)

The **CNI** is a specification for configuring container networking.

### What It Standardizes
- How network plugins set up networking for containers/pods.
- Adding a container to a network and removing it.
- IP address management (IPAM).

### Implementations

| Plugin | Key Feature |
|--------|-------------|
| **Calico** | Network policies, BGP routing, widely used |
| **Cilium** | eBPF-based, high performance, advanced security. CNCF Graduated. |
| **Flannel** | Simple overlay network, easy to set up |
| **Weave Net** | Mesh networking, encryption |
| **AWS VPC CNI** | Uses native AWS VPC networking |
| **Azure CNI** | Uses native Azure networking |

### Why CNI Matters
- Kubernetes does **not implement networking itself** — it delegates to CNI plugins.
- Different environments need different networking (overlay, native, eBPF).
- CNI allows choosing the right network solution for each use case.

---

## CSI (Container Storage Interface)

The **CSI** is a standard for exposing block and file storage to containerized workloads.

### What It Standardizes
- How Kubernetes communicates with storage providers.
- Operations: create/delete volumes, attach/detach, mount/unmount, snapshot.

### Implementations

| Provider | Storage Type |
|----------|-------------|
| **AWS EBS CSI** | Block storage (EBS volumes) |
| **GCP PD CSI** | Persistent Disks |
| **Azure Disk CSI** | Azure Managed Disks |
| **Ceph/Rook** | Distributed storage |
| **NFS CSI** | Network file system |
| **Portworx** | Enterprise container storage |
| **Longhorn** | Lightweight distributed storage (CNCF Incubating) |

```
  Without CSI                         With CSI
  ───────────                         ────────

  Kubernetes ──► in-tree volume       Kubernetes ──► CSI ──┬──► AWS EBS
                 plugins (hard-                             ├──► GCP PD
                 coded in k8s)                              ├──► Ceph
                                                            ├──► NFS
  Hard to add new storage;                                  └──► any CSI
  requires K8s code changes                                      driver

                                      New storage = new driver,
                                      no K8s changes needed
```

### Why CSI Matters
- Storage providers can develop drivers **independently** from Kubernetes releases.
- Old "in-tree" volume plugins required changes to Kubernetes core code.
- CSI is the **standard way** to add storage to Kubernetes (in-tree plugins are being migrated to CSI).

---

## SMI (Service Mesh Interface)

The **SMI** is a specification for a common interface to service meshes on Kubernetes.

### What It Standardizes

| API | Purpose |
|-----|---------|
| **Traffic Access Control** | Define access policies (which services can talk to which) |
| **Traffic Specs** | Describe traffic (routes, headers, methods) |
| **Traffic Split** | Control traffic distribution (canary, A/B testing) |
| **Traffic Metrics** | Expose standard metrics for service-to-service communication |

### Implementations
- Istio, Linkerd, Consul Connect, Open Service Mesh (OSM).

### Why SMI Matters
- Without SMI, each service mesh has its own configuration format.
- SMI allows writing mesh-agnostic configuration.
- Applications do not need to be rewritten when switching meshes.

**Note:** SMI has seen varying levels of adoption. The Gateway API (from SIG Network) is increasingly used for some of the same traffic management use cases.

---

## CPI (Cloud Provider Interface)

The **CPI**, more commonly referred to as the **Cloud Controller Manager** interface, standardizes how Kubernetes integrates with cloud providers.

### What It Standardizes
- **Node management**: Register/deregister nodes, check health, get node metadata.
- **Route management**: Configure routes so pods on different nodes can communicate.
- **Load balancer management**: Provision cloud load balancers for Service type=LoadBalancer.

### Why CPI Matters
- Cloud-specific code was moved **out of the Kubernetes core** into external cloud controller managers.
- Each cloud provider maintains their own controller manager.
- Enables Kubernetes to run on **any cloud** without core code changes.
- AWS, GCP, Azure, and other providers each have their own Cloud Controller Manager.

```
  Old approach                        New approach (CPI)
  ────────────                        ──────────────────

  ┌──────────────────────┐           ┌──────────────────┐
  │ kube-controller-     │           │ kube-controller-  │
  │   manager            │           │   manager         │
  │                      │           │  (cloud-agnostic) │
  │ Contains AWS,        │           └────────┬──────────┘
  │ GCP, Azure code      │                    │
  │ (monolithic)         │           ┌────────┴──────────┐
  └──────────────────────┘           │  Cloud Controller │
                                     │  Manager          │
  Hard to maintain;                  │  (per provider)   │
  all clouds in one binary           └──────────────────┘

                                     Clean separation;
                                     each provider maintains
                                     their own component
```

---

## Standards Summary Table

| Standard | Full Name | What It Standardizes | Layer |
|----------|-----------|---------------------|-------|
| **OCI** | Open Container Initiative | Container images, runtimes, distribution | Container |
| **CRI** | Container Runtime Interface | Kubelet-to-runtime communication | Runtime |
| **CNI** | Container Network Interface | Pod networking setup | Network |
| **CSI** | Container Storage Interface | Volume provisioning and management | Storage |
| **SMI** | Service Mesh Interface | Service mesh configuration | Service Mesh |
| **CPI** | Cloud Provider Interface | Cloud integration (nodes, LBs, routes) | Cloud |

---

## What to Remember for the Exam

1. **Open standards prevent vendor lock-in** and enable interoperability.
2. **OCI** has three specs: **image, runtime, distribution**. runc is the reference runtime. OCI ensures container portability.
3. **CRI** lets Kubernetes use different container runtimes (containerd, CRI-O). This is why Kubernetes does not depend on Docker.
4. **CNI** standardizes pod networking. Kubernetes delegates networking to CNI plugins (Calico, Cilium, Flannel).
5. **CSI** standardizes storage. Storage vendors provide CSI drivers instead of in-tree plugins.
6. **SMI** standardizes service mesh configuration (traffic splitting, access control, metrics).
7. **CPI** (Cloud Controller Manager) separates cloud-specific code from Kubernetes core.
8. Each standard follows the **plugin pattern**: define an interface, let multiple implementations compete.
9. These standards are why Kubernetes is **portable and extensible** across clouds and environments.
10. Know which standard corresponds to which layer: **CRI=runtime, CNI=network, CSI=storage, OCI=container format**.
