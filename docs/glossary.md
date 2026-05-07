# KCNA Glossary

> Key terms and definitions for the KCNA exam, organized alphabetically.

---

## A

**Admission Controller** — A plugin that intercepts requests to the API server after authentication and authorization but before the object is persisted. Can mutate or validate requests.

**Alertmanager** — A Prometheus component that handles alert routing, deduplication, grouping, and silencing.

**Annotation** — Non-identifying key-value metadata attached to Kubernetes objects. Used for tool config, build info, or contact details. Not used by selectors.

**ArgoCD** — A GitOps continuous delivery tool that syncs Kubernetes cluster state with definitions stored in a Git repository.

**Autoscaling** — Automatically adjusting compute resources (Pods or Nodes) based on demand or metrics.

## B

**Blue-Green Deployment** — A deployment strategy where two identical environments (blue and green) exist side by side. Traffic is switched entirely from the old version to the new version at once.

## C

**Calico** — A popular CNI plugin providing networking and network policy enforcement for Kubernetes.

**Canary Deployment** — A strategy where a small percentage of traffic is routed to the new version first. If it performs well, traffic is gradually shifted.

**Cilium** — A CNI plugin that uses eBPF for networking, observability, and security in Kubernetes.

**ClusterIP** — The default Service type. Exposes the Service on an internal IP reachable only within the cluster.

**ClusterRole** — A set of RBAC permissions that applies cluster-wide (not namespaced).

**Cloud Controller Manager** — A control plane component that integrates Kubernetes with cloud provider APIs for nodes, routes, and load balancers.

**Cloud Native** — An approach to building and running applications that exploits the advantages of cloud computing: scalable, resilient, observable, loosely coupled, and manageable.

**CNCF** — Cloud Native Computing Foundation. Part of the Linux Foundation. Hosts Kubernetes and 150+ cloud native open-source projects.

**CNI (Container Network Interface)** — A specification and set of plugins that provide network connectivity to containers/Pods.

**ConfigMap** — A Kubernetes object that stores non-sensitive configuration data as key-value pairs. Can be consumed as environment variables or mounted as files.

**Container** — A lightweight, standalone, executable package that includes code, runtime, libraries, and settings. Shares the host OS kernel.

**containerd** — A CRI-compliant container runtime. CNCF graduated. The default runtime in most Kubernetes distributions.

**CoreDNS** — The default DNS server in Kubernetes. Resolves Service names to ClusterIPs within the cluster.

**CRD (Custom Resource Definition)** — Extends the Kubernetes API with custom resource types defined by users.

**CRI (Container Runtime Interface)** — The API specification that Kubernetes uses to communicate with container runtimes.

**CRI-O** — A lightweight CRI-compliant container runtime built specifically for Kubernetes.

**CronJob** — A Kubernetes resource that creates Jobs on a repeating schedule using cron syntax.

**CSI (Container Storage Interface)** — A standard API for exposing storage systems to container orchestrators.

## D

**DaemonSet** — A workload resource that ensures one Pod runs on every (or selected) node(s). Used for logging agents, monitoring, etc.

**Declarative Model** — An approach where you describe the desired state, and the system continuously reconciles actual state to match. Kubernetes uses this model.

**Deployment** — A Kubernetes resource that manages ReplicaSets and provides declarative updates, rolling updates, and rollbacks for Pods.

**DevOps** — A culture and set of practices that brings together software development and IT operations to shorten the development lifecycle.

## E

**eBPF** — Extended Berkeley Packet Filter. A Linux kernel technology used by tools like Cilium for efficient networking and observability.

**EFK Stack** — Elasticsearch + Fluentd + Kibana. A common logging stack for collecting, storing, and visualizing logs.

**Envoy** — A high-performance L7 proxy used as the data plane in Istio service mesh. CNCF graduated.

**Error Budget** — The allowed amount of unreliability (1 - SLO). Used by SRE teams to balance reliability and velocity.

**etcd** — A distributed key-value store that holds all Kubernetes cluster state. Only the API server communicates with etcd directly.

## F

**Flannel** — A simple CNI plugin that provides basic overlay networking for Kubernetes.

