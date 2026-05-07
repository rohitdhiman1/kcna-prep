# Kubernetes Networking

> Every pod gets its own IP address. Pods communicate with each other directly without NAT. This flat networking model is fundamental to how Kubernetes works.

---

## The Kubernetes Networking Model

Kubernetes imposes three fundamental rules on any networking implementation:

1. **Every pod gets its own unique IP address.**
2. **Pods on any node can communicate with pods on any other node without NAT.**
3. **Agents on a node (kubelet, kube-proxy) can communicate with all pods on that node.**

These rules create a **flat network** where every pod can reach every other pod by IP, regardless of which node it runs on.

```
  Kubernetes Flat Network Model

  ┌─────────── Cluster Network (e.g., 10.244.0.0/16) ───────────┐
  │                                                                │
  │  Node 1 (10.0.1.10)              Node 2 (10.0.1.11)          │
  │  ┌─────────────────────┐         ┌─────────────────────┐      │
  │  │                     │         │                     │      │
  │  │  Pod A              │         │  Pod C              │      │
  │  │  10.244.1.5         │         │  10.244.2.8         │      │
  │  │                     │         │                     │      │
  │  │  Pod B              │  ◄───►  │  Pod D              │      │
  │  │  10.244.1.6         │  No NAT │  10.244.2.9         │      │
  │  │                     │         │                     │      │
  │  └─────────────────────┘         └─────────────────────┘      │
  │                                                                │
  │  Pod A (10.244.1.5) can directly reach Pod D (10.244.2.9)    │
  │  No network address translation needed.                       │
  └────────────────────────────────────────────────────────────────┘
```

---

## Container Networking Interface (CNI)

**CNI** is the standard that defines how networking is configured for containers. Kubernetes uses CNI plugins to implement the networking model.

### How CNI Works

```
  CNI Flow (Pod Creation)

  kubelet                CNI Plugin              Network
  ┌──────┐   1. ADD     ┌──────────┐            ┌────────┐
  │      │─────────────►│          │ 3. Set up  │        │
  │      │  (pod info)  │  Calico  │───────────►│ Routes │
  │      │              │  Flannel │  routes,   │ veth   │
  │      │◄─────────────│  Cilium  │  bridges,  │ pairs  │
  │      │  2. Return   │          │  overlays  │        │
  └──────┘  (pod IP)    └──────────┘            └────────┘
```

When a pod is created:
1. kubelet calls the CNI plugin with ADD command.
2. The CNI plugin assigns an IP to the pod.
3. The plugin configures networking (virtual ethernet pairs, bridges, routes).
4. The pod can now communicate on the cluster network.

### Popular CNI Plugins

| Plugin | Key Features | Notes |
|--------|-------------|-------|
| **Calico** | Network policies, BGP routing, eBPF | Most popular. Supports NetworkPolicy. |
| **Flannel** | Simple overlay network (VXLAN) | Easy to set up. No NetworkPolicy support alone. |
| **Cilium** | eBPF-based, advanced security, observability | High performance. CNCF Graduated. |
| **Weave Net** | Mesh network, encryption | Simple, encrypted by default. |
| **Canal** | Flannel networking + Calico policies | Combines the best of both. |

**For the exam**: Know that CNI is the standard, and be aware of Calico, Flannel, and Cilium by name.

---

## Pod-to-Pod Communication

### Same Node

Pods on the same node communicate via a **virtual bridge**:

```
  Same-Node Communication

  ┌──────────────── Node 1 ──────────────────┐
  │                                           │
  │  Pod A               Pod B               │
  │  10.244.1.5          10.244.1.6           │
  │  ┌──────┐            ┌──────┐            │
  │  │ eth0 │            │ eth0 │            │
  │  └──┬───┘            └──┬───┘            │
  │     │ veth pair          │ veth pair      │
  │     │                    │                │
  │  ┌──┴────────────────────┴──┐             │
  │  │      cbr0 (bridge)       │             │
  │  │      10.244.1.1          │             │
  │  └──────────────────────────┘             │
  │                                           │
  └───────────────────────────────────────────┘

  Traffic: Pod A ──► bridge ──► Pod B
  (stays on the same node, very fast)
```

### Cross-Node

Pods on different nodes communicate through the cluster network:

```
  Cross-Node Communication

  Node 1                              Node 2
  ┌───────────────────┐               ┌───────────────────┐
  │  Pod A            │               │  Pod C            │
  │  10.244.1.5       │               │  10.244.2.8       │
  │  ┌──────┐         │               │  ┌──────┐        │
  │  │ eth0 │         │               │  │ eth0 │        │
  │  └──┬───┘         │               │  └──┬───┘        │
  │     │             │               │     │             │
  │  ┌──┴──────────┐  │               │  ┌──┴──────────┐ │
  │  │   bridge    │  │               │  │   bridge    │ │
  │  └──────┬──────┘  │               │  └──────┬──────┘ │
  │         │         │               │         │        │
  │  ┌──────┴──────┐  │               │  ┌──────┴──────┐ │
  │  │  Node NIC   │  │               │  │  Node NIC   │ │
  │  │  eth0       │  │               │  │  eth0       │ │
  │  └──────┬──────┘  │               │  └──────┬──────┘ │
  └─────────┼─────────┘               └─────────┼────────┘
            │                                   │
            └───────────┬───────────────────────┘
                        │
               ┌────────┴────────┐
               │  Physical/Cloud │
               │  Network        │
               │  (overlay or    │
               │   routed)       │
               └─────────────────┘
```

