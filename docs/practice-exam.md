# KCNA Practice Exam

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

**Q1.** Which control plane component is the only one that communicates directly with etcd?

- A) kube-scheduler
- B) kube-controller-manager
- C) kube-apiserver
- D) kubelet

**Q2.** What is the primary role of etcd in a Kubernetes cluster?

- A) Scheduling Pods to Nodes
- B) Running container images
- C) Storing all cluster state as key-value pairs
- D) Managing network routing rules

**Q3.** A Pod is stuck in `Pending` state. Which component is most likely responsible for assigning it to a node?

- A) kubelet
- B) kube-proxy
- C) kube-scheduler
- D) kube-controller-manager

**Q4.** Which of the following best describes the Kubernetes declarative model?

- A) Users issue step-by-step commands that Kubernetes executes in order
- B) Users describe the desired state and controllers reconcile actual state to match
- C) Users directly manage containers on each node
- D) Users define network routes that Kubernetes follows statically

**Q5.** What does the kubelet do on a worker node?

- A) Schedules Pods across the cluster
- B) Stores cluster state in a key-value store
- C) Manages Pod lifecycle and reports node status to the API server
- D) Configures iptables rules for Services

**Q6.** Which component programs iptables or IPVS rules on worker nodes to route Service traffic?

- A) kubelet
- B) kube-proxy
- C) kube-scheduler
- D) CoreDNS

**Q7.** Which resource ensures a specified number of identical Pods are running at all times?

- A) DaemonSet
- B) StatefulSet
- C) ReplicaSet
- D) Job

**Q8.** What is the relationship between a Deployment and a ReplicaSet?

- A) They are the same resource
- B) A ReplicaSet creates Deployments
- C) A Deployment manages ReplicaSets, which in turn manage Pods
- D) A Deployment bypasses ReplicaSets and manages Pods directly

**Q9.** Which Service type exposes a service on a static port on every node in the cluster?

- A) ClusterIP
- B) NodePort
- C) LoadBalancer
- D) ExternalName

**Q10.** What is the default Service type in Kubernetes?

- A) NodePort
- B) LoadBalancer
- C) ClusterIP
- D) ExternalName

**Q11.** Which resource provides logical isolation of resources within a cluster?

- A) Labels
- B) Annotations
- C) Namespaces
- D) Taints

**Q12.** How are labels used in Kubernetes?

- A) To store large metadata blobs on resources
- B) To identify and select groups of resources using selectors
- C) To encrypt data at rest
- D) To define resource quotas

**Q13.** What is the main difference between a ConfigMap and a Secret?

- A) ConfigMaps can only store strings; Secrets can store any data type
- B) Secrets are base64-encoded and intended for sensitive data; ConfigMaps are for non-sensitive configuration
- C) ConfigMaps are stored in etcd; Secrets are stored on the node filesystem
- D) There is no functional difference

**Q14.** Which workload resource ensures exactly one Pod runs on every node in the cluster?

- A) ReplicaSet
- B) StatefulSet
- C) DaemonSet
- D) Deployment

**Q15.** What distinguishes a StatefulSet from a Deployment?

- A) StatefulSets cannot be scaled
- B) StatefulSets provide stable network identities and persistent storage for each Pod
- C) Deployments cannot do rolling updates
- D) StatefulSets do not use ReplicaSets

**Q16.** A Job resource in Kubernetes is designed to:

- A) Run Pods indefinitely
- B) Run a task to completion and then stop
- C) Schedule Pods on a recurring basis
- D) Manage long-running services

**Q17.** Which resource is used to schedule Jobs on a recurring time-based schedule?

- A) DaemonSet
- B) CronJob
- C) ReplicaSet
- D) StatefulSet

**Q18.** What is the effect of applying a taint to a node?

- A) It increases the node's priority for scheduling
- B) It repels Pods that do not have a matching toleration
- C) It labels the node for identification
- D) It drains all existing Pods from the node

**Q19.** Node affinity rules are used to:

- A) Prevent a node from joining the cluster
- B) Attract Pods to specific nodes based on node labels
- C) Set resource limits on nodes
- D) Configure network policies on nodes

**Q20.** What are resource requests in Kubernetes?

- A) The maximum amount of CPU/memory a container can use
- B) The guaranteed minimum amount of CPU/memory allocated to a container for scheduling
- C) The actual current usage of a container
- D) Annotations for monitoring purposes

**Q21.** What is the difference between a container and a virtual machine?

- A) Containers include a full operating system; VMs share the host kernel
- B) VMs include a full OS with a hypervisor; containers share the host kernel and are more lightweight
- C) There is no difference
- D) Containers are heavier and slower than VMs

