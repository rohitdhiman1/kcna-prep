# Deployment Strategies

**Deployment strategies define how new versions of an application are rolled out to replace old versions, each offering different trade-offs between safety, speed, and resource cost.**

---

## Table of Contents

- [Overview](#overview)
- [Rolling Update](#rolling-update)
- [Blue-Green Deployment](#blue-green-deployment)
- [Canary Deployment](#canary-deployment)
- [Recreate Strategy](#recreate-strategy)
- [A/B Testing](#ab-testing)
- [Comparison Table](#comparison-table)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## Overview

When you deploy a new version of an application, you need a strategy to transition from the old version to the new one. The right strategy depends on your tolerance for downtime, risk appetite, and infrastructure budget.

```
  Old Version (v1)  ──── Strategy ────>  New Version (v2)

  Strategies vary in:
    - Downtime:       Does the app go offline?
    - Risk:           What if v2 is broken?
    - Resource cost:  Do you need double the infrastructure?
    - Rollback speed: How fast can you revert?
```

---

## Rolling Update

**The default strategy in Kubernetes.** Pods running the old version are gradually replaced with pods running the new version. At no point are all old pods removed before new ones are ready.

### How It Works

```
Time ──────────────────────────────────────────────────>

Step 1: All pods are v1
+--------+ +--------+ +--------+ +--------+
|  v1    | |  v1    | |  v1    | |  v1    |
+--------+ +--------+ +--------+ +--------+
     ^          ^          ^          ^
     └──────────┴──────────┴──────────┘
                    |
               [ Service ]
                    |
               [ Traffic ]

Step 2: One new v2 pod starts, one v1 pod terminates
+--------+ +--------+ +--------+ +--------+
|  v1    | |  v1    | |  v1    | |  v2    |  <-- new
+--------+ +--------+ +--------+ +--------+
                                  (v1 terminating)

Step 3: Continues replacing...
+--------+ +--------+ +--------+ +--------+
|  v1    | |  v1    | |  v2    | |  v2    |
+--------+ +--------+ +--------+ +--------+

Step 4: All pods are v2
+--------+ +--------+ +--------+ +--------+
|  v2    | |  v2    | |  v2    | |  v2    |
+--------+ +--------+ +--------+ +--------+
```

### Key Parameters

| Parameter       | Description                                      | Default |
|-----------------|--------------------------------------------------|---------|
| `maxSurge`      | Max extra pods above desired count during update  | 25%     |
| `maxUnavailable`| Max pods that can be unavailable during update    | 25%     |

**Example:** With 4 replicas, `maxSurge=1`, `maxUnavailable=1`:
- At most 5 pods running (4 + 1 surge)
- At least 3 pods available (4 - 1 unavailable)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

### Pros and Cons

- **Pros:** Zero downtime, no extra infrastructure needed, built into Kubernetes
- **Cons:** Slow rollout, both versions serve traffic simultaneously during update, rollback requires another rolling update

---

## Blue-Green Deployment

Two identical environments exist: **Blue** (current production) and **Green** (new version). Traffic is switched instantly by changing the Service selector.

### How It Works

```
BEFORE SWITCH:
                              +---------------------+
                              |   Green (v2) - idle  |
                              |  +----+ +----+       |
                              |  | v2 | | v2 |       |
                              |  +----+ +----+       |
                              +---------------------+
                                    (no traffic)

   Users                      +---------------------+
     |                        |   Blue (v1) - active |
     |        +----------+    |  +----+ +----+       |
     +------->| Service  |--->|  | v1 | | v1 |       |
              | selector:|    |  +----+ +----+       |
              | app: v1  |    +---------------------+
              +----------+

AFTER SWITCH (instant):

                              +---------------------+
                              |   Blue (v1) - idle   |
                              |  +----+ +----+       |
                              |  | v1 | | v1 |       |
                              |  +----+ +----+       |
                              +---------------------+
                                    (no traffic)

   Users                      +---------------------+
     |                        |   Green (v2) - active|
     |        +----------+    |  +----+ +----+       |
     +------->| Service  |--->|  | v2 | | v2 |       |
              | selector:|    |  +----+ +----+       |
              | app: v2  |    +---------------------+
              +----------+
```

### In Kubernetes

1. Deploy `v1` with label `version: blue`
2. Service selector points to `version: blue`
3. Deploy `v2` with label `version: green`
4. Test `v2` internally
5. Update Service selector to `version: green` — instant switch
6. Keep `v1` around for quick rollback

### Pros and Cons

- **Pros:** Instant rollback (switch selector back), zero downtime, full testing of new version before switch
- **Cons:** Requires double the infrastructure, database migrations can be tricky

---

## Canary Deployment

A small percentage of traffic is routed to the new version first. If it behaves well, traffic is gradually shifted until 100% goes to the new version.

### How It Works

```
Phase 1: 10% canary
                     +----------+
   Users ──────────> | Service  |
                     +----+-----+
                          |
              +-----------+-----------+
              |                       |
    +---------+--------+    +---------+--------+
    |  v1 (9 replicas) |    |  v2 (1 replica)  |
    |  +--+ +--+ +--+  |    |  +--+             |
    |  |v1| |v1| |v1|  |    |  |v2|  <-- canary |
    |  +--+ +--+ +--+  |    |  +--+             |
    |  +--+ +--+ +--+  |    +------------------+
    |  |v1| |v1| |v1|  |
    |  +--+ +--+ +--+  |          ~10% traffic
    |  +--+ +--+ +--+  |
    |  |v1| |v1| |v1|  |
    |  +--+ +--+ +--+  |
    +------------------+
          ~90% traffic

Phase 2: 50% canary (scale up v2, scale down v1)
              +-----------+-----------+
              |                       |
    +---------+--------+    +---------+--------+
    |  v1 (5 replicas) |    |  v2 (5 replicas) |
    +------------------+    +------------------+
          ~50%                    ~50%

Phase 3: 100% — full rollout
    +------------------+
    |  v2 (10 replicas)|
    +------------------+
          100%
```

### In Kubernetes (Simple Approach)

Both deployments use the **same label** (e.g., `app: myapp`) so the Service routes to both. Traffic split is controlled by replica ratio.

For more precise traffic splitting (e.g., exactly 5%), use a **service mesh** like Istio or a tool like **Flagger**.

### Pros and Cons

- **Pros:** Low risk (only a fraction of users see v2), real production testing, gradual confidence building
- **Cons:** Slow rollout, requires monitoring to detect issues, basic K8s only offers coarse traffic splitting (by replica count)

---

## Recreate Strategy

All old pods are terminated **first**, then all new pods are created. This causes downtime but is the simplest strategy.

### How It Works

```
Step 1: All v1 pods running
+--------+ +--------+ +--------+
|  v1    | |  v1    | |  v1    |
+--------+ +--------+ +--------+
     |          |          |
     └──────────┴──────────┘
                |
           [ Service ]

Step 2: All v1 pods terminated (DOWNTIME!)
+--------+ +--------+ +--------+
|  xxxx  | |  xxxx  | |  xxxx  |
+--------+ +--------+ +--------+

           [ Service ]  -->  NO BACKENDS!

Step 3: All v2 pods created
+--------+ +--------+ +--------+
|  v2    | |  v2    | |  v2    |
+--------+ +--------+ +--------+
     |          |          |
     └──────────┴──────────┘
                |
           [ Service ]
```

```yaml
spec:
  strategy:
    type: Recreate
```

### Pros and Cons

- **Pros:** Simple, no version mixing (good if old and new are incompatible), no extra resources
- **Cons:** Causes downtime, not suitable for production-critical services

---

## A/B Testing

A/B testing routes different user segments to different versions based on criteria like HTTP headers, cookies, geographic location, or user agent. The goal is to **measure user behavior** differences between versions.

```
                     +------------------+
   Users ──────────> | Ingress / Mesh   |
                     +--------+---------+
                              |
               +--------------+--------------+
               |                             |
    (header: group=A)              (header: group=B)
               |                             |
       +-------+-------+           +--------+--------+
       |  Version A    |           |  Version B      |
       |  (control)    |           |  (experiment)   |
       +---------------+           +-----------------+
```

A/B testing is **not a built-in Kubernetes strategy**. It requires an Ingress controller or service mesh (like Istio) that can route based on request attributes.

**Key distinction:** Canary is about **safely rolling out** a new version. A/B testing is about **comparing user experiences** to make product decisions.

---

## Comparison Table

| Strategy        | Downtime | Rollback Speed | Resource Cost | Risk Level | K8s Native? |
|-----------------|----------|----------------|---------------|------------|-------------|
| **Rolling Update** | None  | Slow (re-roll) | Low           | Medium     | Yes (default) |
| **Blue-Green**     | None  | Instant        | High (2x)     | Low        | Manual (labels) |
| **Canary**         | None  | Fast           | Medium        | Low        | Manual (replicas) |
| **Recreate**       | Yes   | Slow           | Low           | High       | Yes         |
| **A/B Testing**    | None  | Fast           | Medium        | Low        | No (needs mesh/ingress) |

---

## What to Remember for the Exam

1. **Rolling Update is the default** Kubernetes deployment strategy. It uses `maxSurge` and `maxUnavailable` to control the pace of the rollout.

2. **maxSurge** = how many extra pods can exist above the desired count. **maxUnavailable** = how many pods can be down during the update.

3. **Blue-Green** uses two full environments and switches traffic by changing the Service selector. Requires double the resources but enables instant rollback.

4. **Canary** sends a small percentage of traffic to the new version first. In basic Kubernetes, the traffic split is proportional to the replica count.

5. **Recreate** kills all old pods before starting new ones. Simple but causes **downtime**.

6. **A/B testing** is about comparing user experiences across versions. It requires a service mesh or smart ingress — it is NOT a native Kubernetes deployment strategy.

7. Use `kubectl rollout status` to monitor a rolling update and `kubectl rollout undo` to roll back.

8. **Canary vs A/B testing:** Canary is about safe rollout; A/B testing is about measuring user behavior differences.
