# KCNA Practice Exam 2

> 60 multiple-choice questions | 90 minutes | Passing score: 75% (45/60)
>
> Questions are weighted to match the real exam domains.

---

## How to Use

1. Set a 90-minute timer.
2. Answer all 60 questions without looking at notes.
3. Check your answers against the [Answer Key](#answer-key) at the bottom.
4. Review explanations for anything you got wrong — each answer references the relevant concept file.

---

## Domain 1 — Kubernetes Fundamentals (Questions 1–28)

**Q1.** Which control plane component watches for newly created Pods with no assigned node and selects a node for them to run on?

- A) kube-controller-manager
- B) kube-apiserver
- C) kube-scheduler
- D) etcd

**Q2.** What happens if etcd becomes unavailable in a Kubernetes cluster?

- A) Existing Pods continue running but no new changes can be made to cluster state
- B) All Pods are immediately terminated
- C) The cluster automatically switches to a backup database
- D) Worker nodes restart themselves

**Q3.** Which component in the control plane runs reconciliation loops such as the ReplicaSet controller and the Node controller?

- A) kube-scheduler
- B) kubelet
- C) kube-apiserver
- D) kube-controller-manager

**Q4.** What format does Kubernetes use to store data in etcd?

- A) SQL tables
- B) JSON documents in a document store
- C) Key-value pairs
- D) Protocol buffer files on disk

**Q5.** A developer runs `kubectl apply -f deployment.yaml`. Which component receives this request first?

- A) etcd
- B) kube-scheduler
- C) kube-apiserver
- D) kubelet

**Q6.** What is the function of the container runtime on a worker node?

- A) Assigning IP addresses to Pods
- B) Pulling images and running containers
- C) Routing Service traffic via iptables
- D) Reporting cluster state to etcd

**Q7.** A Deployment is updated with a new container image. What does Kubernetes create to manage the rollout?

- A) A new Deployment resource
- B) A new ReplicaSet with the updated Pod template
- C) A new DaemonSet
- D) A new Service

**Q8.** What command can you use to undo the last Deployment rollout?

- A) kubectl delete deployment
- B) kubectl rollout undo deployment/<name>
- C) kubectl restart deployment/<name>
- D) kubectl apply --revert

**Q9.** Which Service type creates a cloud provider load balancer that routes external traffic to the Service?

- A) ClusterIP
- B) NodePort
- C) ExternalName
- D) LoadBalancer

**Q10.** An ExternalName Service is used to:

- A) Expose a Service on each node's IP at a static port
- B) Map a Service to a DNS name (CNAME) instead of a set of Pods
- C) Create an external load balancer
- D) Assign a public IP to a Pod

**Q11.** Which resources are NOT namespaced in Kubernetes?

- A) Pods and Deployments
- B) ConfigMaps and Secrets
- C) Nodes and PersistentVolumes
- D) Services and Ingresses

**Q12.** A label selector `app=frontend,tier=web` will match resources that have:

- A) Either the label `app=frontend` or `tier=web`
- B) Both labels `app=frontend` and `tier=web`
- C) Any label starting with `app` or `tier`
- D) Resources in the `frontend` namespace

**Q13.** How can a Pod consume a ConfigMap?

- A) Only as environment variables
- B) Only as mounted volumes
- C) As environment variables, mounted volumes, or command-line arguments
- D) ConfigMaps cannot be consumed by Pods directly

**Q14.** A DaemonSet is updated. How does the update roll out by default?

- A) All Pods are replaced simultaneously
- B) Pods are updated one node at a time in a rolling fashion
- C) A new DaemonSet is created alongside the old one
- D) The DaemonSet must be deleted and recreated

**Q15.** Which workload resource would you use to run a database like MySQL that requires stable storage and ordered startup?

- A) Deployment
- B) DaemonSet
- C) StatefulSet
- D) Job

**Q16.** What is the `completions` field in a Job spec used for?

- A) Setting the number of retries for failed Pods
- B) Specifying how many Pods must successfully complete before the Job is done
- C) Limiting the CPU time for each Pod
- D) Defining the number of parallel Pods

**Q17.** A CronJob with the schedule `*/5 * * * *` will run:

- A) Every 5 hours
- B) Every 5 minutes
- C) At 5:00 AM every day
- D) On the 5th of every month

**Q18.** A Pod has a toleration that matches a node's taint with effect `NoSchedule`. What happens?

