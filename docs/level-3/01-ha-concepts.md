# 01 · High Availability Concepts (redundancy, failover, health checks)

High availability (HA) is not a product you install — it's a design property
that comes from removing single points of failure (SPOFs) and giving the
system a way to detect and route around failures automatically. Everything
else in Level 3 (DNS/LB patterns, backups, IaC, chaos engineering) is a tool
in service of this one goal.

## Availability as a number

Availability is usually expressed as a percentage of uptime over a year,
often called "the nines":

| Availability | Downtime / year | Downtime / month |
|---|---|---|
| 99% ("two nines") | ~3.65 days | ~7.3 hours |
| 99.9% ("three nines") | ~8.76 hours | ~43.8 minutes |
| 99.95% | ~4.38 hours | ~21.9 minutes |
| 99.99% ("four nines") | ~52.6 minutes | ~4.4 minutes |
| 99.999% ("five nines") | ~5.26 minutes | ~26 seconds |

Each additional nine is disproportionately more expensive — it usually means
another layer of redundancy, more automation, and less tolerance for manual
intervention during an incident. Before designing for five nines, know what
the business actually needs (see Level 4, "Choosing the Right HA Architecture
for a Given SLA") — over-engineering has a real cost too.

## Redundancy: eliminate single points of failure

A SPOF is any single component whose failure takes down the whole system.
Walk the request path end to end and ask "if this dies, what happens?":

```
client → DNS → load balancer → app servers → database → shared storage
```

- **One app server** → SPOF. Fix: run N ≥ 2 identical instances behind a
  load balancer.
- **One load balancer** → SPOF. Fix: run a pair with a floating/virtual IP
  (keepalived + VRRP, or a managed LB) — see module 02.
- **One database primary** → SPOF for writes. Fix: replication + failover
  (a full topic on its own, out of scope here, but the failover *pattern*
  is the same one covered below).
- **One availability zone / one physical rack / one power feed** → SPOF for
  everything above. Fix: spread redundant copies across zones (Level 4,
  "Multi-Region / Multi-AZ Architecture Concepts").

Redundancy alone isn't enough — you also need something watching the
redundant copies and rerouting traffic away from a failed one. That's a
health check.

## Health checks

A health check is a cheap, frequent probe that answers one question: "can
this instance currently serve traffic correctly?" There are two common
shapes:

**Liveness check** — is the process even running/responding at all?

```nginx
location /healthz {
    return 200 "ok\n";
}
```

**Readiness check** — is the process running *and* actually able to do its
job (DB connection open, cache warm, disk not full)?

```python
# Flask example
from flask import Flask, jsonify
import psycopg2

app = Flask(__name__)

@app.route("/readyz")
def readyz():
    try:
        conn = psycopg2.connect(dbname="app", connect_timeout=2)
        conn.close()
        return jsonify(status="ready"), 200
    except Exception as e:
        return jsonify(status="not ready", error=str(e)), 503
```

A load balancer or orchestrator polls `/readyz` every few seconds; a few
consecutive failures mark the instance "down" and traffic stops flowing to
it. This is exactly the `max_fails`/`fail_timeout` mechanism from Level 2's
load balancing module, generalized: readiness checks are what make failover
*automatic* instead of requiring a human to notice and act.

Two failure modes to design against:

- **False positive (marked healthy when it isn't)** — traffic keeps flowing
  to a broken instance. Guard against this by making the readiness check
  actually exercise the dependency (DB ping), not just "process is up."
- **False negative (marked unhealthy when it's fine)** — a slow but working
  instance gets pulled from rotation unnecessarily, reducing capacity right
  when it's needed. Guard against this with a sensible timeout and a
  "needs N consecutive failures" threshold instead of failing on one blip.

## Failover: automatic vs. manual

**Automatic failover** — the system detects the failure and reroutes without
a human in the loop. Examples: LB stops sending traffic to a failed health
check; a database replica is promoted by a cluster manager (Patroni,
Galera). Fast (seconds), but must be trustworthy — a bad automatic failover
(e.g. split-brain, promoting a replica that's behind) can turn a small
outage into data loss.

**Manual failover** — a human runs a documented, tested procedure. Slower
(minutes, depends on who's paged), but a human can apply judgment a
health-check script can't ("the primary's CPU is at 100% but it's still
correct — don't fail over yet, just add capacity").

A common middle ground: automatic failover for known-safe cases (single
instance behind an LB), manual (but scripted and rehearsed) failover for
higher-stakes cases (database primary) until you've built enough confidence
and tooling to automate it. Runbooks for the manual case are covered in
Level 4, "Incident Response & Runbooks."

## Worked example: keepalived VRRP failover between two nginx boxes

Two nginx hosts, `10.0.0.11` and `10.0.0.12`, share a virtual IP
`10.0.0.100` that clients actually connect to. `keepalived` uses VRRP to
have exactly one host "own" the VIP at a time, and moves it automatically if
the owner goes down.

```bash
sudo apt install -y keepalived
```

`/etc/keepalived/keepalived.conf` on the primary (`10.0.0.11`):

```
vrrp_script chk_nginx {
    script "/usr/bin/pgrep nginx"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass changeme_shared_secret
    }
    virtual_ipaddress {
        10.0.0.100/24
    }
    track_script {
        chk_nginx
    }
}
```

Same file on the backup (`10.0.0.12`), with `state BACKUP` and a lower
`priority 100`. Both hosts run this identically otherwise; whichever has the
higher effective priority (adjusted down by `weight` if `chk_nginx` fails)
holds the VIP.

```bash
sudo systemctl enable --now keepalived
ip addr show eth0 | grep 10.0.0.100   # shows the VIP only on the current master
```

Test the failover:

```bash
# on the master
sudo systemctl stop nginx
# within ~2-3 seconds (interval * a few missed adverts), the VIP moves to .12
ip addr show eth0 | grep 10.0.0.100   # now empty on .11
# on .12:
ip addr show eth0 | grep 10.0.0.100   # now present
```

Clients connecting to `10.0.0.100` see, at worst, a couple of failed
connections during the ~2-3 second handover — this is the mechanics behind
"the load balancer is highly available too," not just the backends behind
it.

## Exercise

1. Stand up two small VMs (or containers) running nginx and configure
   `keepalived` as above so they share a virtual IP.
2. Write a script that curls the VIP once per second and logs the response
   and which backend's `Server` header (add one via
   `add_header X-Served-By $hostname;` in nginx) answered.
3. Kill nginx on the current VRRP master mid-run and confirm from the log:
   how many requests failed (if any), and how long the VIP took to move.
4. Repeat with the `chk_nginx` health check removed from the config, and
   observe why "keepalived is running" alone doesn't guarantee "nginx is
   serving traffic" — this is the liveness-vs-readiness distinction from
   above, applied at the VIP level.
