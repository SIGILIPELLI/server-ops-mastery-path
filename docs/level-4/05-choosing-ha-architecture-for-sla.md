# 05 · Choosing the Right HA Architecture for a Given SLA

Levels 3 and 4 have built a full toolbox — multi-AZ, multi-region, warm
standby, active-active, chaos testing. The skill this module teaches is
choosing the *right subset* for a given system, because every one of these
tools costs money and operational complexity, and over-engineering is a
real failure mode, not just under-engineering.

## Start from the SLA, not the technology

An SLA (Service Level Agreement) — or, internally, an SLO (Service Level
Objective) if there's no external contract — states the target in business
terms: "99.9% monthly uptime," "checkout API responds in <500ms p99." The
architecture is derived *from* that target, not chosen first and justified
after.

```
SLA: 99.9% monthly uptime  →  ~43 min downtime/month budget (Level 3 module 01's table)
SLA: 99.99% monthly uptime →  ~4.4 min downtime/month budget
```

The gap between these two numbers is roughly 10x — and the infrastructure
needed to close that gap is not 10% more expensive, it's usually a
qualitatively different architecture (multi-AZ isn't enough for 99.99% if
your provider's regional control plane itself has historically had more
downtime than your budget allows).

## The error budget: SLOs give you permission to take risks, on purpose

An **error budget** is the flip side of an SLO: if the SLO is 99.9%
uptime, the error budget is the 0.1% of time you're *allowed* to be down or
degraded — and that budget is meant to be spent, not hoarded.

```
Monthly error budget at 99.9% SLO = 43.2 minutes

Spent this month:
  - 12 min: planned maintenance window (announced, low-traffic time)
  - 8 min: incident (payments provider outage, Level 4 module 03)
  - Remaining budget: 23.2 minutes
```

The practical use of an error budget: if it's healthy (mostly unspent),
the team can justify shipping faster and taking more risk (a bigger
migration, a less-tested feature flag rollout) — if it's exhausted,
that's the trigger to freeze risky changes and invest in reliability work
instead, a decision rule instead of a political argument. This ties
directly to whether more HA investment (multi-region, active-active) is
even justified: an SLO that's comfortably met month after month with
budget to spare doesn't need a more expensive architecture; a chronically
overspent budget does.

## Mapping SLA tiers to architecture

| SLA target | Downtime/month | Reasonable architecture | Notes |
|---|---|---|---|
| 99% | ~7.3 hrs | Single server + backups (Level 1-2) | Fine for internal tools, low-traffic side projects |
| 99.9% | ~43 min | Multi-AZ, redundant instances, LB health checks (Level 3 module 01) | Typical "production web app" baseline |
| 99.95% | ~22 min | Multi-AZ + automated failover (DB replica promotion, module 01) | Needs tested runbooks (module 03), not just architecture |
| 99.99% | ~4.4 min | Multi-region warm standby or active-active (Level 4 module 01) | Needs automated (not manual) regional failover to hit the RTO |
| 99.999% | ~26 sec | Active-active multi-region, no single dependency (including your DNS/cert provider) can exceed the budget alone | Extremely expensive; justify hard before committing |

Two things to check before committing to a row: **can any single
dependency alone blow the budget** (if your DNS provider's own historical
uptime is 99.95%, your system cannot exceed 99.95% no matter what you build
on top of it — audit every dependency's own SLA against your target), and
**is the target actually driven by a real business/contractual need**, not
aspiration.

## Cost is not linear — it's roughly exponential per nine

```
99.9%  → 99.95%:  ~2x downtime reduction, roughly +20-40% infra/ops cost
                   (add automated failover, more redundancy)
99.95% → 99.99%:  ~5x downtime reduction, roughly +80-150% infra/ops cost
                   (multi-region, likely doubling steady-state compute)
99.99% → 99.999%: ~10x downtime reduction, often 3x+ cost and a dedicated
                   reliability engineering investment
                   (active-active, chasing every dependency's own SLA)
```

These multipliers are illustrative, not universal constants — but the
*shape* (accelerating cost per nine) holds broadly, and is the concrete
argument against defaulting to the most resilient architecture available:
each additional nine must be justified against what it actually buys the
business, not assumed to be free because "more reliable is always better."

## A worked decision: three example systems, three different answers

**Internal admin dashboard, used by 20 employees during business hours.**
SLA: 99% is generous — a few hours of downtime a month is a minor
inconvenience, not a business risk. Architecture: single instance with
automated restarts (systemd, Level 1) and nightly backups. Multi-AZ would
be pure overhead here.

**Customer-facing checkout API, revenue-generating, global customers.**
SLA: 99.95%, justified by revenue-per-minute of downtime exceeding the cost
of the redundancy. Architecture: multi-AZ (Level 3 capstone), automated DB
failover, tested runbooks, chaos-tested. Multi-region active-active would
be over-engineering unless the revenue-per-minute number is large enough to
justify roughly doubling infra cost — model that number explicitly before
deciding.

**Payment processing core for a fintech, regulated, contractual 99.99%
SLA with financial penalties for breach.** Architecture: multi-region
active-active, or at minimum automated warm standby with a proven
sub-5-minute RTO (Level 4 module 01), because the SLA is contractual and
the cost of the penalty plus reputational damage clearly exceeds the
doubled infrastructure cost.

The same engineering toolbox, three different correct answers — the
determining factor is the actual cost of downtime for that specific system,
not which architecture is "best practice" in the abstract.

## A decision framework to apply

```
1. State the SLA/SLO explicitly (get it from the business, don't invent it)
2. Compute the downtime budget it implies (Level 3 module 01's table)
3. List every dependency in the request path and its own historical/stated
   uptime — the system's ceiling is the worst of these, not your own design
4. Estimate the cost of downtime (revenue, reputation, contractual penalty)
   per minute/hour, even roughly — this is what justifies the next line
5. Pick the cheapest architecture tier (from the mapping table) that meets
   the budget from step 2, using the dependency ceiling from step 3 as a
   sanity check
6. Validate with chaos testing (Level 3 module 08) — does the chosen
   architecture actually deliver the numbers, measured, not assumed
7. Track the error budget over time and revisit the decision when it's
   chronically over or comfortably under spent
```

## Exercise

1. For three systems you know (or invent three realistic ones spanning
   internal tool / customer-facing / regulated), state an SLA for each and
   compute its monthly downtime budget.
2. For one of them, list every dependency in its request path (DNS
   provider, cloud provider, third-party APIs) and find or estimate each
   one's own uptime track record — identify whether any of them alone
   would already blow the target SLA.
3. Using the decision framework's steps 4-5, pick and justify an
   architecture tier for that system, including an explicit (even if rough)
   cost-of-downtime estimate that justifies the choice over a cheaper or
   more expensive alternative.
4. Define an error budget for that system's SLO and describe, in concrete
   terms, what policy change would kick in if the budget were exhausted two
   months in a row.