- A) The Pod is evicted from the node
- B) The Pod is allowed to be scheduled on that node
- C) The node is drained of all Pods
- D) The Pod runs with reduced resource limits

**Q19.** What is the difference between `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution` in node affinity?

- A) There is no difference
- B) `required` is a hard constraint (must be met); `preferred` is a soft constraint (best effort)
- C) `required` only applies to DaemonSets; `preferred` applies to Deployments
- D) `preferred` takes precedence over `required`

**Q20.** What happens when a container exceeds its CPU limit?

- A) The container is OOM-killed
- B) The container is throttled (CPU time is restricted)
- C) The node is cordoned
- D) The Pod is evicted from the cluster

**Q21.** Which of the following is true about container images?

- A) Each image layer is mutable and can be edited in place
- B) Images are built from a Dockerfile using layers; layers are immutable and shared across images
- C) Images must include a full operating system
- D) Images cannot be stored in private registries

**Q22.** What does `kubectl get pods -A` do?

- A) Gets Pods with all labels
- B) Gets Pods across all namespaces
- C) Gets all resources, not just Pods
- D) Gets Pods in alphabetical order

**Q23.** In a multi-container Pod, how do the containers communicate with each other?

- A) Through the Kubernetes API server
- B) Via localhost, since they share the same network namespace
- C) Through a Service resource
- D) Each container gets a unique IP; they use those IPs

**Q24.** What is the purpose of an init container?

- A) To run alongside the main container for the lifetime of the Pod
- B) To run to completion before the main containers start, performing setup tasks
- C) To handle graceful shutdown of the Pod
- D) To monitor the main container's health

**Q25.** A readiness probe differs from a liveness probe in that:

- A) A failing readiness probe causes the container to restart
- B) A failing readiness probe removes the Pod from Service endpoints without restarting it
- C) Readiness probes run only at startup
- D) Readiness probes are only available for StatefulSets

**Q26.** What is a resource limit in Kubernetes?

- A) The minimum resources guaranteed to a container
- B) The maximum resources a container is allowed to consume
- C) The total resources available on a node
- D) A quota applied at the namespace level

**Q27.** Which kubectl command shows detailed information about a specific Pod, including events?

- A) kubectl get pod <name>
- B) kubectl logs <name>
- C) kubectl describe pod <name>
- D) kubectl top pod <name>

**Q28.** What is the purpose of the `spec.selector` field in a Deployment?

- A) To select which nodes the Pods run on
- B) To match the Deployment to the Pods it manages via label selectors
- C) To choose which namespace the Deployment belongs to
- D) To filter which users can modify the Deployment

---

## Domain 2 — Container Orchestration (Questions 29–41)

**Q29.** After Kubernetes removed dockershim, which runtime became the most widely adopted CRI implementation?

- A) rkt
- B) containerd
- C) Docker Engine
- D) LXC

**Q30.** What is runc?

- A) A Kubernetes scheduler plugin
- B) A low-level OCI-compliant container runtime that creates and runs containers
- C) A container image registry
- D) A high-level container orchestration tool

**Q31.** What two specifications does the OCI define?

- A) Container Runtime Interface and Container Network Interface
- B) Image Specification and Runtime Specification
- C) Pod Specification and Service Specification
- D) Storage Specification and Security Specification

**Q32.** Which CNI plugin uses eBPF for high-performance networking and observability?

- A) Flannel
- B) Calico
- C) Cilium
- D) Weave Net

**Q33.** What does CoreDNS do in a Kubernetes cluster?

- A) Manages container images
- B) Provides DNS-based service discovery so Pods can resolve Service names to IP addresses
- C) Schedules Pods to nodes
- D) Manages TLS certificates

**Q34.** An Ingress controller is required because:

- A) Kubernetes has a built-in HTTP reverse proxy
- B) Ingress resources are only metadata; a controller (like NGINX Ingress) implements the actual routing
- C) Without it, Services cannot have ClusterIPs
- D) It replaces kube-proxy for all traffic routing

**Q35.** A NetworkPolicy with an empty `podSelector: {}` in a namespace applies to:

- A) No Pods
- B) All Pods in that namespace
- C) Only Pods with the label `network-policy=enabled`
- D) Pods across all namespaces

**Q36.** What is the difference between a Role and a ClusterRole?

- A) There is no difference
- B) A Role is scoped to a namespace; a ClusterRole applies cluster-wide or to non-namespaced resources
- C) A ClusterRole can only be used with ServiceAccounts
- D) A Role has more permissions than a ClusterRole

