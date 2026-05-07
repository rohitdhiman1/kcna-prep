# Cloud Native Roles and Personas

> Understanding the key roles, reliability concepts, and service level terminology in cloud native organizations.

---

## Key Roles in Cloud Native

### Site Reliability Engineer (SRE)

SRE is a discipline that originated at Google. It applies **software engineering practices to operations** problems.

**Core principles:**
- **Reliability is the most important feature.** If a service is not reliable, users cannot use any of its features.
- **Use software engineering to solve operations problems.** Automate everything.
- **Measure everything.** Decisions are data-driven, based on SLIs and SLOs.
- **Embrace risk.** Pursue an optimal balance between reliability and velocity using error budgets.

**Key responsibilities:**
- Define and maintain **SLOs** (service level objectives).
- Manage **error budgets** and use them to balance reliability vs feature velocity.
- Reduce **toil** (manual, repetitive operational work).
- Build and maintain automation, monitoring, and incident response systems.
- Conduct **postmortems** (blameless analysis of incidents).
- On-call rotation for production systems.

---

### DevOps Engineer

DevOps is a **culture and set of practices** that bridges development and operations to deliver software faster and more reliably.

**Core principles:**
- **Collaboration** between development and operations teams.
- **Automation** of build, test, deploy, and infrastructure processes.
- **Continuous improvement** through feedback loops.

**Key responsibilities:**
- Build and maintain **CI/CD pipelines** (Jenkins, GitHub Actions, GitLab CI, Tekton).
- **Infrastructure as Code** (Terraform, Pulumi, Ansible).
- Container image builds and registry management.
- Deployment automation and configuration management.
- Monitoring and alerting setup.

---

### Platform Engineer

Platform Engineering is a newer discipline focused on building **Internal Developer Platforms (IDPs)** — self-service tools that make developers more productive.

**Core principles:**
- **Reduce cognitive load** on developers — they should not need to be Kubernetes experts.
- **Self-service** — developers get what they need through a portal or API, not tickets.
- **Golden paths** — provide recommended, well-supported ways to build and deploy.

**Key responsibilities:**
- Build and maintain the **Internal Developer Platform** (IDP).
- Abstract infrastructure complexity behind simple interfaces.
- Provide developer portals (e.g., Backstage).
- Define templates, best practices, and guardrails.
- Manage shared platform services (CI/CD, observability, security).

---

## Role Comparison

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                  Cloud Native Roles Compared                      │
  │                                                                    │
  │   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │
  │   │     SRE       │  │    DevOps     │  │   Platform    │        │
  │   │   Engineer    │  │   Engineer    │  │   Engineer    │        │
  │   └───────┬───────┘  └───────┬───────┘  └───────┬───────┘        │
  │           │                  │                  │                  │
  │   Focus:  │          Focus:  │          Focus:  │                  │
  │   Reliability       Automation       Developer                    │
  │   & uptime          & delivery       experience                   │
  │                                                                    │
  │   Drives: SLOs,     Drives: CI/CD,  Drives: Self-               │
  │   error budgets,    IaC, build      service platform,            │
  │   incident          pipelines       golden paths,                 │
  │   response                          developer portal              │
  │                                                                    │
  └──────────────────────────────────────────────────────────────────┘
```

| Aspect | SRE | DevOps Engineer | Platform Engineer |
|--------|-----|-----------------|-------------------|
| **Primary focus** | Reliability and availability | Automation and delivery speed | Developer experience |
| **Key output** | SLOs, error budgets, automation | CI/CD pipelines, IaC | Internal Developer Platform |
| **Measures success by** | Uptime, error budget remaining | Deployment frequency, lead time | Developer satisfaction, onboarding time |
| **Works with** | Production systems, monitoring | Build/deploy systems | Developer tools, abstractions |
| **Reduces** | Toil, incidents | Manual processes, silos | Cognitive load on developers |
| **Origin** | Google (2003) | DevOps movement (2008) | Platform engineering (2020s) |

---

## SLIs, SLOs, and SLAs

These three concepts form a hierarchy for measuring and committing to service reliability.

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                               │
  │                        ┌──────────┐                          │
  │                        │   SLA    │  Contractual             │
  │                        │          │  (external promise)      │
  │                        │ 99.9%    │  "If we miss this,       │
  │                        │ uptime   │   we pay penalties"      │
  │                        └────┬─────┘                          │
  │                             │                                │
  │                        ┌────┴─────┐                          │
  │                        │   SLO    │  Target                  │
  │                        │          │  (internal goal)         │
  │                        │ 99.95%   │  "We aim for this"       │
  │                        │ uptime   │                          │
  │                        └────┬─────┘                          │
  │                             │                                │
  │                        ┌────┴─────┐                          │
  │                        │   SLI    │  Measurement             │
  │                        │          │  (actual metric)         │
  │                        │ Request  │  "What we actually       │
  │                        │ latency, │   measure"               │
  │                        │ error    │                          │
  │                        │ rate     │                          │
  │                        └──────────┘                          │
  │                                                               │
  │   SLI feeds SLO, SLO is stricter than SLA                   │
  │                                                               │
  └─────────────────────────────────────────────────────────────┘
```

### SLI (Service Level Indicator)

A **quantitative metric** that measures some aspect of service quality.

