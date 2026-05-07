# Containers

**Containers are lightweight, isolated processes that package an application with its dependencies. Unlike VMs, containers share the host OS kernel, making them faster to start and more resource-efficient. Kubernetes orchestrates containers using container runtimes that implement the Container Runtime Interface (CRI).**

---

## Table of Contents

- [Containers vs Virtual Machines](#containers-vs-virtual-machines)
- [Container Images](#container-images)
- [Dockerfile Basics](#dockerfile-basics)
- [Image Pull Policies](#image-pull-policies)
- [Container Runtimes](#container-runtimes)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## Containers vs Virtual Machines

### Key Differences

| Aspect               | Containers                        | Virtual Machines                   |
|----------------------|-----------------------------------|------------------------------------|
| **Isolation**        | Process-level (shared kernel)     | Hardware-level (separate kernel)   |
| **OS**               | Shares host OS kernel             | Each VM has its own OS             |
| **Size**             | Megabytes (lightweight)           | Gigabytes (includes full OS)       |
| **Startup time**     | Seconds                           | Minutes                            |
| **Resource usage**   | Low overhead                      | High overhead (hypervisor + OS)    |
| **Density**          | 100s per host                     | 10s per host                       |
| **Portability**      | Run anywhere with container runtime| Requires compatible hypervisor    |
| **Security**         | Weaker isolation (shared kernel)  | Stronger isolation (separate kernel)|

### Architecture Comparison

```
  VIRTUAL MACHINES                         CONTAINERS
  ==================                       ==========

  +----------+ +----------+               +---------+ +---------+ +---------+
  |  App A   | |  App B   |               |  App A  | |  App B  | |  App C  |
  +----------+ +----------+               +---------+ +---------+ +---------+
  +----------+ +----------+               |  Bins/  | |  Bins/  | |  Bins/  |
  |  Bins/   | |  Bins/   |               |  Libs   | |  Libs   | |  Libs   |
  |  Libs    | |  Libs    |               +---------+ +---------+ +---------+
  +----------+ +----------+               +---------------------------------------+
  +----------+ +----------+               |         Container Runtime             |
  | Guest OS | | Guest OS |               |       (containerd, CRI-O)             |
  | (Linux)  | | (Windows)|               +---------------------------------------+
  +----------+ +----------+               +---------------------------------------+
  +-----------------------------+         |            Host OS (Linux)             |
  |        Hypervisor            |         +---------------------------------------+
  |   (VMware, KVM, Hyper-V)    |         +---------------------------------------+
  +-----------------------------+         |         Hardware / Infrastructure       |
  +-----------------------------+         +---------------------------------------+
  |     Host OS / Hardware       |
  +-----------------------------+

  Key difference:
  - VMs virtualize HARDWARE (each VM has its own OS kernel)
  - Containers virtualize the OS (share the host kernel)
```

### How Container Isolation Works

Containers use Linux kernel features for isolation:

```
  +------------------------------------------------+
  |               HOST LINUX KERNEL                  |
  |                                                  |
  |  Namespaces (what a container can SEE):          |
  |    - PID namespace:    own process tree          |
  |    - Network namespace: own IP, ports, routes    |
  |    - Mount namespace:  own filesystem view       |
  |    - UTS namespace:    own hostname              |
  |    - IPC namespace:    own shared memory         |
  |    - User namespace:   own user/group IDs        |
  |                                                  |
  |  cgroups (what a container can USE):             |
  |    - CPU limits                                  |
  |    - Memory limits                               |
  |    - I/O limits                                  |
  |    - Process count limits                        |
  +------------------------------------------------+
```

---

## Container Images

A container image is a read-only template that contains everything needed to run an application: code, runtime, libraries, and configuration.

### Image Layers

Images are built in **layers**. Each instruction in a Dockerfile creates a new layer. Layers are cached and shared between images.

```
  Image: my-app:v2
  =================

  +-----------------------------------+
  |  Layer 5: CMD ["python", "app.py"]|  (metadata, no filesystem change)
  +-----------------------------------+
  |  Layer 4: COPY app.py /app/       |  ~5 KB (application code)
  +-----------------------------------+
  |  Layer 3: RUN pip install flask   |  ~15 MB (dependencies)
  +-----------------------------------+
  |  Layer 2: RUN apt-get update ...  |  ~30 MB (OS packages)
  +-----------------------------------+
  |  Layer 1: FROM python:3.11-slim   |  ~120 MB (base image)
  +-----------------------------------+

  - Layers are READ-ONLY and cached
  - When a container runs, a thin WRITABLE layer is added on top
  - Shared layers are stored only once on disk (saves space)
```

### Image Naming and References

```
  Full image reference:
  registry.example.com/namespace/repository:tag

  Examples:
  docker.io/library/nginx:1.25        (Docker Hub official image)
  nginx:1.25                           (shorthand for above)
  gcr.io/my-project/my-app:v2         (Google Container Registry)
  my-app:latest                        (default registry, latest tag)

  Using digest (immutable reference):
  nginx@sha256:abc123def456...         (exact image, cannot change)
```

### Tags vs Digests

| Aspect          | Tags                              | Digests                              |
|-----------------|-----------------------------------|--------------------------------------|
| **Format**      | `nginx:1.25`                     | `nginx@sha256:abc123...`            |
| **Mutable**     | Yes -- same tag can point to different images | No -- always the same image |
| **Use case**    | Development, convenience          | Production, reproducibility          |
| **Risk**        | `latest` tag can change anytime   | None -- always gets exact image     |
| **Best practice**| Use specific version tags        | Use digests for critical workloads  |

```
  Tag "latest" danger:
  =====================

  Monday:    nginx:latest ---> Image A (nginx 1.24)
  Wednesday: nginx:latest ---> Image B (nginx 1.25)  <-- tag moved!

  Your Pod spec says "nginx:latest" but you might get
  a different image depending on when it was pulled.

  Best practice: Use specific tags (nginx:1.25) or digests.
```

### Container Registries

A registry stores and distributes container images.

| Registry                  | Description                                    |
|---------------------------|------------------------------------------------|
| Docker Hub                | Default public registry, official images        |
| Google Container Registry (GCR) | Google Cloud, `gcr.io`                  |
| Amazon ECR                | AWS, `<account>.dkr.ecr.<region>.amazonaws.com`|
| Azure Container Registry  | Azure, `<name>.azurecr.io`                     |
| GitHub Container Registry | GitHub, `ghcr.io`                              |
| Quay.io                   | Red Hat, CNCF projects often use this          |
| Harbor                    | CNCF open-source registry (self-hosted)        |

---

## Dockerfile Basics

A Dockerfile is a text file with instructions to build a container image. Each instruction creates a layer.

### Key Instructions

```dockerfile
# Base image -- every Dockerfile starts with FROM
FROM python:3.11-slim

# Set working directory inside the container
WORKDIR /app

# Copy files from build context to image
COPY requirements.txt .
COPY app.py .

# Run commands during image BUILD (creates new layer)
RUN pip install --no-cache-dir -r requirements.txt

# Set environment variables
ENV FLASK_APP=app.py
ENV FLASK_ENV=production

# Expose a port (documentation only, does not publish)
EXPOSE 5000

# Default command when container STARTS
CMD ["python", "app.py"]
```

### Instruction Reference

| Instruction    | Purpose                                              | When Runs      |
|----------------|------------------------------------------------------|----------------|
| `FROM`         | Set the base image                                   | Build time     |
| `WORKDIR`      | Set the working directory                            | Build time     |
| `COPY`         | Copy files from build context into image             | Build time     |
| `ADD`          | Like COPY but also handles URLs and tar extraction   | Build time     |
| `RUN`          | Execute a command (install packages, etc.)           | Build time     |
| `ENV`          | Set environment variable                             | Build + runtime|
| `EXPOSE`       | Document which port the app listens on               | Documentation  |
| `CMD`          | Default command to run when container starts         | Runtime        |
| `ENTRYPOINT`   | Fixed command that always runs (CMD becomes args)    | Runtime        |
| `ARG`          | Build-time variable (not available at runtime)       | Build time     |
| `VOLUME`       | Create a mount point for external storage            | Runtime        |
| `USER`         | Set the user to run subsequent commands              | Build + runtime|

### CMD vs ENTRYPOINT

```
  CMD ["python", "app.py"]
  ========================
  - Defines the DEFAULT command
  - Can be OVERRIDDEN at runtime:
    docker run my-app python other.py    <-- replaces CMD

  ENTRYPOINT ["python"]
  CMD ["app.py"]
  =====================
  - ENTRYPOINT is the FIXED executable
  - CMD provides DEFAULT arguments to ENTRYPOINT
  - CMD can be overridden, ENTRYPOINT stays:
    docker run my-app other.py
    => runs: python other.py

  Together they form: ENTRYPOINT + CMD = full command
    python app.py (default)
    python other.py (CMD overridden)
```

### Multi-Stage Builds

```dockerfile
# Stage 1: Build
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Stage 2: Run (minimal image)
FROM alpine:3.18
COPY --from=builder /app/myapp /usr/local/bin/
CMD ["myapp"]
```

Multi-stage builds produce smaller final images by discarding build tools and intermediate files.

---

## Image Pull Policies

The `imagePullPolicy` field controls when Kubernetes pulls a container image from the registry.

```yaml
containers:
- name: app
  image: my-app:v2
  imagePullPolicy: IfNotPresent    # Pull only if not already on the node
```

### Policies

| Policy           | Behavior                                                  |
|------------------|-----------------------------------------------------------|
| `Always`         | Always pull the image from the registry, even if cached  |
| `IfNotPresent`   | Pull only if the image is not already on the node        |
| `Never`          | Never pull -- image must already exist on the node       |

### Default Behavior

```
  What happens if you don't set imagePullPolicy?
  ================================================

  Image tag is "latest" or no tag specified:
    nginx            --> imagePullPolicy defaults to "Always"
    nginx:latest     --> imagePullPolicy defaults to "Always"

  Image tag is a specific version:
    nginx:1.25       --> imagePullPolicy defaults to "IfNotPresent"

  Image uses a digest:
    nginx@sha256:... --> imagePullPolicy defaults to "IfNotPresent"
```

### Private Registries

To pull from a private registry, you need an `imagePullSecret`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: registry.example.com/my-app:v2
  imagePullSecrets:
  - name: registry-credentials      # Secret of type kubernetes.io/dockerconfigjson
```

---

## Container Runtimes

A container runtime is the software that actually runs containers on a node. Kubernetes does not run containers directly -- it uses the **Container Runtime Interface (CRI)** to communicate with the runtime.

### Architecture

```
  +-----------------------------------------------------------+
  |                     KUBERNETES NODE                         |
  |                                                             |
  |  +--------+                                                 |
  |  | kubelet | --- CRI (gRPC) ---> +---------------------+   |
  |  +--------+                      | Container Runtime    |   |
  |                                  | (containerd, CRI-O) |   |
  |                                  +----------+----------+   |
  |                                             |               |
  |                                  +----------+----------+   |
  |                                  |  Low-Level Runtime   |   |
  |                                  |  (runc, crun, kata)  |   |
  |                                  +---------------------+   |
  |                                             |               |
  |                                  +----------+----------+   |
  |                                  |   Linux Kernel        |   |
  |                                  |  (namespaces, cgroups)|   |
  |                                  +---------------------+   |
  +-----------------------------------------------------------+
```

### Runtime Landscape

| Runtime          | Type        | Description                                    |
|------------------|-------------|------------------------------------------------|
| **containerd**   | High-level  | Industry standard, graduated CNCF project. Default for most Kubernetes distributions. |
| **CRI-O**        | High-level  | Lightweight, designed specifically for Kubernetes. Used by OpenShift. |
| **runc**         | Low-level   | OCI reference implementation. Actually creates and runs containers. Used by containerd and CRI-O. |
| **crun**         | Low-level   | Written in C, faster alternative to runc       |
| **Kata Containers** | Low-level | Runs containers in lightweight VMs for stronger isolation |
| **gVisor**       | Low-level   | Provides a user-space kernel for container isolation |

### Key Concepts

```
  High-Level vs Low-Level Runtimes
  ==================================

  High-Level (containerd, CRI-O):
    - Manages container lifecycle (create, start, stop, delete)
    - Handles image pulling and storage
    - Implements the CRI for Kubernetes
    - Delegates actual container creation to low-level runtime

  Low-Level (runc, kata):
    - Actually creates the container process
    - Sets up namespaces and cgroups
    - Follows the OCI Runtime Specification
```

### Docker and Kubernetes

```
  Historical timeline:
  ====================

  Before K8s 1.24:
    kubelet ---> dockershim ---> Docker Engine ---> containerd ---> runc
                 (built into     (extra layer,
                  kubelet)        overhead)

  After K8s 1.24 (dockershim removed):
    kubelet ---> CRI ---> containerd ---> runc
                          (direct, no Docker needed)

  Docker images still work! The OCI image format is the standard.
  You just don't need Docker Engine installed on Kubernetes nodes.
```

### OCI Standards

The **Open Container Initiative (OCI)** defines standards for containers:

| Standard                | What It Defines                                    |
|-------------------------|----------------------------------------------------|
| OCI Image Specification | Format for container images (layers, config)       |
| OCI Runtime Specification| How to run a container (filesystem, namespaces)  |
| OCI Distribution Specification | How to distribute images (registry API)   |

---

## What to Remember for the Exam

1. **Containers share the host OS kernel**; VMs each have their own OS. Containers are lighter (MBs vs GBs) and start in seconds (vs minutes for VMs).

2. **Container isolation** uses Linux namespaces (what you can see) and cgroups (what you can use). This is weaker isolation than VMs.

3. **Images are built in layers**. Each Dockerfile instruction creates a layer. Layers are cached and shared, saving disk space and build time.

4. **Tags are mutable** (can point to different images over time). **Digests are immutable** (always reference the exact same image). Avoid using `latest` in production.

5. **FROM** sets the base image. **RUN** executes commands at build time. **COPY** copies files into the image. **CMD** sets the default runtime command. **ENTRYPOINT** sets a fixed executable (CMD becomes arguments to it).

6. **Image pull policies**: `Always` (always pull), `IfNotPresent` (pull if not cached), `Never` (must be pre-loaded). Default is `Always` for `:latest` tag and `IfNotPresent` for specific tags.

7. **containerd** is the most widely used container runtime in Kubernetes. **CRI-O** is the alternative, commonly used with OpenShift.

8. **Docker was removed** from Kubernetes in v1.24 (dockershim removal). Docker images still work because they follow the OCI standard. You just use containerd or CRI-O directly.

9. **CRI (Container Runtime Interface)** is the standard API between kubelet and the container runtime. Any CRI-compliant runtime works with Kubernetes.

10. **OCI (Open Container Initiative)** defines standards for image format and runtime behavior. This is why Docker-built images work with containerd, CRI-O, and any OCI-compliant runtime.

11. **Multi-stage builds** use multiple FROM statements to produce smaller final images by discarding build-time dependencies.

12. **Private registries** require an `imagePullSecret` (of type `kubernetes.io/dockerconfigjson`) to authenticate when pulling images.
