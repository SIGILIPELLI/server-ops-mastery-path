# 01 · Multi-Region / Multi-AZ Architecture Concepts

Level 3's capstone spread a service across two availability zones (AZs)
within one region. This module goes one level up: what changes — and what
gets much harder — when you spread across entire *regions*, and how to
decide whether you actually need to.

## AZ vs. region: what's actually different

An **availability zone** is one or more physically separate data centers
within a region, with independent power, cooling, and networking, but
low-latency (typically sub-millisecond to a few ms) links to other AZs in
the same region. A **region** is a fully independent geographic location
with its own set of AZs — cross-region latency is tens to hundreds of
milliseconds, and regions can fail independently of each other (a regional
control-plane outage, a natural disaster, a submarine cable cut).

```
Region: us-east-1
├── AZ us-east-1a  (data center cluster 1)
├── AZ us-east-1b  (data center cluster 2)  ← ~1-2ms from 1a
└── AZ us-east-1c  (data center cluster 3)

Region: eu-west-1  ← ~80-100ms from us-east-1
├── AZ eu-west-1a
└── AZ eu-west-1b
```

Multi-AZ protects against a data-center-level failure (power, cooling, a
single building's network gear). Multi-region protects against a
region-level failure (control plane outage, natural disaster, a
misconfiguration that takes an entire region's networking down) — a much
rarer event, but one multi-AZ alone does nothing for.

## The cost multi-region adds

Multi-region is not "multi-AZ but bigger" — the higher cross-region latency
changes what's even possible:

- **Synchronous replication becomes impractical.** A database write that
  must be acknowledged by a replica 80ms away adds 80ms+ to every write,
  which is usually unacceptable. Cross-region database replication is
  almost always **asynchronous**, which reintroduces the RPO discussion
  from Level 3 module 03 — some data loss window exists on region failover
  by design, not by mistake.
- **Data residency and consistency get harder.** If both regions can accept
  writes ("active-active"), you need a strategy for conflicting concurrent
  writes to the same record (last-write-wins, CRDTs, or routing all writes
  for a given entity to one "home" region).
- **Operational complexity roughly doubles per region added** — deploys,
  monitoring, on-call runbooks, and IaC all need to account for N regions
  instead of 1, and testing failover between regions (Level 3 module 08) is
  itself a bigger exercise than testing AZ failover.

## Common multi-region patterns

**Active-passive (warm standby)** — one region serves all traffic; a second
region runs a smaller/idle copy of the stack, kept in sync via async
replication, and is promoted only on a regional failure.

```
Region A (active): full traffic, full capacity
Region B (passive): minimal capacity, receiving replicated data,
                     scaled up and promoted only during failover
```

Cheaper (Region B can run at reduced capacity most of the time) but slower
to fail over — Region B needs to scale up before it can absorb full
traffic, which is exactly the RTO number you must measure and rehearse
(Level 3 module 03).

**Active-active** — both regions serve live traffic simultaneously,
typically split by geography via GeoDNS/latency routing (Level 3 module
02). Faster failover (the surviving region is already warm and serving
traffic) but requires solving the write-conflict problem above, and costs
roughly 2x steady-state capacity since each region alone should be able to
absorb the other's traffic if it fails.

```
GeoDNS: EU clients → eu-west-1 (active)
        US clients → us-east-1 (active)
On eu-west-1 failure: GeoDNS reroutes EU clients to us-east-1,
                      which must have headroom to absorb that traffic
```

**Pilot light** — the cheapest pattern: only the data layer is kept
continuously replicated in the second region; compute is provisioned
on-demand only during a real failover (via IaC, Level 3 module 05). Lowest
steady-state cost, longest RTO (you're standing up infrastructure from
scratch during the incident, even if scripted).

## Choosing based on RTO/RPO, not fashion

| Pattern | Relative cost | Typical RTO | Typical RPO |
|---|---|---|---|
| Single region, multi-AZ only | 1x | N/A (no regional failover) | near 0 (sync replication) |
| Pilot light | ~1.1x | hours | minutes (data replicated, compute is not) |
| Active-passive (warm standby) | ~1.3-1.5x | minutes | seconds-minutes |
| Active-active | ~2x | seconds (already serving) | depends on conflict strategy |

This table exists to be used, not admired: pick the row matching a
business-stated RTO/RPO (Level 4 module 05, "Choosing the Right HA
Architecture for a Given SLA," goes deeper on this decision), and don't pay
for active-active's 2x cost if the business need is "we can be down for an
hour once a year" — that's a pilot-light or warm-standby problem.

## Worked example: DNS + async replication topology for warm standby

```
# Route 53-style failover DNS (Level 3 module 02) between two regions
app.example.com → failover routing
  primary:   us-east-1 ALB   (health check every 30s)
  secondary: eu-west-1 ALB   (only served if primary fails)
```

```sql
-- Postgres logical replication (async) from primary region to standby region
-- On the primary (us-east-1):
CREATE PUBLICATION app_pub FOR ALL TABLES;

-- On the standby (eu-west-1), pointing back at the primary over a VPN/peering link:
CREATE SUBSCRIPTION app_sub
  CONNECTION 'host=primary.us-east-1.internal dbname=app user=replicator'
  PUBLICATION app_pub;
```

```bash
# Monitor replication lag — this number IS your effective RPO, continuously
psql -h eu-west-1-standby -c \
  "SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;"
```

If `replication_lag` regularly sits at 45 seconds under normal load, the
system's real RPO is ~45 seconds (plus whatever additional lag a failure
scenario introduces), regardless of what a design document claims — measure
it, don't assume it.

## Exercise

1. For a system you know (real or from an earlier module's capstone), write
   down its current failure domain: single-server, multi-AZ, or
   multi-region. Identify one specific event that would take the whole
   thing down given its current domain.
2. Pick a target pattern (pilot light, warm standby, or active-active) for
   that system, justified by a stated RTO/RPO, and sketch the DNS +
   replication topology needed to support it.
3. If you have access to two regions/locations (even two VMs standing in
   for regions), set up async replication between them and measure actual
   replication lag under a realistic write load — compare it to the RPO you
   claimed in step 2.