**Common SLIs:**
| SLI | What It Measures | Example |
|-----|-----------------|---------|
| Availability | % of successful requests | 99.95% of requests return non-5xx |
| Latency | Response time | 95th percentile latency < 200ms |
| Throughput | Requests processed per second | 1000 req/s sustained |
| Error rate | % of failed requests | < 0.1% errors |
| Durability | Data not lost | 99.999999999% of objects retained |

### SLO (Service Level Objective)

A **target value or range** for an SLI. It defines how reliable the service should be.

Examples:
- "99.95% of requests will complete successfully in a rolling 30-day window."
- "95th percentile latency will be under 200ms."
- "The service will be available 99.9% of the time per quarter."

**Key points:**
- SLOs are **internal goals** — the team sets them.
- SLOs should be **achievable but ambitious**.
- SLOs are typically **stricter than SLAs** (to provide a buffer).
- If the SLO is too aggressive, teams spend all their time on reliability and ship no features.

### SLA (Service Level Agreement)

A **formal contract** between a service provider and a customer that specifies consequences if SLOs are not met.

Examples:
- "99.9% uptime guaranteed. If missed, customer receives 10% service credit."
- "Support tickets responded to within 4 hours."

**Key points:**
- SLAs are **external commitments** — they have legal or financial consequences.
- SLAs are typically **less strict than SLOs** (the team aims higher internally).
- Not every service has an SLA, but every production service should have SLOs.

### The Relationship

```
  SLI:  "We measure request success rate"           (the metric)
  SLO:  "We target 99.95% success rate"             (the goal)
  SLA:  "We guarantee 99.9% to customers"           (the contract)

  SLO is stricter than SLA to provide a safety buffer:
  ┌─────────────────────────────────────────────────────┐
  │                                                       │
  │  100% ──────────────────────────────── Perfect        │
  │         ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓                  │
  │  99.95% ────────────────────────────── SLO target     │
  │         ░░░░░░░░░░  buffer zone  ░░░░                 │
  │  99.9%  ────────────────────────────── SLA threshold   │
  │         xxxx  SLA violation = penalties  xxxx          │
  │  99.0%  ──────────────────────────────                │
  │                                                       │
  └─────────────────────────────────────────────────────┘
```

---

## Error Budgets

An **error budget** is the amount of **allowed unreliability** within an SLO.

### Calculation

```
  Error Budget = 1 - SLO target

  Example: SLO = 99.9% availability
  Error Budget = 1 - 0.999 = 0.1%

  In a 30-day month:
  0.1% of 30 days = 0.03 days = 43.2 minutes of allowed downtime
```

### How Error Budgets Work

```
  ┌─────────────────────────────────────────────────────────┐
  │                   Error Budget Usage                      │
  │                                                           │
  │  Budget remaining ████████████████████████░░░░ 100%       │
  │                                                           │
  │  If budget is available:              If budget is spent: │
  │  ┌──────────────────────┐            ┌─────────────────┐ │
  │  │ Ship new features    │            │ STOP feature    │ │
  │  │ Deploy changes       │            │ development     │ │
  │  │ Experiment           │            │                 │ │
  │  │ Take risks           │            │ Focus entirely  │ │
  │  └──────────────────────┘            │ on reliability  │ │
  │                                      │ improvements    │ │
  │  Velocity ◄──────────────────────►   └─────────────────┘ │
  │  Reliability                                              │
  │                                                           │
  └─────────────────────────────────────────────────────────┘
```

**Key insight:** Error budgets create a **data-driven negotiation** between development velocity and reliability. If the service is too unreliable (budget spent), feature work pauses. If the service is very reliable (budget remaining), teams can take more risks.

---

## Toil

**Toil** is work that is:

1. **Manual** — a human must do it.
2. **Repetitive** — done over and over.
3. **Automatable** — could be done by a machine.
4. **Tactical** — interrupt-driven, not strategic.
5. **No enduring value** — does not improve the system permanently.
6. **Scales linearly with service growth** — more users = more toil.

### Examples of Toil
- Manually restarting a failed service.
- Manually provisioning user accounts.
- Running the same deployment script by hand every release.
- Manually checking logs for errors.
- Copy-pasting configuration between environments.

### Toil Reduction

SRE teams aim to spend **no more than 50% of their time on toil**. The rest should be spent on engineering work that permanently reduces toil.

Strategies:
- **Automate** repetitive tasks (scripts, CI/CD, self-healing).
- **Self-service** tools so developers handle routine tasks themselves.
- **Eliminate** the root cause of recurring operational work.
- Build **platforms** that handle common patterns automatically.

---

## What to Remember for the Exam

1. **SRE** focuses on reliability, error budgets, and toil reduction. Originated at Google.
2. **DevOps** focuses on bridging dev and ops through CI/CD and automation.
3. **Platform Engineering** focuses on building Internal Developer Platforms for self-service.
4. **SLI** = the metric you measure (latency, error rate, availability).
5. **SLO** = the target for an SLI (99.95% availability). Internal goal, set by the team.
6. **SLA** = contractual commitment with penalties. Less strict than SLO.
7. **Error Budget** = 1 - SLO. The allowed unreliability. When spent, feature work pauses; focus shifts to reliability.
8. **Toil** = manual, repetitive, automatable work with no enduring value. SREs aim to keep it below 50%.
9. SLO > SLA in strictness. The buffer protects against SLA violations.
10. Know the difference between the three roles and what each one drives.
