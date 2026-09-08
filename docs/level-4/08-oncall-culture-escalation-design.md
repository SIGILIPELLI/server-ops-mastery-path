# 08 · On-Call Culture & Escalation Design

Runbooks and incident process (module 03) assume someone is actually
available, alert, and equipped to respond when paged. This module is about
designing the on-call *system* — rotation, escalation paths, alert quality,
and the human sustainability side — so that assumption holds up over
months and years, not just the first week.

## Alert quality: page for the right things, or people stop trusting pages

The single biggest driver of on-call burnout and, paradoxically, of missed
real incidents, is **alert fatigue** — too many low-value or non-actionable
pages train responders to under-react, including to the page that matters.

A page should meet a specific bar:

```
Good page criteria (ask before adding any new alert):
[ ] Is this actionable RIGHT NOW by whoever's paged? (if the fix can wait
    until business hours, it's a ticket, not a page)
[ ] Does it indicate real or imminent user impact? (not just "a metric
    crossed an arbitrary threshold with no clear consequence")
[ ] Is there a runbook, or is the fix genuinely a judgment call that needs
    a human paged rather than automation handling it?
[ ] Would a reasonable person paged at 3am agree this warranted waking up?
```

```yaml
# Bad: pages on every 5xx, even isolated/transient ones
- alert: AnyErrorOccurred
  expr: increase(http_requests_total{status=~"5.."}[1m]) > 0
  labels: { severity: page }

# Better: pages only on a sustained rate that indicates real, ongoing impact
- alert: HighErrorRate
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m])) 
    / sum(rate(http_requests_total[5m])) > 0.05
  for: 5m
  labels: { severity: page }
  annotations:
    runbook: "https://wiki.internal/runbooks/high-error-rate"
```

The `for: 5m` clause and the linked runbook are both deliberate: `for`
avoids paging on a single noisy spike that self-resolves, and a runbook
link means the person paged has a starting point instead of investigating
from zero every time, even for a rare alert they don't remember.

## Tracking and reducing pages that shouldn't have been pages

```
# Weekly on-call retro question set:
- How many pages fired this week? How many were actionable?
- Of the non-actionable ones, was the alert threshold wrong, or should it
  not exist at all?
- Did any page recur more than twice — if so, is there a fix (not just a
  runbook workaround) that would prevent it recurring a third time?
```

A team that tracks **pages per week** and **% actionable** as an explicit
metric, and treats a rising trend or a low actionable-percentage as a
problem to fix (tune the threshold, fix the underlying flaky behavior,
delete the alert) — rather than an accepted cost of running production —
is the difference between on-call staying sustainable and slowly becoming
something people dread and try to avoid.

## Escalation policies: what happens when the primary doesn't respond

```yaml
# Conceptual PagerDuty escalation policy
escalation_policy:
  name: orders-service-oncall
  rules:
    - escalation_delay_minutes: 5
      targets: [primary_oncall]
    - escalation_delay_minutes: 10
      targets: [secondary_oncall]
    - escalation_delay_minutes: 15
      targets: [team_lead, engineering_manager]
```