**Q37.** In a service mesh, what does mTLS (mutual TLS) provide?

- A) Load balancing between services
- B) Encrypted communication where both client and server verify each other's identity
- C) DNS-based service discovery
- D) Automatic container restarts

**Q38.** SMI (Service Mesh Interface) was created to:

- A) Replace Kubernetes Services
- B) Provide a standard interface for service mesh implementations, enabling portability
- C) Define container image formats
- D) Manage persistent storage

**Q39.** What is the advantage of dynamic provisioning with a StorageClass over static PV provisioning?

- A) Dynamic provisioning is more secure
- B) Storage is automatically created when a PVC is submitted, reducing manual admin work
- C) Dynamic provisioning only works with local storage
- D) Static provisioning is always preferred

**Q40.** What `accessMode` allows a PersistentVolume to be mounted as read-write by multiple nodes?

- A) ReadWriteOnce
- B) ReadOnlyMany
- C) ReadWriteMany
- D) ReadWriteOncePod

**Q41.** What is the purpose of the `reclaimPolicy` on a PersistentVolume?

- A) To set the storage capacity
- B) To define what happens to the underlying storage when the PVC is deleted (Retain, Delete, or Recycle)
- C) To specify which Pods can use the volume
- D) To encrypt data at rest

---

## Domain 3 — Cloud Native Architecture (Questions 42–51)

**Q42.** Which of the following is a cloud native design principle?

- A) Manually scale infrastructure based on weekly traffic forecasts
- B) Design for failure — assume components will fail and build resilience into the system
- C) Deploy all services as a single binary for simplicity
- D) Avoid automation to reduce complexity

**Q43.** Which 12-Factor App principle states that an application should export services via port binding?

- A) Factor VII — Port Binding
- B) Factor III — Config
- C) Factor VI — Processes
- D) Factor X — Dev/prod parity

**Q44.** What is a key disadvantage of microservices compared to monoliths?

- A) Microservices cannot be containerized
- B) Increased operational complexity from managing many services, their networking, and debugging distributed systems
- C) Microservices cannot scale independently
- D) Microservices require a single programming language

**Q45.** The Vertical Pod Autoscaler (VPA) adjusts:

- A) The number of Pod replicas
- B) The CPU and memory requests/limits of existing containers
- C) The number of nodes in the cluster
- D) The number of namespaces

**Q46.** KEDA (Kubernetes Event-Driven Autoscaling) differs from HPA in that:

- A) KEDA can only scale Deployments
- B) KEDA scales based on external event sources (queues, streams, databases) and can scale to zero
- C) KEDA replaces the Kubernetes scheduler
- D) KEDA is a CNCF Sandbox project with no production users

**Q47.** Which of the following is a Graduated CNCF project?

- A) KEDA
- B) Backstage
- C) Kubernetes
- D) OpenKruise

**Q48.** What is a SIG (Special Interest Group) in the Kubernetes community?

- A) A company that sponsors Kubernetes
- B) A group that owns and develops a specific area of the Kubernetes project (e.g., SIG-Network, SIG-Storage)
- C) A security team that handles vulnerability reports
- D) A certification body for Kubernetes administrators

**Q49.** What is Knative primarily used for?

- A) Container image scanning
- B) Running serverless workloads on Kubernetes with scale-to-zero capability
- C) Managing Helm charts
- D) Monitoring cluster health

**Q50.** The difference between an SLI and an SLO is:

- A) They are the same thing
- B) An SLI is the actual measured metric (e.g., 99.95% uptime); an SLO is the target for that metric (e.g., 99.9%)
- C) An SLO is measured; an SLI is the target
- D) SLIs are only used in contracts; SLOs are internal

**Q51.** Platform Engineering aims to:

- A) Replace DevOps entirely
- B) Provide internal developer platforms with self-service tools, reducing cognitive load on development teams
- C) Eliminate the need for Kubernetes
- D) Outsource all infrastructure management to cloud providers

---

## Domain 4 — Cloud Native Observability (Questions 52–56)

**Q52.** Which Prometheus metric type is used to track values that can go up or down, such as current memory usage?

- A) Counter
- B) Gauge
- C) Histogram
- D) Summary

**Q53.** Fluentd is a CNCF Graduated project used for:

- A) Distributed tracing
- B) Unified log collection and forwarding
- C) Container image building
- D) Autoscaling Pods

**Q54.** What is the difference between Fluentd and Fluent Bit?