**Fluentd** — An open-source log collector and forwarder. CNCF graduated.

**Fluent Bit** — A lightweight log processor and forwarder. Part of the Fluentd ecosystem.

**FluxCD** — A GitOps tool that watches Git repositories and reconciles cluster state to match.

## G

**Grafana** — An open-source visualization and dashboarding platform. Commonly used with Prometheus for metrics dashboards.

**GitOps** — An operational model where Git is the single source of truth for declarative infrastructure and application definitions. Changes are applied automatically via reconciliation.

## H

**Helm** — The package manager for Kubernetes. Uses charts (templates + values) to deploy and manage applications.

**Helm Chart** — A package of pre-configured Kubernetes resource templates that can be deployed as a unit.

**HPA (Horizontal Pod Autoscaler)** — Automatically scales the number of Pod replicas based on observed metrics (CPU, memory, custom metrics).

## I

**Ingress** — A Kubernetes resource that manages external HTTP/HTTPS access to services. Requires an Ingress Controller.

**Ingress Controller** — A component that implements the Ingress rules (e.g., NGINX Ingress Controller, Traefik).

**Init Container** — A container that runs before the main application containers in a Pod. Used for setup tasks.

**Istio** — A popular service mesh that provides traffic management, security (mTLS), and observability using Envoy sidecars.

## J

**Jaeger** — A distributed tracing system used to monitor and troubleshoot transactions in microservices. CNCF graduated.

**Job** — A Kubernetes resource that runs one or more Pods to completion (finite workload).

## K

**KCNA** — Kubernetes and Cloud Native Associate. An entry-level CNCF certification.

**KEDA** — Kubernetes Event-Driven Autoscaling. Scales workloads based on external event sources (queues, streams).

**Knative** — A platform built on Kubernetes for deploying and managing serverless workloads.

**kube-apiserver** — The central API gateway of the Kubernetes control plane. All communication flows through it.

**kube-controller-manager** — Runs reconciliation loops (controllers) that maintain desired state: ReplicaSet, Node, Endpoint controllers, etc.

**kube-proxy** — A network proxy on each node that programs iptables/IPVS rules to route Service traffic to Pods.

**kube-scheduler** — Assigns unscheduled Pods to Nodes using a filter-then-score algorithm.

**kubectl** — The command-line tool for interacting with the Kubernetes API server.

**kubelet** — The agent on each worker node that manages Pod lifecycle and reports node status.

**KubeCon** — The primary conference for the Kubernetes and cloud native community, organized by CNCF.

**Kustomize** — A template-free way to customize Kubernetes YAML configurations using overlays and patches.

## L

**Label** — A key-value pair attached to Kubernetes objects for identification and grouping. Used by selectors to filter resources.

**Label Selector** — A query mechanism that filters Kubernetes resources based on their labels (equality-based or set-based).

**Linkerd** — A lightweight service mesh for Kubernetes. CNCF graduated.

**Liveness Probe** — A health check that detects if a container is stuck. On failure, kubelet restarts the container.

**LoadBalancer** — A Service type that provisions an external cloud load balancer to expose a service.

## M

**Microservices** — An architectural style where an application is composed of small, independent services that communicate over the network.

**Monolith** — A single, unified application where all components are tightly coupled and deployed as one unit.

**mTLS (Mutual TLS)** — Both client and server authenticate each other. Service meshes use mTLS for secure service-to-service communication.

## N

**Namespace** — A mechanism for logically isolating groups of resources within a single Kubernetes cluster.

**NetworkPolicy** — A Kubernetes resource that controls network traffic flow to and from Pods at the IP/port level.

**Node** — A machine (physical or virtual) in a Kubernetes cluster that runs Pods. Can be a control plane node or worker node.

**Node Affinity** — Scheduling rules that attract Pods to specific nodes based on node labels.

**NodePort** — A Service type that exposes the service on a static port on every node's IP.

## O

**OCI (Open Container Initiative)** — An organization that defines open standards for container image formats and runtimes.

**OpenTelemetry** — A CNCF project providing a vendor-neutral framework for collecting metrics, logs, and traces. Merges OpenTracing and OpenCensus.