**Q22.** Which command displays the current context and cluster that kubectl is configured to use?

- A) kubectl get nodes
- B) kubectl cluster-info
- C) kubectl config current-context
- D) kubectl describe cluster

**Q23.** What is the smallest deployable unit in Kubernetes?

- A) Container
- B) Node
- C) Pod
- D) Deployment

**Q24.** A Pod can contain:

- A) Exactly one container
- B) One or more containers that share the same network namespace and storage
- C) Multiple containers each with their own IP address
- D) Only init containers

**Q25.** Which annotation vs label statement is correct?

- A) Annotations are used by selectors to group resources
- B) Labels are for large metadata; annotations are for small identifiers
- C) Annotations store non-identifying metadata (e.g., build info, contact); labels identify and select resources
- D) Annotations and labels are interchangeable

**Q26.** What happens when a container exceeds its memory limit?

- A) The container is throttled
- B) The container is OOM-killed (terminated)
- C) The node is drained
- D) Nothing; limits are advisory

**Q27.** Which API group do Deployments belong to?

- A) v1 (core)
- B) apps/v1
- C) batch/v1
- D) networking.k8s.io/v1

**Q28.** What is the purpose of a liveness probe?

- A) To check if a container is ready to accept traffic
- B) To detect if a container is stuck and needs to be restarted
- C) To delay container startup
- D) To monitor CPU usage

---

## Domain 2 — Container Orchestration (Questions 29–41)

**Q29.** What does CRI stand for in the Kubernetes ecosystem?

- A) Container Resource Interface
- B) Container Runtime Interface
- C) Cluster Runtime Integration
- D) Container Registry Index

**Q30.** Which of the following is a CRI-compliant container runtime?

- A) Docker Engine (dockershim)
- B) containerd
- C) VirtualBox
- D) Vagrant

**Q31.** What is the OCI responsible for?

- A) Managing Kubernetes releases
- B) Defining industry standards for container image formats and runtimes
- C) Providing a container registry service
- D) Certifying Kubernetes administrators

**Q32.** What is the role of a CNI plugin in Kubernetes?

- A) Managing container images
- B) Providing network connectivity to Pods
- C) Scheduling Pods to Nodes
- D) Storing cluster state

**Q33.** In Kubernetes networking, which statement is true?

- A) Pods on different nodes cannot communicate without NAT
- B) Every Pod gets its own unique IP address and can communicate with any other Pod without NAT
- C) Pods share IP addresses within a node
- D) Services are required for any pod-to-pod communication

**Q34.** What is an Ingress resource used for?

- A) Encrypting etcd data
- B) Managing external HTTP/HTTPS access to services within the cluster
- C) Creating persistent volumes
- D) Scheduling Pods across zones

**Q35.** Which Kubernetes resource restricts network traffic between Pods?

- A) ResourceQuota
- B) PodSecurityPolicy
- C) NetworkPolicy
- D) LimitRange

**Q36.** RBAC in Kubernetes uses which of the following to define permissions?

- A) Roles and RoleBindings (or ClusterRoles and ClusterRoleBindings)
- B) ConfigMaps and Secrets
- C) NetworkPolicies
- D) Annotations and Labels

**Q37.** What is the sidecar pattern in a service mesh?

- A) Running two identical clusters side by side
- B) Injecting a proxy container alongside the application container in every Pod to handle network traffic
- C) A backup container that takes over when the main container fails
- D) A logging container that writes to local storage

**Q38.** Which of the following are popular service mesh implementations? (Select the best answer)

- A) Prometheus and Grafana
- B) Istio and Linkerd
- C) Helm and Kustomize
- D) ArgoCD and FluxCD

**Q39.** What does CSI stand for?

- A) Cluster State Interface
- B) Container Storage Interface
- C) Container Security Integration
- D) Cloud Service Index

**Q40.** What is the relationship between a PersistentVolume (PV) and a PersistentVolumeClaim (PVC)?

- A) A PV and PVC are the same resource
- B) A PV represents storage provisioned by an admin; a PVC is a request for that storage by a user
- C) A PVC provisions storage; a PV requests it
- D) PVCs can only be used with StatefulSets

**Q41.** Which Kubernetes resource dynamically provisions PersistentVolumes?

- A) PersistentVolumeClaim
- B) StorageClass
- C) ConfigMap
- D) DaemonSet

---

## Domain 3 — Cloud Native Architecture (Questions 42–51)

**Q42.** Which of the following is NOT a characteristic of cloud native applications?