Every layer exists to answer "what if the previous layer doesn't
acknowledge in time" — a phone on silent, a dead battery, being in an area
with no signal. An escalation policy with only one layer (page the primary,
and if they don't respond, nothing happens) has a silent single point of
failure identical in shape to the infrastructure SPOFs from Level 3 module
01, just in the human layer instead of the technical one.

```yaml
# Multi-channel notification within one layer, not just one channel
targets: [primary_oncall]
notification_rules:
  - { delay_minutes: 0, channel: push }
  - { delay_minutes: 2, channel: sms }
  - { delay_minutes: 4, channel: phone_call }
```

## Rotation design: sustainable schedules

```
Weekly rotation:  1 person primary + 1 person secondary, swap every Monday
                  → predictable, easy to plan around, but one bad week can
                    be exhausting with no relief
Follow-the-sun:   3 regions (APAC/EMEA/Americas) each cover their own
                  daytime hours → nobody is paged overnight, but requires
                  a genuinely distributed team and careful handoff process
Primary/secondary
with escalation:  primary gets first page; secondary exists as backup, not
                  as "also gets paged every time" (avoids doubling fatigue)
```

Rotation fairness matters concretely, not just as a morale nicety:

```
# Track and review, don't just assume fairness:
- pages per person per rotation (should be roughly even, or explicitly
  compensated/rotated if one service is genuinely noisier)
- off-hours pages per person per month
- time-to-acknowledge, per person (a consistently slow acknowledger may
  need a schedule adjustment, or may indicate a burned-out responder)
```

Compensation for on-call (extra pay, time off in lieu) is a policy decision
outside engineering's control, but the *data* enabling that decision fairly
— actual page load per person — is an engineering responsibility to
collect and surface.

## Handoff: context transfer between rotations

```markdown
# On-call handoff notes template — filled in at the end of every rotation
## Ongoing issues to watch
- `orders` p99 latency has been creeping up over the week, not yet
  alert-worthy but worth watching (see dashboard link)

## Incidents this rotation
- SEV2 on Tuesday, payments timeout — see postmortem [link]
- Action items from that postmortem are still open, due 2026-09-07

## Changes deployed this rotation that could still be a factor
- New rate limiter shipped Thursday — if odd 429s show up, check this first

## Anything flaky/annoying but not yet fixed
- The disk-usage alert on `log-01` fires around 2am most nights and
  self-resolves — investigated once, inconclusive, not yet silenced
  (should probably be fixed or explicitly muted, see retro backlog)
```

A structured handoff prevents the next rotation from rediscovering
context the previous one already had — especially the "flaky but not
fixed" category, which otherwise gets independently investigated by every
new on-call engineer, wasting the same effort repeatedly instead of once.

## Blameless culture, applied to on-call specifically

The same blameless principle from module 03's postmortems applies to
on-call performance: a slow acknowledgment or a missed page should trigger
"is the alert routing/escalation policy adequate" and "is this person
overloaded," not individual blame. A culture where being paged and failing
to immediately fix a complex issue solo is treated as a personal failing
teaches people to hide struggle and avoid escalating for help — exactly the
opposite of what a healthy escalation policy depends on.

## How It Actually Works

**Why `for: 5m` on an alert rule changes what Prometheus actually
evaluates, not just when it fires.** Without `for`, Prometheus's alerting
rule fires the instant a single evaluation cycle's expression crosses the
threshold — a metric that spikes for one 15-second scrape interval and
recovers immediately still triggers a page. `for: 5m` requires the
expression to remain continuously true across every evaluation within that
5-minute window before the alert transitions from `pending` to `firing`
(visible as the `ALERTS` metric's state) — it's not a delay on notification,
it's a persistence requirement on the underlying condition itself, which
is exactly what filters out single-scrape noise while still catching a
genuinely sustained error rate within a bounded, known latency.

**Why escalation timers alone don't guarantee delivery, and multi-channel
within a layer is the actual fix.** A push notification depends on the
device having connectivity and the app being allowed to interrupt Do Not
Disturb settings — both can silently fail (phone in airplane mode, OS
notification permission revoked) with no signal back to the paging system
that delivery didn't happen. Escalating to a *different channel* (SMS
uses the cellular carrier's independent delivery path; a phone call uses
yet another) after a short delay isn't redundancy for its own sake — each
channel has an independent failure mode, so stacking channels within one
escalation layer is the only way to get real delivery confidence before
falling back to escalating to a different *person*, which is a much
slower and more disruptive recovery path.

**Why tracking pages-per-person and time-to-acknowledge reveals problems a
single incident retro can't.** Any individual page looks reasonable in
isolation — a repeat page for the same underlying flaky alert is
indistinguishable, incident-by-incident, from N unrelated real issues
unless someone aggregates across time. Trending "pages per week" and "%
actionable" surfaces exactly the pattern an in-the-moment incident
response can't see: a specific alert firing 15 times a month at 40%
actionable is a systemic alert-quality problem invisible from any single
occurrence, but glaring once the data is aggregated — which is the same
argument for tracking patch compliance % (Level 4 module 04) or error
budget burn (module 05) as a trend rather than trusting any single
snapshot.

## Exercise

1. Audit your current (or a hypothetical) alerting setup against the "good
   page criteria" checklist above — for each alert, keep, tune the
   threshold, or delete.
2. Design an escalation policy with at least 3 layers and multi-channel
   notification within the first layer, and write down the specific
   condition that triggers each escalation step.
3. Pick a rotation model (weekly, follow-the-sun, etc.) appropriate for
   your team's size and geographic distribution, and justify the choice
   against the trade-offs described above.
4. Write a handoff template for your team and fill it out once, honestly,
   for a real or recent rotation — including at least one "flaky but not
   yet fixed" item.
