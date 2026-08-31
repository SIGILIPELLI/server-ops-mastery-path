# 10 · Capstone — Highly Available Web Service Design

This capstone ties together every Level 3 module into one coherent design:
a web service that survives a single instance failing, a single AZ failing,
and data loss — with the reasoning behind each decision, not just a
diagram. Treat it as a template for how to *think through* an HA design,
which you can then apply to your own systems.

## The scenario

A checkout service: `web` (stateless API) → `orders` (stateless) → calls
`payments` (external) and `inventory` (internal) → backed by a Postgres
database. Target: 99.95% availability (module 01's table: ~22 minutes of
downtime tolerated per month), RPO of 5 minutes, RTO of 30 minutes (module
03).

## Step 1 — walk the request path and eliminate SPOFs (module 01)

```
Internet
  │
  ▼
DNS (app.example.com, health-checked failover) ── module 02
  │
  ▼
Load balancer pair (VIP via keepalived, or managed cloud LB) ── module 01
  │
  ▼
web × 3 instances, spread across 2 AZs ── module 01
  │
  ▼
orders × 3 instances, spread across 2 AZs
  │              │
  ▼              ▼
payments      inventory
(external,    (internal service,
 has its own   own HA — out of
 SLA)          scope here)
  │
  ▼
Postgres primary + 1 sync replica (different AZ) ── module 03
  + nightly logical backup shipped off-region
```

Each layer is redundant (≥2 of everything, spread across failure domains)
and each layer has a health check feeding the layer above it, so failure at
any single point is detected and routed around automatically without a
human paged at 3am for the common case.

## Step 2 — health checks per layer (module 01)

```nginx
# web tier readiness: process up AND can reach `orders`
location /readyz {
    # a small script/endpoint that checks orders connectivity, not just "process alive"
    proxy_pass http://127.0.0.1:8081/internal/readyz;
}
```

```
upstream orders_backend {
    least_conn;
    server 10.0.1.11:8080 max_fails=3 fail_timeout=15s;
    server 10.0.1.12:8080 max_fails=3 fail_timeout=15s;
    server 10.0.2.13:8080 max_fails=3 fail_timeout=15s;  # different AZ (10.0.2.x)
}
```

Splitting instance IPs across `10.0.1.x` and `10.0.2.x` subnets (mapped to
two AZs) means `max_fails` handles single-instance loss and an AZ-level
health signal (all of `10.0.1.x` unreachable at once) is a distinguishable
event the on-call runbook can recognize immediately, rather than looking
identical to three unrelated instance failures.

## Step 3 — DNS and cross-AZ failover (module 02)

```
app.example.com → failover routing policy
  primary:   LB VIP in AZ-a  (health check: GET /healthz every 30s, 3 failures = unhealthy)
  secondary: LB VIP in AZ-b  (only answered if primary fails health check)
TTL: 60s
```

The 60s TTL is a deliberate trade-off: low enough that a full AZ failure
recovers within the RTO budget once combined with the LB pair's own
keepalived failover (seconds) handling single-node loss within an AZ —
DNS failover is the second line of defense for "the entire AZ housing the
primary LB is gone," not the first line for ordinary instance churn.

## Step 4 — backup and DR (module 03)

```bash
# nightly logical backup + off-region copy (from module 03), sized to a 5-minute RPO
# alone this only gives RPO = 24h — so pair it with continuous WAL shipping:
archive_command = 'aws s3 cp %p s3://company-backups-offregion/wal/%f'
```

```
RPO achieved: ~5 min (continuous WAL archiving to off-region object storage)
RTO plan:
  1. Promote the sync replica in AZ-b if the primary AZ is lost           (~5 min, automated via Patroni or manual runbook)
  2. If BOTH AZs are lost (full-region event): restore from last full
     backup + replay WAL from off-region storage into a fresh instance    (~25 min, tested quarterly per module 03's DR drill schedule)
```