- A) Designed for scalability
- B) Tightly coupled monolithic architecture
- C) Resilient and self-healing
- D) Observable and manageable

**Q43.** The 12-Factor App methodology recommends:

- A) Storing configuration in the codebase
- B) Storing configuration in environment variables, separate from code
- C) Using a single deployment target for all environments
- D) Bundling all dependencies into the container image statically

**Q44.** What is a key advantage of microservices over monoliths?

- A) Simpler debugging across the entire application
- B) Independent deployment and scaling of individual services
- C) No need for network communication between components
- D) Lower infrastructure costs in all cases

**Q45.** What does HPA stand for in Kubernetes?

- A) Horizontal Pod Autoscaler
- B) High Performance Allocation
- C) Host Port Assignment
- D) Horizontal Proxy Agent

**Q46.** How does the Horizontal Pod Autoscaler work?

- A) It adds more nodes to the cluster
- B) It adjusts the number of Pod replicas based on observed metrics (e.g., CPU utilization)
- C) It increases CPU and memory limits on existing Pods
- D) It migrates Pods to larger nodes

**Q47.** What is the Cluster Autoscaler responsible for?

- A) Scaling the number of Pod replicas
- B) Adjusting container resource requests
- C) Adding or removing nodes in the cluster based on pending Pods and resource utilization
- D) Restarting failed Pods

**Q48.** CNCF projects follow a maturity model. What are the three stages in order?

- A) Alpha, Beta, Stable
- B) Sandbox, Incubating, Graduated
- C) Draft, Review, Published
- D) Experimental, Production, Retired

**Q49.** What is the role of the CNCF Technical Oversight Committee (TOC)?

- A) Writing Kubernetes source code
- B) Overseeing project acceptance and maturity level progression
- C) Selling Kubernetes certifications
- D) Managing cloud provider integrations

**Q50.** Which of the following best describes serverless computing?

- A) Running servers without an operating system
- B) An execution model where the cloud provider manages infrastructure and scales automatically; developers only provide code
- C) A model where servers are shared between tenants
- D) Running containers without Kubernetes

**Q51.** An SLO (Service Level Objective) is:

- A) A legal contract between a service provider and customer
- B) A target value or range for a service level measured by an SLI
- C) A metric that measures actual service performance
- D) A tool for monitoring Kubernetes clusters

---

## Domain 4 — Cloud Native Observability (Questions 52–56)

**Q52.** What are the three pillars of observability?

- A) CPU, Memory, Disk
- B) Metrics, Logs, Traces
- C) Availability, Latency, Throughput
- D) Alerts, Dashboards, Reports

**Q53.** Prometheus collects metrics using which model?

- A) Push model — applications send metrics to Prometheus
- B) Pull model — Prometheus scrapes metrics from target endpoints
- C) Streaming model — metrics flow through a message queue
- D) Polling model — agents on each node collect and batch metrics

**Q54.** What is the primary purpose of Grafana in the observability stack?

- A) Collecting and storing metrics
- B) Visualizing metrics and logs through dashboards
- C) Tracing requests across services
- D) Alerting on security vulnerabilities

**Q55.** What is OpenTelemetry?

- A) A container runtime
- B) A vendor-neutral framework for collecting and exporting telemetry data (metrics, logs, traces)
- C) A Kubernetes scheduler plugin
- D) A service mesh implementation

**Q56.** Which tool is commonly used for distributed tracing?

- A) Prometheus
- B) Fluentd
- C) Jaeger
- D) Grafana

---

## Domain 5 — Cloud Native Application Delivery (Questions 57–60)

**Q57.** In a blue-green deployment, what happens when the new version is ready?

- A) Traffic is gradually shifted from old to new
- B) All traffic is switched from the blue (old) environment to the green (new) environment at once
- C) New Pods replace old Pods one at a time
- D) The old version is deleted before the new one starts

**Q58.** What is a Helm chart?

- A) A monitoring dashboard
- B) A package of pre-configured Kubernetes resource templates that can be deployed as a unit
- C) A network policy definition
- D) A container image format

**Q59.** Which of the following is a core principle of GitOps?

- A) Infrastructure is defined imperatively through CLI commands
- B) Git is the single source of truth for declarative infrastructure and application definitions
- C) Deployments are triggered manually through a web UI
- D) Configuration is stored in a database

**Q60.** ArgoCD is best described as:

- A) A CI pipeline tool like Jenkins
- B) A GitOps continuous delivery tool that syncs Kubernetes cluster state with definitions in a Git repository
- C) A container image registry
- D) A Kubernetes network plugin

---

## Answer Key

> Scroll down only after completing the exam!

