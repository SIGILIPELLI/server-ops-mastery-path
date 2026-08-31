# 02 · Capacity Planning

Every HA/DR design in Level 3 assumed the surviving instances/region could
actually absorb the traffic redirected to them. Capacity planning is the
discipline of making that assumption true on purpose — knowing today's
headroom, forecasting tomorrow's need, and provisioning ahead of demand
instead of reacting to it.

## The core question and the core metric

The question capacity planning answers: **"if load grows by X%, or we lose
Y% of capacity, do we stay within acceptable performance?"** The core
metric it's built on is **utilization** — what fraction of a resource's
maximum capacity is currently in use — tracked per resource (CPU, memory,
disk I/O, network, connection pools, database connections) because
different resources saturate at different growth rates.

```bash
# quick per-host utilization snapshot
top -bn1 | head -5                  # CPU
free -h                             # memory
iostat -x 1 3                       # disk I/O and %util
ss -s                                # connection counts
```

```
# Prometheus: CPU utilization per instance, useful as a fleet-wide trend
avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)
```

## Little's Law: the formula that connects load, latency, and concurrency

Little's Law: `L = λ × W` — the average number of requests in the system
(**L**, concurrency) equals the arrival rate (**λ**, requests/sec) times
the average time each request spends in the system (**W**, latency).

Rearranged, this is the formula behind "how many backend instances do I
need":

```
required_concurrency = requests_per_second × avg_request_duration_seconds
required_instances    = required_concurrency / max_concurrent_requests_per_instance
```

**Worked numbers:** a service handling 500 req/s, average request duration
200ms (0.2s):

```
required_concurrency = 500 × 0.2 = 100 concurrent requests in flight

# if each instance can safely handle 25 concurrent requests before latency degrades:
required_instances = 100 / 25 = 4 instances minimum

# add capacity for N-1 redundancy (Level 3 module 01) and headroom:
provisioned_instances = 4 instances × 1.5 (50% headroom) rounded up, spread
                         across AZs so losing one AZ still leaves enough = 6-8 instances
```

This is the actual arithmetic behind "how many servers do we need," derived
from measured numbers, not a guess — and it immediately shows *why*
latency and capacity are coupled: if `avg_request_duration` doubles (a
downstream dependency gets slower), required concurrency doubles too, even
at the same request rate.

## Forecasting growth

```bash
# pull 90 days of request-rate history and fit a trend (conceptual — Prometheus + a script)
promtool query range \
  --start=$(date -d '90 days ago' -Iseconds) --end=$(date -Iseconds) --step=1d \
  'sum(rate(http_requests_total[1h]))' http://prometheus:9090
```

```python
# simple linear/exponential trend fit to decide "when do we hit capacity"
import numpy as np

days = np.arange(90)
req_per_sec = load_from_prometheus()   # 90 daily samples

# fit exponential growth (common for a growing product) rather than linear
log_fit = np.polyfit(days, np.log(req_per_sec), 1)
daily_growth_rate = np.exp(log_fit[0]) - 1
print(f"~{daily_growth_rate*100:.2f}% daily growth")

current_capacity_rps = 800   # measured max sustained throughput at acceptable latency
current_rps = req_per_sec[-1]
days_until_saturation = np.log(current_capacity_rps / current_rps) / np.log(1 + daily_growth_rate)
print(f"~{days_until_saturation:.0f} days until current capacity is saturated")
```

The output of this kind of forecast — "current capacity runs out in ~45
days at current growth" — is what turns capacity planning into a scheduled
task (order more capacity, run a load test to confirm the new number, add
headroom) instead of a reactive scramble the week traffic actually hits the
ceiling.

## Load testing to find the real ceiling

Forecasting from historical trend tells you *when* you'll need more
capacity; load testing tells you *how much* one instance can actually
handle before it degrades, which the Little's Law calculation above depends
on.

```bash
# k6 load test: ramp up load and find the point where p99 latency degrades
cat <<'EOF' > loadtest.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 50 },
    { duration: '5m', target: 200 },
    { duration: '5m', target: 400 },
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(99)<500'],   // fail the test if p99 exceeds 500ms
  },
};

export default function () {
  const res = http.get('https://staging.example.com/api/orders');
  check(res, { 'status is 200': (r) => r.status === 200 });
}
EOF

k6 run loadtest.js
```

Run this against **staging at production-equivalent instance sizing**, not
a scaled-down dev environment — a load test against undersized hardware
tells you nothing about production's real ceiling. Watch the point where
p99 latency starts climbing sharply (not just where the test "fails") —
that inflection point is the practical per-instance capacity limit to plug
into the Little's Law calculation above.

## Headroom, autoscaling, and reserved buffer

- **Headroom** — deliberately provisioning above the measured need (the 50%
  in the worked example) to absorb traffic spikes, one instance's failure
  (Level 3 module 01), and forecast error, without paging anyone.
- **Autoscaling** — reactive capacity that adds instances as utilization
  rises, reducing the need for large static headroom, but only helps if new
  instances come up faster than the traffic spike develops (a spike
  hitting in 30 seconds and instances taking 3 minutes to boot and warm up
  still needs static headroom to survive that gap).

```yaml
# Conceptual Kubernetes HPA: scale on CPU, with min replicas providing the static headroom floor
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: orders-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: orders }
  minReplicas: 6    # the static headroom floor from the Little's Law calc above
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 60 }
```

`minReplicas: 6` is not arbitrary — it's the `provisioned_instances` number
from the worked example above, so the fleet never dips below the level
already known to survive one AZ's worth of instances failing, even during
a scale-down.

## Capacity planning for known events

Predictable spikes (a marketing launch, a seasonal sale, a scheduled
migration) deserve explicit pre-event capacity planning, not reliance on
autoscaling alone:

```
Pre-event capacity checklist:
[ ] Forecast expected peak (from marketing's traffic estimate, or last
    year's comparable event scaled by measured growth)
[ ] Load test at that forecast peak, in staging, at production sizing
[ ] Pre-scale (raise minReplicas / provision ahead) rather than relying on
    reactive autoscaling to keep up with a sudden step-change in traffic
[ ] Confirm downstream dependencies (database connection limits, third-
    party API rate limits, payments provider's own capacity) are also
    sized for the forecast peak — the fleet can be perfectly sized and
    still fail if the database's max_connections isn't
[ ] Have a scale-down plan for after the event, to avoid paying for
    peak capacity indefinitely
```

## Exercise

1. Measure a real (or realistic toy) service's average request rate and
   average request duration, and use Little's Law to compute required
   concurrency and instance count for both current load and a hypothesized
   2x growth scenario.
2. Run a k6 (or similar) load test against that service at
   production-equivalent sizing, ramping load until p99 latency clearly
   degrades, and record the actual per-instance ceiling.
3. Compare the load test's measured ceiling to your Little's Law estimate
   from step 1 — if they disagree significantly, investigate which input
   assumption was wrong (average duration under load vs. idle? a downstream
   dependency limit you didn't account for?).
4. Write a pre-event capacity checklist entry for one upcoming predictable
   spike (real or hypothetical) using the checklist above, with concrete
   numbers instead of placeholders.