- A) They are the same tool
- B) Fluent Bit is a lightweight, high-performance log forwarder; Fluentd is a more feature-rich log aggregator with a plugin ecosystem
- C) Fluentd is newer and replaces Fluent Bit
- D) Fluent Bit only works with Prometheus

**Q55.** In OpenTelemetry, what is a "span"?

- A) A time-series data point
- B) A unit of work in a distributed trace, representing an operation with a start time and duration
- C) A log entry
- D) A Kubernetes Pod lifecycle event

**Q56.** What is the primary benefit of using Alertmanager with Prometheus?

- A) It stores metrics long-term
- B) It handles alert routing, grouping, silencing, and notification (email, Slack, PagerDuty)
- C) It replaces Grafana for visualization
- D) It collects logs from Pods

---

## Domain 5 — Cloud Native Application Delivery (Questions 57–60)

**Q57.** In a canary deployment, what happens initially?

- A) All traffic is switched to the new version at once
- B) A small percentage of traffic is routed to the new version while most traffic stays on the old version
- C) The old version is deleted before the new version starts
- D) Two complete environments run simultaneously with equal traffic

**Q58.** What does `helm upgrade --install` do?

- A) Deletes and recreates the release
- B) Upgrades an existing release, or installs it if it does not exist
- C) Only installs new charts, never upgrades
- D) Rolls back to the previous version

**Q59.** In GitOps, what is the role of a reconciliation loop?

- A) It pushes code changes to the Git repository
- B) It continuously compares the desired state in Git with the actual cluster state and corrects any drift
- C) It runs unit tests on every commit
- D) It builds container images from source code

**Q60.** FluxCD differs from ArgoCD in that:

- A) FluxCD does not support GitOps
- B) FluxCD uses a pull-based model and is designed as a set of composable controllers, while ArgoCD provides a more integrated UI-driven experience
- C) ArgoCD cannot sync from Git repositories
- D) FluxCD only works with Helm charts

---

## Answer Key

> Scroll down only after completing the exam!

<br><br><br><br><br><br><br><br><br><br>

