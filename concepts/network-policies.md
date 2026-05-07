# Network Policies

> By default, all pods in a Kubernetes cluster can communicate with all other pods. Network Policies let you restrict this traffic using firewall-like rules.

---

## Default Behavior

Without any NetworkPolicy, Kubernetes allows **all** traffic between all pods — no restrictions.

```
  Default: All-Open Network

  Namespace: default              Namespace: production
  ┌───────────────────┐           ┌───────────────────┐
  │                   │           │                   │
  │  Pod A ◄─────────►│───────────│──► Pod C          │
  │                   │           │                   │
  │  Pod B ◄─────────►│───────────│──► Pod D          │
  │                   │           │                   │
  └───────────────────┘           └───────────────────┘

  Every pod can talk to every other pod.
  Every pod can receive traffic from every other pod.
  No restrictions at all.
```

This is fine for simple setups but is a **security risk** in production. If an attacker compromises one pod, they can reach every other pod in the cluster.

---

## NetworkPolicy Resource

A NetworkPolicy is a Kubernetes resource that selects pods and defines allowed ingress (incoming) and/or egress (outgoing) traffic.

### Key Concepts

```
  NetworkPolicy Structure

  ┌────────────────────────────────────────────────────┐
  │                  NetworkPolicy                      │
  │                                                     │
  │  1. podSelector      "Which pods does this apply   │
  │                       to?"                          │
  │                                                     │
  │  2. policyTypes       "Ingress, Egress, or both?"  │
  │                                                     │
  │  3. ingress rules     "Who can send traffic IN?"   │
  │     - from:                                         │
  │       - podSelector   (specific pods)               │
  │       - namespaceSelector (specific namespaces)     │
  │       - ipBlock       (CIDR ranges)                 │
  │     - ports:          (specific ports)              │
  │                                                     │
  │  4. egress rules      "Where can traffic go OUT?"  │
  │     - to:                                           │
  │       - podSelector                                 │
  │       - namespaceSelector                           │
  │       - ipBlock                                     │
  │     - ports:                                        │
  │                                                     │
  └────────────────────────────────────────────────────┘
```

### Important Rules

- **NetworkPolicies are additive** — if multiple policies select the same pod, the union of all rules applies. There is no deny rule; you control access by what you allow.
- **If any NetworkPolicy selects a pod**, then **only** traffic matching a rule is allowed. Everything else is denied.
- **If no NetworkPolicy selects a pod**, all traffic is allowed (default behavior).
- NetworkPolicies are **namespace-scoped**.

---

## Default Deny Policies

The most common starting point: deny all traffic, then allow only what you need.

### Default Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}          # Selects ALL pods in the namespace
  policyTypes:
  - Ingress                # No ingress rules = deny all incoming
```

### Default Deny All Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress                 # No egress rules = deny all outgoing
```

### Default Deny All (Both Directions)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

---

## Example Policies

### Allow Traffic From Specific Pods

Only allow the `frontend` pods to talk to the `backend` pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend            # This policy applies to backend pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend       # Allow traffic FROM frontend pods
    ports:
    - protocol: TCP
      port: 8080              # Only on port 8080
```

```
  Result:

  ┌─────────────────────── production ───────────────────────┐
  │                                                           │
  │  frontend pods          backend pods          db pods     │
  │  ┌──────────┐          ┌──────────┐         ┌────────┐  │
  │  │ app:     │──(8080)─►│ app:     │    X    │ app:   │  │
  │  │ frontend │  ALLOW   │ backend  │◄──(X)───│ db     │  │
  │  └──────────┘          └──────────┘  DENIED └────────┘  │
  │                              ▲                           │
  │                              │                           │
  │          any other pod ──────┘ DENIED                    │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

### Allow Traffic From a Specific Namespace

Allow monitoring namespace to scrape metrics from all pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector: {}             # Applies to ALL pods in production
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: monitoring  # Allow from namespaces labeled purpose=monitoring
    ports:
    - protocol: TCP
      port: 9090              # Metrics port only
