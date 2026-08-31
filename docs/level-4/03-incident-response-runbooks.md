# 03 · Incident Response & Runbooks

Every design in Levels 1-3 assumes something eventually goes wrong despite
the redundancy, health checks, and chaos testing. This module is about what
happens in the minutes and hours after it does — the process, roles, and
documentation that turn "something is broken" into "it's fixed, and we
understand why," without depending on any one person's memory under
pressure.

## Incident severity levels

Not every failure deserves the same response. A shared severity scale lets
anyone — even someone unfamiliar with the specific system — gauge urgency
correctly from the first message:

| Severity | Definition | Example | Response |
|---|---|---|---|
| SEV1 | Full outage or data loss, all/most customers affected | Checkout completely down | Page immediately, all-hands, exec notified |
| SEV2 | Partial outage or major degradation | One region down but others absorbing traffic | Page on-call, escalate if not resolving |
| SEV3 | Minor degradation, workaround exists | Elevated latency on a non-critical endpoint | Ticket, business-hours fix |
| SEV4 | Cosmetic or no customer impact | A dashboard panel is wrong | Ticket, no urgency |

Declaring a severity is itself an action someone takes explicitly and
early — an incident that's quietly assumed to be "probably SEV3" often
doesn't get the escalation and communication a real SEV1 needs until it's
already dragged on.

## Incident roles

Clear roles prevent the two classic failure modes of incident response:
nobody making decisions ("who's actually driving this?"), and everybody
trying to fix it at once with nobody communicating status.

- **Incident Commander (IC)** — coordinates the response, makes the
  call on severity/escalation, is not necessarily the person typing
  commands. Owns the timeline and the decision to declare "resolved."
  Empowered to pull in more people, without needing permission.
  Recommended for SEV1-2 and worth declaring on any potential SEV1.
- **Operator(s)** — the person(s) actually running diagnostics and
  applying fixes, following the IC's prioritization.
  Recommended for all severities where hands-on debugging is needed.
- **Communications lead** — posts status updates to stakeholders/status
  page, so the IC and operators aren't interrupted repeatedly for "any
  update?"
  Recommended for SEV1-2, especially customer-facing incidents.
- **Scribe** — keeps a live timestamped log of what was tried, what was
  observed, and when — this becomes the raw material for the postmortem
  and dramatically speeds it up if kept in real time instead of
  reconstructed afterward.
  Recommended for SEV1, valuable for SEV2.

On a small team, one person may hold two of these roles simultaneously
(never IC + sole operator with no scribe on a SEV1, if avoidable) — the
point is that each function gets done by someone, explicitly assigned, not
assumed.

## The incident timeline: what to record, and when

```
14:02  Alert fires: error rate on `orders` > 5% for 5 min
14:03  On-call acknowledges page
14:05  IC declared: alice. Operator: bob. Scribe: carol.
14:06  Severity set: SEV2 (checkout degraded, not fully down)
14:07  bob: checked orders dashboard — p99 latency 3x normal, error rate 8%
14:09  bob: traced errors to payments timeout (Level 3 module 09's tracing)
14:11  bob: payments provider status page shows a known incident on their side
14:12  carol: posted status update — "investigating checkout delays"
14:15  bob: enabled circuit breaker fallback (Level 3 module 07) to fail
       fast instead of timing out slowly
14:17  error rate back to <1%; latency still elevated but acceptable
14:25  payments provider resolves their incident
14:27  alice: declares incident resolved
```

Recording *decisions and observations as they happen*, not just the final
outcome, is what makes the eventual postmortem accurate — a timeline
reconstructed from memory a day later reliably loses the details that
matter most (what was tried and ruled out, and why).

## Runbooks

A runbook is a step-by-step procedure for a *specific, anticipated*
failure, written so someone who didn't build the system can follow it
correctly under pressure. It complements the general incident process
above by removing guesswork for known failure modes.