| # | Answer | Explanation | Reference |
|---|--------|-------------|-----------|
| 1 | **C** | The kube-scheduler watches for unassigned Pods and selects a node for them. | [concepts/control-plane.md](../concepts/control-plane.md) |
| 2 | **A** | Existing workloads keep running, but no state changes (new Pods, scaling, etc.) can be persisted. | [concepts/control-plane.md](../concepts/control-plane.md) |
| 3 | **D** | The kube-controller-manager runs reconciliation loops (controllers) like ReplicaSet, Node, and Endpoint controllers. | [concepts/control-plane.md](../concepts/control-plane.md) |
| 4 | **C** | etcd is a distributed key-value store. | [concepts/control-plane.md](../concepts/control-plane.md) |
| 5 | **C** | All requests (kubectl, API calls) go through the kube-apiserver as the single entry point. | [concepts/kubernetes-api.md](../concepts/kubernetes-api.md) |
| 6 | **B** | The container runtime (e.g., containerd) pulls images from registries and runs containers. | [concepts/worker-nodes.md](../concepts/worker-nodes.md) |
| 7 | **B** | A Deployment creates a new ReplicaSet for each revision of the Pod template. | [concepts/deployments.md](../concepts/deployments.md) |
| 8 | **B** | `kubectl rollout undo` reverts a Deployment to its previous ReplicaSet revision. | [concepts/deployments.md](../concepts/deployments.md) |
| 9 | **D** | LoadBalancer type provisions a cloud provider load balancer for external traffic. | [concepts/services.md](../concepts/services.md) |
| 10 | **B** | ExternalName maps a Service to an external DNS name via a CNAME record. | [concepts/services.md](../concepts/services.md) |
| 11 | **C** | Nodes and PersistentVolumes are cluster-scoped (non-namespaced) resources. | [concepts/namespaces-labels.md](../concepts/namespaces-labels.md) |
| 12 | **B** | Comma-separated selectors are ANDed — the resource must have both labels. | [concepts/namespaces-labels.md](../concepts/namespaces-labels.md) |
| 13 | **C** | ConfigMaps can be consumed as environment variables, mounted files, or command-line arguments. | [concepts/configmaps-secrets.md](../concepts/configmaps-secrets.md) |
| 14 | **B** | DaemonSet default update strategy is RollingUpdate, updating one node at a time. | [concepts/workload-resources.md](../concepts/workload-resources.md) |
| 15 | **C** | StatefulSets provide stable identities, persistent storage, and ordered operations — ideal for databases. | [concepts/workload-resources.md](../concepts/workload-resources.md) |
| 16 | **B** | The `completions` field specifies how many Pods must successfully complete for the Job to be considered finished. | [concepts/workload-resources.md](../concepts/workload-resources.md) |
| 17 | **B** | `*/5 * * * *` means every 5 minutes in cron syntax. | [concepts/workload-resources.md](../concepts/workload-resources.md) |
| 18 | **B** | A toleration matching a taint allows the Pod to be scheduled on that tainted node. | [concepts/scheduling.md](../concepts/scheduling.md) |
| 19 | **B** | `required` is a hard constraint the scheduler must satisfy; `preferred` is a soft constraint it tries to satisfy. | [concepts/scheduling.md](../concepts/scheduling.md) |
| 20 | **B** | Exceeding CPU limits results in throttling; exceeding memory limits results in OOM-kill. | [concepts/scheduling.md](../concepts/scheduling.md) |
| 21 | **B** | Container images are built in immutable layers from a Dockerfile and can be shared across images. | [concepts/containers.md](../concepts/containers.md) |
| 22 | **B** | The `-A` (or `--all-namespaces`) flag lists Pods across all namespaces. | [concepts/kubernetes-api.md](../concepts/kubernetes-api.md) |
| 23 | **B** | Containers in the same Pod share a network namespace and communicate over localhost. | [concepts/pods.md](../concepts/pods.md) |
| 24 | **B** | Init containers run to completion before main containers start, handling setup like migrations or config fetching. | [concepts/pods.md](../concepts/pods.md) |
| 25 | **B** | A failing readiness probe removes the Pod from Service endpoints; a failing liveness probe restarts the container. | [concepts/pods.md](../concepts/pods.md) |
| 26 | **B** | Resource limits define the maximum CPU/memory a container can consume. | [concepts/scheduling.md](../concepts/scheduling.md) |
| 27 | **C** | `kubectl describe` shows detailed resource info including events, status, and conditions. | [concepts/kubernetes-api.md](../concepts/kubernetes-api.md) |
| 28 | **B** | The `spec.selector` in a Deployment matches labels on Pods so the Deployment knows which Pods it owns. | [concepts/deployments.md](../concepts/deployments.md) |
| 29 | **B** | containerd became the dominant CRI runtime after dockershim was removed in Kubernetes 1.24. | [concepts/container-runtimes.md](../concepts/container-runtimes.md) |
| 30 | **B** | runc is the reference low-level OCI runtime that creates and runs containers from OCI bundles. | [concepts/container-runtimes.md](../concepts/container-runtimes.md) |
| 31 | **B** | OCI defines the Image Specification (image format) and Runtime Specification (how to run containers). | [concepts/open-standards.md](../concepts/open-standards.md) |
| 32 | **C** | Cilium uses eBPF for high-performance networking, security, and observability. | [concepts/networking.md](../concepts/networking.md) |
| 33 | **B** | CoreDNS provides cluster DNS so Pods can resolve Service names (e.g., `my-svc.default.svc.cluster.local`). | [concepts/ingress-dns.md](../concepts/ingress-dns.md) |
| 34 | **B** | Ingress resources are just configuration; an Ingress controller implements the actual HTTP routing. | [concepts/ingress-dns.md](../concepts/ingress-dns.md) |
| 35 | **B** | An empty `podSelector: {}` selects all Pods in the namespace where the NetworkPolicy is applied. | [concepts/network-policies.md](../concepts/network-policies.md) |
| 36 | **B** | Roles are namespace-scoped; ClusterRoles are cluster-wide and can cover non-namespaced resources. | [concepts/rbac-security.md](../concepts/rbac-security.md) |
| 37 | **B** | mTLS encrypts traffic and authenticates both sides (client and server) of the connection. | [concepts/service-mesh.md](../concepts/service-mesh.md) |
| 38 | **B** | SMI provides a standard API for service mesh features, allowing portability across implementations. | [concepts/service-mesh.md](../concepts/service-mesh.md) |
| 39 | **B** | Dynamic provisioning automatically creates storage when a PVC is created, removing the need for manual PV creation. | [concepts/storage.md](../concepts/storage.md) |
| 40 | **C** | ReadWriteMany (RWX) allows the volume to be mounted read-write by multiple nodes simultaneously. | [concepts/storage.md](../concepts/storage.md) |
| 41 | **B** | The reclaimPolicy determines what happens to the storage resource when the bound PVC is deleted. | [concepts/storage.md](../concepts/storage.md) |
| 42 | **B** | Cloud native systems are designed for failure — resilience is built in through self-healing and redundancy. | [concepts/cloud-native-principles.md](../concepts/cloud-native-principles.md) |
| 43 | **A** | Factor VII (Port Binding) states that apps should export services by binding to a port. | [concepts/cloud-native-principles.md](../concepts/cloud-native-principles.md) |
| 44 | **B** | Microservices add operational complexity: service discovery, distributed tracing, network latency, and debugging. | [concepts/microservices.md](../concepts/microservices.md) |
| 45 | **B** | VPA adjusts CPU and memory requests/limits on containers based on usage patterns. | [concepts/autoscaling.md](../concepts/autoscaling.md) |
| 46 | **B** | KEDA supports scaling based on external event sources (Kafka, RabbitMQ, etc.) and can scale to zero replicas. | [concepts/autoscaling.md](../concepts/autoscaling.md) |
| 47 | **C** | Kubernetes is a Graduated CNCF project (the first to graduate). | [concepts/cncf-ecosystem.md](../concepts/cncf-ecosystem.md) |
| 48 | **B** | SIGs are community groups that own specific areas of the Kubernetes project (networking, storage, etc.). | [concepts/cncf-ecosystem.md](../concepts/cncf-ecosystem.md) |
| 49 | **B** | Knative provides serverless capabilities on Kubernetes, including scale-to-zero and event-driven workloads. | [concepts/serverless.md](../concepts/serverless.md) |
| 50 | **B** | SLI is the actual measurement (indicator); SLO is the target you set for that measurement (objective). | [concepts/roles-personas.md](../concepts/roles-personas.md) |
| 51 | **B** | Platform Engineering builds internal developer platforms with self-service capabilities, reducing cognitive load. | [concepts/roles-personas.md](../concepts/roles-personas.md) |
| 52 | **B** | A Gauge represents a value that can go up and down (e.g., temperature, memory usage, active connections). | [concepts/prometheus.md](../concepts/prometheus.md) |
| 53 | **B** | Fluentd is a unified logging layer for collecting, transforming, and forwarding log data. | [concepts/observability-tools.md](../concepts/observability-tools.md) |
| 54 | **B** | Fluent Bit is lightweight and fast (forwarder); Fluentd is heavier but more extensible (aggregator). | [concepts/observability-tools.md](../concepts/observability-tools.md) |
| 55 | **B** | A span represents a single operation within a trace, with a name, start time, and duration. | [concepts/opentelemetry.md](../concepts/opentelemetry.md) |
| 56 | **B** | Alertmanager handles deduplication, grouping, routing, and notification of Prometheus alerts. | [concepts/prometheus.md](../concepts/prometheus.md) |
| 57 | **B** | Canary deploys route a small percentage of traffic to the new version first to validate before full rollout. | [concepts/deployment-strategies.md](../concepts/deployment-strategies.md) |
| 58 | **B** | `helm upgrade --install` is idempotent — it installs if the release is new, upgrades if it exists. | [concepts/helm.md](../concepts/helm.md) |
| 59 | **B** | The reconciliation loop continuously ensures the cluster matches the desired state defined in Git. | [concepts/gitops.md](../concepts/gitops.md) |
| 60 | **B** | FluxCD is a set of composable GitOps controllers; ArgoCD offers a more integrated experience with a web UI. | [concepts/gitops.md](../concepts/gitops.md) |

---

## Score Interpretation

| Score | Level | Recommendation |
|-------|-------|----------------|
| 55–60 (92–100%) | Exam ready | Book the exam! |
| 45–54 (75–90%) | Passing range | Review weak domains, then test again |
| 35–44 (58–73%) | Almost there | Re-study the domains where you scored lowest |
| Below 35 (<58%) | Needs more study | Work through concepts and labs systematically |

---

## Domain Score Breakdown

Track your score per domain to find weak areas:

| Domain | Questions | Your Score | Max |
|--------|-----------|------------|-----|
| 1. Kubernetes Fundamentals | 1–28 | ___ | 28 |
| 2. Container Orchestration | 29–41 | ___ | 13 |
| 3. Cloud Native Architecture | 42–51 | ___ | 10 |
| 4. Cloud Native Observability | 52–56 | ___ | 5 |
| 5. Cloud Native App Delivery | 57–60 | ___ | 4 |
| **Total** | | ___ | **60** |