```

### Allow Traffic From a CIDR Block

Allow external traffic from a specific IP range:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 203.0.113.0/24
        except:
        - 203.0.113.5/32      # Block this specific IP
    ports:
    - protocol: TCP
      port: 443
```

### Allow DNS Egress (Critical!)

When you set a default deny egress policy, **pods can no longer resolve DNS**. You must explicitly allow DNS:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}    # Any namespace (kube-system for CoreDNS)
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

---

## CNI Plugin Requirement

**NetworkPolicies are only enforced if the CNI plugin supports them.** The Kubernetes API will accept the resource, but nothing will happen if the CNI doesn't implement it.

```
  CNI Plugin Support for NetworkPolicy

  ┌──────────────┬─────────────────────────┐
  │   Plugin     │  NetworkPolicy Support  │
  ├──────────────┼─────────────────────────┤
  │  Calico      │  YES                    │
  │  Cilium      │  YES                    │
  │  Weave Net   │  YES                    │
  │  Canal       │  YES (Calico component) │
  │  Flannel     │  NO (alone)             │
  │  Flannel +   │  YES (Calico handles    │
  │   Calico     │   the policies)         │
  └──────────────┴─────────────────────────┘

  IMPORTANT: Flannel by itself does NOT enforce
  NetworkPolicies. The YAML is accepted but ignored.
```

---

## Combining Selectors: AND vs OR

Understanding how `from` rules combine is critical:

```yaml
# OR logic — two separate items in the from array
ingress:
- from:
  - podSelector:             # Rule 1: pods with app=frontend
      matchLabels:
        app: frontend
  - namespaceSelector:       # Rule 2: any pod in namespace with team=ops
      matchLabels:
        team: ops
```
Traffic is allowed from frontend pods **OR** from any pod in the ops namespace.

```yaml
# AND logic — combined in one item
ingress:
- from:
  - podSelector:             # Must match BOTH conditions:
      matchLabels:           #   pod has app=frontend
        app: frontend        #   AND
    namespaceSelector:       #   pod is in namespace with team=ops
      matchLabels:
        team: ops
```
Traffic is allowed only from frontend pods **that are in the ops namespace**.

```
  AND vs OR — Visual

  OR (two items in from[]):
  ┌──────────────┐
  │ app=frontend │──► ALLOW
  └──────────────┘
  ┌──────────────┐
  │ ns: team=ops │──► ALLOW
  └──────────────┘
  (either one is sufficient)

  AND (one item with both selectors):
  ┌──────────────────────────────┐
  │ app=frontend AND ns: team=ops│──► ALLOW
  └──────────────────────────────┘
  (both must be true)
```

---

## Key Exam Points

- **Default behavior**: All pods can communicate with all pods. No restrictions.
- **NetworkPolicy selects pods** via `podSelector`. Once selected, only explicitly allowed traffic is permitted.
- **Three selector types**: `podSelector`, `namespaceSelector`, `ipBlock`.
- **Policies are additive** — no explicit deny rules. You control by what you allow.
- **Default deny** = empty `podSelector: {}` + policyType with no rules.
- **CNI must support it** — Calico and Cilium do. Flannel alone does not.
- **AND vs OR** — two separate `from` items = OR. Combined selectors in one item = AND.
- **Egress DNS** — if you deny all egress, pods lose DNS. Always allow port 53 egress.

---

## What to Remember for the Exam

1. **Default = allow all**. NetworkPolicy = restrict. No NetworkPolicy = no restriction.
2. **podSelector: {}** selects ALL pods in the namespace.
3. **Three selectors**: podSelector (which pods), namespaceSelector (which namespaces), ipBlock (which CIDRs).
4. **CNI support is required** — Flannel alone cannot enforce NetworkPolicies. Calico and Cilium can.
5. **Default deny pattern**: Create a policy with podSelector: {} and the policyType but no rules.
6. **AND vs OR logic** — single from item = AND. Multiple from items = OR. This is a common exam question.
7. **NetworkPolicies are namespace-scoped** — they only affect pods in their namespace.
8. **Don't forget DNS** — when you deny egress, explicitly allow UDP/TCP port 53 or pods can't resolve names.