The CNI plugin handles cross-node communication using either:
- **Overlay networks** (VXLAN) — encapsulate pod traffic in node-to-node packets (Flannel default)
- **Direct routing** (BGP) — advertise pod CIDRs so the network routes directly (Calico option)
- **eBPF** — kernel-level packet processing, bypassing iptables (Cilium)

---

## Service Networking

Pods are ephemeral — they come and go. **Services** provide a stable endpoint.

### How Services Work

```
  Service Networking

  ┌──────────────────────────────────────────────────────┐
  │                    Service                            │
  │            "my-service" (ClusterIP: 10.96.45.12)     │
  │                        │                              │
  │                   kube-proxy                          │
  │              (iptables / IPVS rules)                  │
  │                        │                              │
  │            ┌───────────┼───────────┐                  │
  │            ▼           ▼           ▼                  │
  │       ┌────────┐  ┌────────┐  ┌────────┐             │
  │       │ Pod 1  │  │ Pod 2  │  │ Pod 3  │             │
  │       │10.244  │  │10.244  │  │10.244  │             │
  │       │ .1.5   │  │ .2.8   │  │ .1.9   │             │
  │       └────────┘  └────────┘  └────────┘             │
  └──────────────────────────────────────────────────────┘

  Client sends request to 10.96.45.12:80
  kube-proxy's rules redirect to one of the pod IPs
```

### kube-proxy

kube-proxy runs on every node and maintains network rules for Services. It operates in one of three modes:

| Mode | How It Works | Notes |
|------|-------------|-------|
| **iptables** (default) | Creates iptables rules that DNAT traffic to pod IPs | Random selection, no health checking |
| **IPVS** | Uses Linux IPVS kernel module for load balancing | Better performance at scale, multiple algorithms |
| **nftables** | Uses nftables instead of iptables | Newer, available from K8s 1.29+ |

### Service Types

- **ClusterIP** — internal-only virtual IP (default)
- **NodePort** — exposes on a port on every node (30000-32767)
- **LoadBalancer** — provisions an external load balancer (cloud environments)
- **ExternalName** — maps to an external DNS name (CNAME)

---

## CoreDNS — Service Discovery

CoreDNS is the **cluster DNS server** in Kubernetes. It runs as a Deployment in the `kube-system` namespace.

### DNS Records Created Automatically

Every Service gets a DNS entry:

```
  DNS Format:
  <service-name>.<namespace>.svc.cluster.local

  Examples:
  my-app.default.svc.cluster.local          ──► ClusterIP of my-app
  my-db.production.svc.cluster.local        ──► ClusterIP of my-db

  Pods can use short names within the same namespace:
  curl http://my-app           (resolves within same namespace)
  curl http://my-app.production (cross-namespace)
```

### How DNS Resolution Works

```
  DNS Resolution Flow

  ┌────────┐    1. DNS query:        ┌──────────┐
  │  Pod   │    "my-app"             │ CoreDNS  │
  │        │────────────────────────►│          │
  │        │                         │ (kube-   │
  │        │◄────────────────────────│  system) │
  │        │    2. Response:         │          │
  └────────┘    10.96.45.12          └──────────┘
       │
       │  3. Connect to 10.96.45.12
       ▼
  ┌──────────┐
  │ Service  │──► kube-proxy rules ──► Pod
  └──────────┘
```

Pod DNS configuration is set automatically via `/etc/resolv.conf`:
```
nameserver 10.96.0.10        # CoreDNS ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
```

---

## Network Address Spaces

Kubernetes uses three distinct IP ranges:

```
  Three Network Ranges

  ┌────────────────────────────────────────────────┐
  │  1. Node Network     (e.g., 192.168.1.0/24)   │
  │     Real IPs of the machines                    │
  │                                                 │
  │  2. Pod Network      (e.g., 10.244.0.0/16)    │
  │     Assigned by CNI plugin to pods              │
  │     Each node gets a /24 subnet                 │
  │                                                 │
  │  3. Service Network  (e.g., 10.96.0.0/12)     │
  │     Virtual IPs for Services                    │
  │     Managed by kube-proxy (iptables/IPVS)      │
  └────────────────────────────────────────────────┘

  These ranges must not overlap!
```

---

## Key Exam Points

- **Every pod gets a unique IP**. No NAT between pods. This is the fundamental Kubernetes networking rule.
- **CNI** (Container Networking Interface) is the standard for network plugins. The kubelet calls the CNI plugin to set up pod networking.
- **Calico** supports NetworkPolicy. **Flannel** alone does not. **Cilium** uses eBPF for high performance.
- **kube-proxy** implements Service networking using iptables (default), IPVS, or nftables rules.
- **CoreDNS** provides service discovery. DNS format: `service.namespace.svc.cluster.local`.
- **Three IP ranges**: node network, pod network (CNI), service network (kube-proxy). They must not overlap.
- Cross-node pod communication uses either overlay networks (VXLAN) or direct routing (BGP).

---

## What to Remember for the Exam

1. **Flat network model** — all pods can reach all other pods without NAT. This is a requirement, not optional.
2. **CNI is the plugin interface** — like CRI for runtimes and CSI for storage. Know all three acronyms.
3. **Know CNI plugins by name** — Calico (most popular, supports NetworkPolicy), Flannel (simple, no NetworkPolicy), Cilium (eBPF, CNCF Graduated).
4. **kube-proxy modes** — iptables (default), IPVS (better at scale). Know both.
5. **CoreDNS** — the DNS server for Kubernetes. Runs in `kube-system`. Provides service discovery via DNS names.
6. **Service types** — ClusterIP (internal), NodePort (external via node port), LoadBalancer (cloud LB), ExternalName (CNAME).
7. **DNS format** — `service-name.namespace.svc.cluster.local`. Within the same namespace, just the service name works.
