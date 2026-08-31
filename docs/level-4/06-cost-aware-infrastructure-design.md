# 06 · Cost-Aware Infrastructure Design

Module 05 argued that reliability has a real cost curve, and that
over-engineering is a genuine failure mode. This module goes deeper on the
cost side specifically: where infrastructure spend actually goes, how to
find waste, and how to design so cost stays proportional to actual need as
a system grows.

## Where the money actually goes

```
Typical cloud spend breakdown for a mid-size web service:
  Compute (VMs/containers):        40-55%
  Storage (block, object, backups): 10-20%
  Data transfer (egress especially): 5-15%
  Managed services (DB, cache, queue): 15-25%
  Observability/logging stack:       5-15%   (often underestimated — see below)
```

Two line items surprise teams most often: **data egress** (traffic leaving
the cloud provider's network — to the internet, or to another region —
is billed and often the most expensive per-GB item, while intra-AZ traffic
is usually free or nearly so), and **observability** (Level 3 module 09's
metrics/logs/traces stack, at high volume with no sampling, can rival
compute spend — this is exactly why that module covered sampling as a cost
control, not just a nice-to-have).

## Right-sizing: the single highest-leverage cost lever

Over-provisioned instances are the most common and most fixable source of
waste — module 02's capacity planning gives you the actual utilization
data needed to right-size instead of guessing.

```bash
# find instances running well below what they're sized for, sustained over time
# (conceptual — via CloudWatch/Prometheus historical query)
avg_over_time(node_cpu_utilization[30d]) < 15   # AND memory similarly low
```

```
Before: 20 × m5.2xlarge (8 vCPU, 32GB) running at 12% avg CPU, 20% avg mem
After:  20 × m5.large  (2 vCPU, 8GB)  running at ~45% avg CPU, ~75% avg mem
        (validated against the load-test ceiling from module 02 first —
         resize down toward, not past, the safe headroom margin)
Savings: ~75% of compute cost, with static headroom preserved
```

Right-sizing is not a one-time exercise — utilization drifts as usage
patterns change, so it belongs on a recurring cadence (monthly/quarterly
review), the same discipline as drift detection (Level 3 module 06) applied
to cost instead of configuration.

## Autoscaling as a cost control, not just a reliability one

Module 02 introduced autoscaling for handling spikes; the same mechanism
is a direct cost lever for handling *troughs* — scaling down during low-
traffic periods instead of running peak capacity 24/7.

```yaml
# Conceptual scheduled scaling: lower minReplicas overnight when traffic is predictably low
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: orders-hpa }
spec:
  minReplicas: 3     # overnight floor — still meets the redundancy floor from module 02
  maxReplicas: 30
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } }
```

```bash
# Scheduled job to raise the floor before a known daily peak, lower it after
# (keeps the redundancy floor from Level 3 module 01 intact at all times —
#  never scale below N-1 redundancy just to save cost)
kubectl patch hpa orders-hpa -p '{"spec":{"minReplicas": 8}}'   # 08:00, before peak
kubectl patch hpa orders-hpa -p '{"spec":{"minReplicas": 3}}'   # 22:00, after peak
```

The floor (`minReplicas`) must never drop below what module 01's
redundancy requirement demands (enough instances, spread across AZs, to
survive losing one) — cost optimization that erodes the HA design it's
layered on top of is not a saving, it's a hidden risk.

## Spot/preemptible instances: cheap capacity for the right workloads

Spot (AWS)/preemptible (GCP) instances cost 60-90% less than on-demand but
can be reclaimed by the provider with little notice (seconds to two
minutes). Appropriate for workloads that tolerate interruption; wrong for
anything that can't:

```
Good fit: batch processing, CI runners, non-critical background workers,
          stateless web tier IF it can tolerate brief capacity dips and
          the on-demand fleet (module 01's redundancy floor) covers the gap
Bad fit:  database primaries, anything stateful without its own replication,
          the last remaining instance of anything (spot interruption +
          no redundancy = an outage, not a cost saving)
```

```yaml
# Conceptual mixed-instance ASG: base capacity on-demand, burst capacity on spot
MixedInstancesPolicy:
  InstancesDistribution:
    OnDemandBaseCapacity: 6        # matches the redundancy floor — always on-demand
    OnDemandPercentageAboveBaseCapacity: 20   # above the floor, mostly spot
    SpotAllocationStrategy: capacity-optimized
```

`OnDemandBaseCapacity` pinned to the redundancy floor is the key design
choice — spot instances add cheap *extra* capacity above the floor that
HA already requires to exist on stable on-demand instances.

## Storage lifecycle: don't pay premium prices for cold data

```bash
# S3 lifecycle policy: move backups (Level 3 module 03) to cheaper storage
# tiers as they age, since restore-speed requirements drop with age
aws s3api put-bucket-lifecycle-configuration --bucket company-backups-offsite \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "age-out-backups",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30,  "StorageClass": "STANDARD_IA" },
        { "Days": 90,  "StorageClass": "GLACIER" },
        { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
      ],
      "Expiration": { "Days": 2555 }
    }]
  }'
```

This applies directly to the backup retention policy from Level 3 module
03: a 14-day-old nightly backup you might restore from *today* needs fast
retrieval (Standard/Standard-IA); a 2-year-old backup kept only for
compliance (module 07) can sit in Deep Archive at a fraction of the cost,
since its retrieval time (hours) is acceptable for that use case.

## Tagging and cost allocation: you can't optimize what you can't attribute

```hcl
# Terraform: enforce tags on every resource so spend can be broken down
# by team/service/environment
resource "aws_instance" "web" {
  # ...
  tags = {
    Team        = "checkout"
    Service     = "orders"
    Environment = "production"
    CostCenter  = "eng-platform"
  }
}
```

```bash
# Cost Explorer-style query, grouped by tag — impossible without consistent tagging
aws ce get-cost-and-usage \
  --time-period Start=2026-08-01,End=2026-08-31 \
  --granularity MONTHLY --metrics "UnblendedCost" \
  --group-by Type=TAG,Key=Service
```

Without consistent, enforced tagging (a CI/policy check rejecting untagged
resources, not a wiki page asking nicely), cost review degenerates into "the
cloud bill went up, nobody knows which team's traffic caused it" — the
single biggest blocker to any cost-aware culture actually functioning.

## The cost-aware review, alongside the design review

Add cost as an explicit line in the design review checklist from Level 3's
capstone:

```
[ ] Instance sizes validated against actual load test / utilization data,
    not guessed
[ ] Autoscaling floor matches the HA redundancy requirement, not padded
    beyond it "to be safe"
[ ] Spot/preemptible used wherever the workload tolerates interruption
[ ] Storage lifecycle policies applied to backups/logs based on real
    access patterns, not left at default (often "never expire")
[ ] Every resource tagged for cost attribution
[ ] Observability volume (Level 3 module 09) sampled/retained
    appropriately for its actual value, not kept at 100% by default
```

## Exercise

1. Pull 30 days of CPU/memory utilization for a real (or toy) fleet and
   identify any instances running sustained below ~20% utilization —
   propose a right-sized alternative, validated against a load test
   ceiling (module 02) rather than a guess.
2. Design a mixed on-demand/spot capacity plan for one workload, explicitly
   setting the on-demand base to match its HA redundancy floor.
3. Write an S3 (or equivalent) lifecycle policy for your backup retention
   strategy from Level 3 module 03, with transition ages you can justify
   based on how often you'd actually need to restore a backup of that age.
4. Propose a tagging schema for your infrastructure and write the
   Terraform (or equivalent) enforcement that would reject an untagged
   resource at plan/apply time.
