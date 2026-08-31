# 10 · Capstone — Designing a Resilient Global Platform

This is the final capstone of the entire program. Level 3's capstone
designed one highly-available service in one region. This capstone extends
that design across regions, under an explicit SLA, with cost discipline,
compliance readiness, sustainable on-call, and a platform layer that lets
other teams build on it safely — pulling together every module from all
four levels into one architecture with the reasoning made explicit at each
decision point.

## The scenario

A global e-commerce checkout platform. Contractual SLA: **99.95% monthly
uptime**, RPO 5 minutes, RTO 15 minutes, serving customers in North America
and Europe, with a compliance requirement (PCI-DSS scope for payment data)
and a platform team supporting several product teams building on shared
infrastructure.

## Decision 1 — architecture tier, from the SLA (Level 4 module 05)

```
99.95% target → ~22 min/month downtime budget
Dependency audit: cloud provider's own regional SLA is 99.99% per region,
                  DNS provider's SLA is 99.999% — neither is the bottleneck
Decision: multi-region WARM STANDBY (not full active-active) —
  active-active's ~2x cost isn't justified by this SLA tier; warm standby's
  achievable RTO (minutes, once rehearsed) comfortably fits the 15-min
  budget, at roughly half the steady-state cost of active-active.
```

This decision is stated first and drives everything else — it is the
answer to "how much are we willing to spend to hit this specific number,"
not a default reached for because it's the most sophisticated option
available (Level 4 module 05's central lesson).

## Decision 2 — regional topology (Level 4 module 01, Level 3 capstone)

```
Primary region: us-east-1
  ├── AZ a, AZ b — full HA web/orders/inventory tiers (Level 3 capstone shape)
  └── Postgres primary + sync replica (cross-AZ)

Standby region: eu-west-1
  ├── AZ a — reduced-capacity web/orders/inventory (warm, not idle)
  └── Postgres async replica (cross-region, replication lag monitored —
      Level 4 module 01's "measure it, don't assume it")

DNS: failover routing, health-checked, TTL 60s (Level 3 module 02)
  primary:   us-east-1 regional LB
  secondary: eu-west-1 regional LB (promoted on primary region failure)
```

European customers are served from `eu-west-1` in steady state via
latency-based routing layered on top of the failover policy — this
incidentally also helps meet EU data-residency expectations for the
compliance layer below, a case where the availability design and the
compliance requirement reinforce each other rather than conflicting.

## Decision 3 — capacity and cost (Level 4 modules 02, 06)

```
Little's Law sizing (module 02): 2,000 req/s peak, 150ms avg duration →
  300 concurrent requests → sized fleet with N-1 AZ redundancy → 12
  instances baseline in us-east-1, autoscaling 12-40

Standby region (eu-west-1): sized at ~30% of primary steady-state
  (enough to serve EU latency-routed traffic day-to-day; scales up
  further only on full regional failover — this is the "warm," not
  "cold," part of warm standby)

Cost discipline (module 06):
  - On-demand base capacity pinned to the redundancy floor; autoscaling
    burst capacity uses spot where the workload tolerates interruption
    (stateless web tier only, never the database)
  - Backups (module 03) lifecycle-transitioned to cold storage after 90 days
  - All resources tagged; quarterly right-sizing review scheduled
```

## Decision 4 — data layer and DR (Level 3 module 03, Level 4 module 01)

```
RPO target: 5 minutes
  → continuous WAL shipping (Level 3 module 03) to eu-west-1's replica
  → measured replication lag alerted at 60s (well below the 5-min budget,
    giving room to react before RPO is actually at risk)

RTO target: 15 minutes
  → automated replica promotion via Patroni (not manual — a 15-minute
    budget doesn't comfortably fit a paged human running a manual runbook
    from module 03, so this specific failover step is automated, while the
    surrounding process — declaring the regional failure, communicating
    status — still follows the human incident process)
  → full DR drill (module 03's "test your restores") scheduled quarterly,
    timing the actual promotion + DNS failover + traffic ramp end to end
```

## Decision 5 — security and compliance (Level 4 modules 04, 07)

```
PCI scope: payment data segregated into its own VPC subnet, accessed only
  by the `payments` service via a tightly-scoped security group — no
  other service has network access to the payment data store at all
  (network segmentation, a PCI-specific requirement, module 07)

Patching (module 04): Tier 1 OS patches weekly automated, rolling batches
  with LB drain (module 04's playbook), Tier 4 critical patches out-of-band

Access control (module 07): least-privilege IAM/RBAC per service, audit
  logging (CloudTrail equivalent) shipped to a separate, restricted account

Evidence collection: quarterly automated evidence pull (module 07) covering
  DR drill results, patch compliance %, and access reviews — assembled
  continuously, not scrambled together at audit time
```

## Decision 6 — observability and incident readiness (Level 3 module 09, Level 4 modules 03, 08)

```
RED metrics per service, USE metrics for DB/cache, tracing with
  request_id propagation across web → orders → payments/inventory
  (Level 3 module 09)

Alerting: tuned against the "good page" checklist (module 08) — error
  rate and replication lag are page-worthy; most infra metrics are
  dashboard-only, reviewed, not paged

Runbooks (module 03) exist for: regional failover, database promotion,
  payments provider outage (with circuit breaker per Level 3 module 07)

On-call (module 08): follow-the-sun across the platform team's US and EU
  members, given the platform already spans both regions — escalation
  policy has 3 layers, multi-channel notification in the first layer

Postmortem process (module 03) is blameless and tracked, with action
  items reviewed in the same forum as the error-budget review (module 05)
```

## Decision 7 — chaos validation (Level 3 module 08)

```
Game day schedule, increasing blast radius over successive quarters:
  Q1: kill one instance, one AZ's worth of instances — confirm LB/health
      check behavior matches design
  Q2: promote the database replica manually, mid-business-hours, in
      staging — measure actual RTO against the 15-min target
  Q3: full regional failover drill in production, off-peak, all safety
      rails engaged (module 08's stop-condition discipline) — confirm
      DNS failover + database promotion + traffic ramp meets RTO for real
  Q4: latency injection on the payments dependency — confirm the circuit
      breaker (Level 3 module 07) trips before cascading
```

Only the Q3 drill — a real regional failover, executed in production —
actually proves the 15-minute RTO claim; everything before it builds
toward being able to run that drill safely.

## Decision 8 — the platform layer for other teams (Level 4 module 09)

```
Every design decision above is embedded in a golden-path service template
so a new product team building on this platform inherits it by default:
  - Scaffolded service includes readiness/liveness probes, RED metrics,
    tracing instrumentation, and the security-scanning CI gate (module 04)
  - Terraform module for "new service" enforces tagging (module 06),
    network segmentation defaults (module 07), and wires into the shared
    observability stack (module 09) automatically
  - New service's on-call rotation and runbook are scaffolded from a
    template, not written from a blank page

DevEx metrics (module 09) tracked: deployment frequency, change failure
  rate, and a quarterly platform survey, so the platform team knows
  whether teams are actually using the golden path or working around it
```

## The full design review, end to end

```
[x] SLA and downtime budget stated explicitly, dependency ceiling checked (module 05)
[x] Multi-region topology sized to the SLA, not maximal by default (module 01)
[x] Capacity sized via measured load testing + Little's Law, cost-optimized
    without eroding the redundancy floor (modules 02, 06)
[x] RPO/RTO backed by measured replication lag and a REHEARSED failover,
    not just a design-doc claim (modules 01, 03)
[x] Compliance controls (segmentation, least privilege, audit logging,
    continuous evidence collection) built in, not bolted on before an audit (module 07)
[x] Patching, monitoring, and alerting tuned for signal over noise (modules 04, 08, 09)
[x] Chaos-validated at increasing blast radius, culminating in a real
    production failover drill (module 08)
[x] On-call sustainable (follow-the-sun, multi-layer escalation) and
    blameless (module 08)
[x] Embedded in a self-service platform so the design's guarantees extend
    to every team building on it, not just the team that built it (module 09)
```

## Exercise

1. Using this capstone's structure as a template, produce the same
   eight-decision design for a real or realistic system of your own,
   substituting your own SLA, regions, and compliance requirements.
2. For your design, identify the single most expensive decision (in
   dollars or complexity) and write a one-paragraph justification tying it
   directly to the stated SLA/RTO/RPO — if you can't justify it that way,
   revisit whether it's actually needed (module 05's discipline).
3. Design the Q1-Q4 game day schedule for your system, and actually run at
   least the Q1 experiment (kill one instance / one AZ's worth) against a
   real or toy deployment, recording actual versus expected behavior.
4. Sketch what a golden-path scaffolding template would need to include
   for a new service to inherit your design's guarantees automatically,
   and identify one legitimate case that would need an explicit escape
   hatch from that template.

This closes the four-level path: Level 1 got a service running safely on
one box; Level 2 put it behind a reverse proxy with TLS, staged
environments, and CI/CD; Level 3 made it survive component and AZ failure
with backups and IaC underneath; Level 4 turned it into an
organizationally sound, cost-aware, compliant, multi-region platform other
teams can build on. Every later decision in this program has been building
on the primitives from the ones before it — the same way it should in a
real system.