**OPA (Open Policy Agent)** — A general-purpose policy engine. OPA Gatekeeper integrates it with Kubernetes admission control.

## P

**Platform Engineering** — The practice of building and maintaining internal developer platforms (IDPs) to improve developer experience and productivity.

**Pod** — The smallest deployable unit in Kubernetes. Contains one or more containers that share networking and storage.

**Pod Security Standards** — Three policy levels for Pod security: Privileged, Baseline, Restricted.

**PersistentVolume (PV)** — A cluster-level storage resource provisioned by an administrator or dynamically via a StorageClass.

**PersistentVolumeClaim (PVC)** — A request for storage by a user. Binds to a matching PV.

**Prometheus** — A monitoring and alerting toolkit. Uses a pull model to scrape metrics. CNCF graduated.

**PromQL** — The query language for Prometheus metrics.

## R

**RBAC (Role-Based Access Control)** — A method of regulating access based on the roles of users. Uses Roles, ClusterRoles, RoleBindings, and ClusterRoleBindings.

**Readiness Probe** — A check that determines if a container is ready to receive traffic. On failure, the Pod is removed from Service endpoints.

**Reconciliation Loop** — A control loop that continuously compares desired state to actual state and takes corrective action. Core pattern in Kubernetes controllers.

**ReplicaSet** — Ensures a specified number of Pod replicas are running at any given time.

**Rolling Update** — The default Kubernetes deployment strategy. Pods are replaced incrementally, ensuring zero downtime.

**Role** — An RBAC resource that defines permissions within a specific namespace.

**RoleBinding** — Grants the permissions defined in a Role to a user, group, or ServiceAccount within a namespace.

**runc** — A low-level OCI-compliant container runtime. The reference implementation that actually creates and runs containers.

## S

**Secret** — A Kubernetes object for storing sensitive data (passwords, tokens, keys). Base64-encoded (not encrypted by default).

**Selector** — A query used to filter Kubernetes resources by their labels.

**Serverless** — An execution model where the cloud provider manages infrastructure and automatically scales resources. Developers provide only the function code.

**Service** — A Kubernetes abstraction that provides a stable network endpoint for accessing a set of Pods.

**Service Mesh** — An infrastructure layer for managing service-to-service communication. Provides mTLS, traffic management, and observability via sidecar proxies.

**Sidecar** — A container that runs alongside the main application container in a Pod, providing supplementary functionality (e.g., proxy, logging).

**SLI (Service Level Indicator)** — A metric that measures a specific aspect of service performance (e.g., request latency, error rate).

**SLO (Service Level Objective)** — A target value or range for an SLI (e.g., "99.9% of requests complete within 200ms").

**SLA (Service Level Agreement)** — A formal contract specifying consequences if SLOs are not met.

**SMI (Service Mesh Interface)** — A standard API specification for service meshes on Kubernetes.

**SRE (Site Reliability Engineering)** — A discipline that applies software engineering practices to infrastructure and operations. Focuses on reliability via SLOs and error budgets.

**StatefulSet** — A workload resource for managing stateful applications. Provides stable network identity, ordered deployment, and persistent storage.

**StorageClass** — Defines a class of storage and enables dynamic provisioning of PersistentVolumes.

**Startup Probe** — Indicates whether the container application has started. Disables liveness/readiness checks until it succeeds.

## T

**Taint** — A property applied to a node that repels Pods unless they have a matching toleration.

**Tekton** — A Kubernetes-native CI/CD framework. CNCF project.

**Toleration** — A property on a Pod that allows it to be scheduled on a node with a matching taint.

**TOC (Technical Oversight Committee)** — The CNCF body that oversees project acceptance and maturity level progression.

**12-Factor App** — A methodology for building cloud native applications. Key principles: config in env vars, stateless processes, disposability, port binding.

## V

**VPA (Vertical Pod Autoscaler)** — Automatically adjusts the CPU and memory requests of Pods based on historical usage.

## W

**Worker Node** — A machine that runs application workloads (Pods). Runs kubelet, kube-proxy, and a container runtime.

## Z

**Zipkin** — A distributed tracing system for troubleshooting latency in microservices architectures.
