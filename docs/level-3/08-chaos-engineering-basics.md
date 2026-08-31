# 08 · Chaos Engineering Basics

Every module so far has *designed* for failure — redundancy, health checks,
backups, retries. Chaos engineering is how you find out whether those
designs actually work, by deliberately causing failures in a controlled way
before an uncontrolled one happens on its own.

## Why deliberately break things

Untested failure-handling code is one of the most common sources of
surprise outages: the failover logic that's never actually triggered, the
backup that's never actually restored (module 03), the retry policy that
turns out to retry non-idempotent writes and duplicates orders. Chaos
engineering treats "does the failure-handling actually work" as a
falsifiable, testable claim rather than an assumption baked into the
architecture diagram.

The core loop:

1. Define a **steady state** — a measurable signal that the system is
   working normally (e.g. error rate < 1%, p99 latency < 300ms, checkout
   success rate stable).
2. Form a **hypothesis** — "if we kill one app instance, the steady state
   holds, because the load balancer's health checks reroute traffic within
   N seconds."
3. **Inject a real failure**, in a controlled and reversible way.
4. **Observe** — did the steady state hold? If not, where exactly did the
   assumption break?
5. **Fix the gap**, then re-run the same experiment to confirm the fix
   actually closed it.

## Start small and safe: game days before automated chaos

Before running unattended chaos experiments in production, run a **game
day**: a scheduled, announced exercise where the team deliberately breaks
something in staging (or a controlled slice of prod) and watches what
happens together. This builds the muscle — both the tooling and the team's
confidence reading dashboards and reacting — before increasing the blast
radius or removing the human from the loop.

```
Game day checklist:
[ ] Steady-state metric identified and dashboard open
[ ] Hypothesis written down before starting
[ ] Rollback/abort procedure known and tested
[ ] Stakeholders notified of the window
[ ] Blast radius scoped (one instance? one AZ? never "all of prod" on day one)
[ ] Someone assigned to watch alerts, someone assigned to inject the failure
```

## Failure injection techniques, from simple to advanced

**Process/instance kill** — the most basic experiment, directly testing
module 01's redundancy and health checks:

```bash
# pick one backend instance out of the pool and kill its app process
ssh app-03 'sudo systemctl stop app'
# watch: does the LB stop routing to it within max_fails/fail_timeout?
# does overall error rate stay near zero?
sudo systemctl start app   # restore
```

**Network partition / latency injection** — simulate a slow or unreachable
dependency, testing module 07's timeout/circuit-breaker configuration:

```bash
# add 500ms of latency to all outbound traffic on this host (Linux tc/netem)
sudo tc qdisc add dev eth0 root netem delay 500ms

# simulate total packet loss to one specific IP (e.g. the payments service)
sudo tc qdisc add dev eth0 root netem loss 100%

# always remove the rule when done — this is real network impairment
sudo tc qdisc del dev eth0 root
```

**Resource exhaustion** — does the app degrade gracefully or crash the
whole host when CPU/memory/disk are under pressure?

```bash
# stress-ng: consume CPU and memory for a bounded time, then stop automatically
sudo apt install -y stress-ng
stress-ng --cpu 4 --vm 2 --vm-bytes 1G --timeout 120s
```

```bash
# fill disk to test "what happens when /var is full" (do this on a scratch/test host!)
fallocate -l 9G /tmp/fill_disk.img
# ... observe logging, app behavior, alerting ...
rm /tmp/fill_disk.img   # release immediately after observing
```

**Dependency failure** — stop a database or cache the app depends on and
watch how it degrades:

```bash
docker stop redis-cache
# does the app fail every request, or fall back to a slower path (DB query
# instead of cache), as designed? Confirm against the actual design intent.
docker start redis-cache
```

## Tooling for repeatable chaos experiments

Ad-hoc `tc`/`stress-ng`/`systemctl stop` commands work for a first game
day; for repeatable, scheduled experiments with automatic abort conditions,
purpose-built tools add safety rails:

```yaml
# Conceptual Chaos Mesh experiment (Kubernetes): kill one pod, auto-revert after 60s
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kill-one-app-pod
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces: [production]
    labelSelectors:
      app: orders
  duration: "60s"
```

```yaml
# Conceptual AWS FIS (Fault Injection Simulator) experiment with a built-in stop condition
description: "Kill 25% of app instances, abort if error rate exceeds 5%"
targets:
  instances:
    resourceTags: { app: orders }
    selectionMode: PERCENT(25)
actions:
  - actionId: aws:ec2:stop-instances
stopConditions:
  - source: aws:cloudwatch:alarm
    value: high-error-rate-alarm
```

The `stopConditions`/auto-revert pattern is the key safety feature that
separates "chaos engineering" from "recklessly breaking production": the
experiment monitors the same steady-state signal it's testing against, and
aborts itself the moment things look genuinely bad, rather than relying on
a human to notice and intervene in time.

## Blast radius discipline

Always experiment against the smallest scope that still tests the
hypothesis, and expand only after confidence builds:

```
1. One instance, in staging
2. One instance, in production, off-peak, with someone watching
3. One AZ/shard, in production, off-peak
4. Scheduled, unattended, small blast radius, automatic abort on steady-state breach
```

Never start at step 4. Chaos engineering earns broader scope over time by
demonstrating the safety mechanisms (fast, correct auto-revert) actually
work at a small scale first.

## Exercise

1. Pick one HA mechanism you've built in an earlier module (e.g. the
   keepalived VIP failover from module 01, or the nginx `max_fails`
   passive health check from Level 2). Write a one-paragraph hypothesis
   predicting exactly what will happen when you break it.
2. Run a controlled game day: define the steady-state metric you'll watch,
   inject the failure (kill the process/interface), and record what
   actually happened versus your hypothesis.
3. If reality didn't match the hypothesis, identify the specific gap (wrong
   timeout value? health check too lenient? no health check at all?) and
   fix it.
4. Re-run the same experiment after the fix and confirm the steady state
   now holds — this "fix, then re-verify with the same experiment" step is
   what turns chaos engineering into an actual improvement loop rather than
   a one-off finding.