The two-tier plan matters: promoting a same-region replica is fast and
handles the much more common "one AZ down" case; full restore-from-backup
is the slower fallback for the rare "region gone" case, and its 25-minute
estimate is only trustworthy because it's been rehearsed (module 03's "test
your restores" rule), not assumed from the architecture diagram.

## Step 5 — IaC for the whole thing (module 05) and drift checking (module 06)

```
terraform/
├── network.tf       # VPC, subnets across 2 AZs, security groups
├── lb.tf             # LB pair / cloud LB + health check config
├── compute.tf        # web + orders instance groups, spread across AZs
├── database.tf       # Postgres primary + replica, WAL archiving config
└── backend.tf         # remote state (S3 + DynamoDB lock)
```

```yaml
# nightly CI job: catch drift before it undermines the DR plan above
- run: terraform plan -detailed-exitcode
```

Every resource in the diagram above is declared in Terraform, so the DR
plan's "rebuild in a new region" step is "run this Terraform against a new
region's provider config," not a from-memory manual rebuild — this is
exactly what makes the 25-minute full-restore RTO estimate credible.

## Step 6 — validate the design with chaos engineering (module 08)

```
Game day plan for this system:
1. Kill one `orders` instance → expect: LB removes it within fail_timeout,
   error rate stays near zero.
2. Kill ALL `orders` instances in one AZ → expect: LB routes 100% to the
   other AZ, latency may rise (fewer instances) but no errors.
3. Stop the Postgres primary → expect: replica promotes within ~1-2 min
   (Patroni), a brief write-unavailability window during promotion — is
   this within the RTO budget? If not, the design needs a faster failover
   mechanism, not just a documented one.
4. Inject 1s latency to the `payments` call (module 07's tc/netem trick) →
   expect: `orders`' timeout/circuit breaker trips before it cascades into
   `orders` itself becoming unresponsive.
```

Only after running these and confirming the *actual* behavior matches
these expectations does the design get to claim it meets 99.95%/RPO
5min/RTO 30min — the numbers in a design doc are a hypothesis until a game
day (or a real incident) confirms them.

## Step 7 — observe it (module 09)

```
Dashboards: RED metrics for web/orders (rate, errors, p50/p95/p99 latency),
            USE metrics for the Postgres primary (connections, disk, replication lag)
Alerting:   error rate > 1% for 5 min → page; replication lag > 30s → page
            (30s threshold chosen because it's meaningfully below the 5-min
             RPO target — an alert should fire well before RPO is at risk)
Tracing:    request_id propagated web → orders → payments/inventory,
            so a checkout failure can be traced to the exact failing hop
```

## Putting it together: the design review checklist

Before calling any HA design "done," it should be able to answer every one
of these, specifically (not "yes, generally"):

- [ ] Every layer has ≥2 redundant instances across ≥2 failure domains
- [ ] Every layer has a real (readiness, not just liveness) health check
- [ ] DNS failover path exists for the "whole primary path is gone" case,
      with a TTL chosen deliberately, not left at a default
- [ ] Backups exist, are off-site, and have been *restored* successfully
      within the target RTO, not just "backup job succeeds"
- [ ] The IaC actually describes 100% of what's running — verified by a
      clean `terraform plan`, not assumed
- [ ] At least one chaos/game-day experiment has been run against each
      major failure mode, with results matching the design's claims
- [ ] Dashboards and alerts exist for the RED/USE metrics that would
      surface a violation of the RPO/RTO/availability targets before a
      customer notices

## Exercise

1. Take a real (or realistic toy) service you run or have access to, and
   produce the same seven-step design for it: SPOF elimination, health
   checks, DNS/failover, backup/DR with RPO/RTO numbers, IaC coverage, a
   chaos game-day plan, and an observability plan.
2. Run at least the "kill one instance" and "kill one AZ's worth of
   instances" game-day experiments from step 6 against a real or toy
   deployment, and record actual measured behavior against your stated
   expectations.
3. Fill out the design review checklist above honestly — for every
   unchecked box, write one sentence on what concrete work would close the
   gap.