<br><br><br><br><br><br><br><br><br><br>

| # | Answer | Explanation | Reference |
|---|--------|-------------|-----------|
| 1 | **C** | Only the kube-apiserver communicates directly with etcd. | [concepts/control-plane.md](../concepts/control-plane.md) |
| 2 | **C** | etcd is a distributed key-value store holding all cluster state. | [concepts/control-plane.md](../concepts/control-plane.md) |
| 3 | **C** | The kube-scheduler assigns Pods to nodes. Pending state means no node has been assigned yet. | [concepts/scheduling.md](../concepts/scheduling.md) |
| 4 | **B** | Kubernetes uses a declarative model where users define desired state and controllers reconcile. | [concepts/kubernetes-architecture.md](../concepts/kubernetes-architecture.md) |
| 5 | **C** | The kubelet manages Pod lifecycle on each node and reports status back. | [concepts/worker-nodes.md](../concepts/worker-nodes.md) |
| 6 | **B** | kube-proxy programs iptables/IPVS rules to route Service traffic. | [concepts/worker-nodes.md](../concepts/worker-nodes.md) |
| 7 | **C** | A ReplicaSet ensures a specified number of Pod replicas are running. | [concepts/deployments.md](../concepts/deployments.md) |
| 8 | **C** | A Deployment creates and manages ReplicaSets, which manage Pods. | [concepts/deployments.md](../concepts/deployments.md) |
| 9 | **B** | NodePort exposes a Service on a static port on every node. | [concepts/services.md](../concepts/services.md) |
| 10 | **C** | ClusterIP is the default Service type, accessible only within the cluster. | [concepts/services.md](../concepts/services.md) |
| 11 | **C** | Namespaces provide logical isolation of resources. | [concepts/namespaces-labels.md](../concepts/namespaces-labels.md) |
| 12 | **B** | Labels are key-value pairs used to identify and select resources. | [concepts/namespaces-labels.md](../concepts/namespaces-labels.md) |
| 13 | **B** | Secrets are base64-encoded and designed for sensitive data; ConfigMaps are for non-sensitive config. | [concepts/configmaps-secrets.md](../concepts/configmaps-secrets.md) |
| 14 | **C** | A DaemonSet ensures one Pod per node. | [concepts/workload-resources.md](../concepts/workload-resources.md) |
| 15 | **B** | StatefulSets provide stable network identities, ordered deployment, and persistent storage. | [concepts/workload-resources.md](../concepts/workload-resources.md) |
| 16 | **B** | A Job runs a task to completion (finite workload). | [concepts/workload-resources.md](../concepts/workload-resources.md) |
| 17 | **B** | CronJobs schedule Jobs on a time-based schedule (cron syntax). | [concepts/workload-resources.md](../concepts/workload-resources.md) |
| 18 | **B** | Taints repel Pods that lack a matching toleration. | [concepts/scheduling.md](../concepts/scheduling.md) |
| 19 | **B** | Node affinity rules attract Pods to nodes based on node labels. | [concepts/scheduling.md](../concepts/scheduling.md) |
| 20 | **B** | Resource requests are the guaranteed minimum for scheduling decisions. | [concepts/scheduling.md](../concepts/scheduling.md) |
| 21 | **B** | VMs run a full OS on a hypervisor; containers share the host kernel and are lighter. | [concepts/containers.md](../concepts/containers.md) |
| 22 | **C** | `kubectl config current-context` shows the active context. | [concepts/kubernetes-api.md](../concepts/kubernetes-api.md) |
| 23 | **C** | The Pod is the smallest deployable unit in Kubernetes. | [concepts/pods.md](../concepts/pods.md) |
| 24 | **B** | A Pod can contain one or more containers sharing the same network namespace and storage volumes. | [concepts/pods.md](../concepts/pods.md) |
| 25 | **C** | Annotations store non-identifying metadata; labels identify and select resources. | [concepts/namespaces-labels.md](../concepts/namespaces-labels.md) |
| 26 | **B** | Exceeding memory limits results in the container being OOM-killed. | [concepts/scheduling.md](../concepts/scheduling.md) |
| 27 | **B** | Deployments belong to the `apps/v1` API group. | [concepts/kubernetes-api.md](../concepts/kubernetes-api.md) |
| 28 | **B** | A liveness probe detects stuck containers and triggers a restart. | [concepts/pods.md](../concepts/pods.md) |
| 29 | **B** | CRI = Container Runtime Interface. | [concepts/container-runtimes.md](../concepts/container-runtimes.md) |
| 30 | **B** | containerd is a CRI-compliant runtime. Docker Engine used dockershim (now removed). | [concepts/container-runtimes.md](../concepts/container-runtimes.md) |
| 31 | **B** | OCI (Open Container Initiative) defines standards for container images and runtimes. | [concepts/open-standards.md](../concepts/open-standards.md) |
| 32 | **B** | CNI plugins provide network connectivity to Pods. | [concepts/networking.md](../concepts/networking.md) |
| 33 | **B** | Every Pod gets a unique IP; pods communicate across nodes without NAT. | [concepts/networking.md](../concepts/networking.md) |
| 34 | **B** | Ingress manages external HTTP/HTTPS access to cluster services. | [concepts/ingress-dns.md](../concepts/ingress-dns.md) |
| 35 | **C** | NetworkPolicy restricts traffic between Pods. | [concepts/network-policies.md](../concepts/network-policies.md) |
| 36 | **A** | RBAC uses Roles/ClusterRoles and RoleBindings/ClusterRoleBindings. | [concepts/rbac-security.md](../concepts/rbac-security.md) |
| 37 | **B** | The sidecar pattern injects a proxy container alongside the app container. | [concepts/service-mesh.md](../concepts/service-mesh.md) |
| 38 | **B** | Istio and Linkerd are popular service mesh implementations. | [concepts/service-mesh.md](../concepts/service-mesh.md) |
| 39 | **B** | CSI = Container Storage Interface. | [concepts/storage.md](../concepts/storage.md) |
| 40 | **B** | A PV is provisioned storage; a PVC is a user's request to consume it. | [concepts/storage.md](../concepts/storage.md) |
| 41 | **B** | StorageClass enables dynamic provisioning of PersistentVolumes. | [concepts/storage.md](../concepts/storage.md) |
| 42 | **B** | Cloud native apps are loosely coupled, not tightly coupled monoliths. | [concepts/cloud-native-principles.md](../concepts/cloud-native-principles.md) |
| 43 | **B** | 12-Factor recommends storing config in environment variables. | [concepts/cloud-native-principles.md](../concepts/cloud-native-principles.md) |
| 44 | **B** | Microservices allow independent deployment and scaling of services. | [concepts/microservices.md](../concepts/microservices.md) |
| 45 | **A** | HPA = Horizontal Pod Autoscaler. | [concepts/autoscaling.md](../concepts/autoscaling.md) |
| 46 | **B** | HPA adjusts Pod replica count based on observed metrics. | [concepts/autoscaling.md](../concepts/autoscaling.md) |
| 47 | **C** | Cluster Autoscaler adds/removes nodes based on resource demand. | [concepts/autoscaling.md](../concepts/autoscaling.md) |
| 48 | **B** | CNCF maturity: Sandbox -> Incubating -> Graduated. | [concepts/cncf-ecosystem.md](../concepts/cncf-ecosystem.md) |
| 49 | **B** | The TOC oversees project acceptance and maturity progression. | [concepts/cncf-ecosystem.md](../concepts/cncf-ecosystem.md) |
| 50 | **B** | Serverless: provider manages infra, auto-scales, developers provide code. | [concepts/serverless.md](../concepts/serverless.md) |
| 51 | **B** | An SLO is a target value for a service level measured by an SLI. | [concepts/roles-personas.md](../concepts/roles-personas.md) |
| 52 | **B** | Three pillars: Metrics, Logs, Traces. | [concepts/observability.md](../concepts/observability.md) |
| 53 | **B** | Prometheus uses a pull model — it scrapes metrics endpoints. | [concepts/prometheus.md](../concepts/prometheus.md) |
| 54 | **B** | Grafana is a visualization and dashboarding tool. | [concepts/observability-tools.md](../concepts/observability-tools.md) |
| 55 | **B** | OpenTelemetry is a vendor-neutral telemetry collection framework. | [concepts/opentelemetry.md](../concepts/opentelemetry.md) |
| 56 | **C** | Jaeger is a distributed tracing tool. | [concepts/observability-tools.md](../concepts/observability-tools.md) |
| 57 | **B** | Blue-green switches all traffic at once from old to new. | [concepts/deployment-strategies.md](../concepts/deployment-strategies.md) |
| 58 | **B** | A Helm chart is a package of Kubernetes resource templates. | [concepts/helm.md](../concepts/helm.md) |
| 59 | **B** | GitOps: Git is the single source of truth for declarative definitions. | [concepts/gitops.md](../concepts/gitops.md) |
| 60 | **B** | ArgoCD is a GitOps CD tool that syncs cluster state from Git. | [concepts/gitops.md](../concepts/gitops.md) |

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