```markdown
# Runbook: Postgres primary unresponsive

## Symptoms
- `pg_isready` fails against the primary host
- Alert: "database connection errors" fired for `orders` and `payments`

## Diagnosis
1. SSH to the primary: `ssh db-primary-01`
2. Check if the process is running: `systemctl status postgresql`
3. Check disk space: `df -h /var/lib/postgresql` — if full, this is the
   likely cause; see "Disk full" section below.
4. Check the replica's lag before doing anything else:
   `psql -h db-replica-01 -c "SELECT now() - pg_last_xact_replay_timestamp();"`
   If lag > 60s, promoting now risks losing recent writes — page a senior
   engineer before proceeding rather than promoting unilaterally.

## Resolution: promote the replica
1. On the replica: `sudo -u postgres pg_ctl promote -D /var/lib/postgresql/16/main`
2. Update the app's connection string / service discovery entry to point
   at the new primary (see `deploy/db-failover.sh`).
3. Confirm writes succeed: `psql -h <new-primary> -c "INSERT INTO health_check ..."`
4. Page the database team to rebuild a new replica once the dust settles —
   the system is now running without redundancy until that's done.

## Rollback
There is no clean rollback once a replica is promoted — the old primary,
if it comes back, must be reconfigured as a replica of the new primary
(re-image it), never brought back as a second primary (split-brain).

## Escalation
If replica lag > 60s or promotion fails: page the database on-call lead
directly (see #db-oncall in PagerDuty), do not attempt manual recovery
without them.
```

The structure that matters, in every runbook: **symptoms** (how do you know
this is the problem), **diagnosis** (how do you confirm it before acting),
**resolution** (exact commands, not vague descriptions), **rollback** (what
if the fix doesn't work or makes it worse), and **escalation** (when to
stop and get help, and exactly who).

## Postmortems: blameless, and focused on systems not people

A postmortem is written after every SEV1 (and often SEV2) once things are
stable, answering: what happened, what was the impact, what was the
timeline, and — the part that actually prevents recurrence — what
systemic changes will reduce the chance or impact of this class of failure
happening again.

**Blameless** means the postmortem asks "what about our systems and
processes allowed this to happen" instead of "who caused this." A blameful
postmortem culture teaches people to hide mistakes and avoid reporting
near-misses, which removes exactly the information needed to prevent
repeats — it actively makes the organization less reliable over time.

```markdown
# Postmortem: Checkout degradation, 2026-08-31

## Impact
Checkout success rate dropped from 99.8% to 92% for 15 minutes (14:02-14:17),
affecting an estimated 340 orders.

## Root cause
Upstream payments provider had a regional outage. Our `orders` service had
no circuit breaker configured for payments calls, so it kept trying and
timing out slowly (2s timeout × 2 retries) instead of failing fast, which
amplified latency and consumed connection pool capacity.

## What went well
- Alert fired promptly and correctly identified the affected service.
- Tracing (Level 3 module 09) correctly pinpointed payments as the source
  within 3 minutes of investigation starting.

## What went poorly
- No circuit breaker existed for the payments dependency despite it being
  an external, less-reliable call — this is exactly the scenario Level 3
  module 07 describes as circuit breaking's purpose.
- Status page update took 5 minutes after severity was declared — slower
  than our 2-minute target.

## Action items
| Action | Owner | Due |
|---|---|---|
| Add circuit breaker to all `orders → payments` calls | bob | 2026-09-07 |
| Add automated status-page draft on SEV1/2 declaration | carol | 2026-09-14 |
| Add a chaos game day (Level 3 module 08) simulating payments latency | alice | 2026-09-21 |
```

The action items table, with named owners and dates, is what separates a
postmortem that actually improves reliability from one that's a formality
— an "action items" section with no owner or due date reliably never gets
done.

## Exercise

1. Write a runbook, in the format above, for one real failure mode in a
   system you know (or one from an earlier module's capstone) — be specific
   enough with commands that someone unfamiliar with the system could
   follow it.
2. Run (or simulate) a tabletop incident: have someone else read only your
   runbook's "Symptoms" section and try to diagnose and resolve the
   simulated issue using just the runbook. Note anywhere they got stuck or
   needed information the runbook didn't provide, and fix the runbook.
3. Write a blameless postmortem for a real or simulated incident, including
   a timeline, root cause, and an action items table with owners and dates.
4. Assign the four incident roles (IC, operator, comms, scribe) for a
   hypothetical SEV1 on your team (or a team of one, explicitly noting
   which roles you'd combine) and write one sentence on what each role is
   responsible for *not* doing (e.g. "the IC does not personally debug the
   database — that's the operator's job").
