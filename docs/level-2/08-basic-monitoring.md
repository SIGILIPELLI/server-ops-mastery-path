# 08 · Basic Monitoring (uptime checks, resource metrics)

You cannot fix what you don't know is broken. This module is the minimum
viable monitoring setup: is the service up, and is the box running out of
a critical resource — before either becomes a page from an angry user.

## Layer 1: an external uptime check

An uptime check must run **outside** the server it's checking — if the
whole box is down, a monitor running on that same box can't tell you
anything. A cheap, real setup: a free-tier third-party uptime service
(UptimeRobot, Better Uptime, healthchecks.io) hitting your health endpoint
every 1-5 minutes, or a tiny script on a *different* machine.

```bash
# from a separate monitoring host/cron machine
curl -sf -o /dev/null -w "%{http_code} %{time_total}s\n" https://example.com/health
```

A good `/health` endpoint checks that the app can actually do its job, not
just that the process is alive:

```python
# Flask example
@app.route("/health")
def health():
    try:
        db.session.execute("SELECT 1")   # confirm DB connectivity, not just "process exists"
    except Exception:
        return {"status": "unhealthy"}, 503
    return {"status": "healthy"}, 200
```

A process that's running but can't reach its database should report
unhealthy — "the process exists" and "the service works" are different
facts, and only the second one matters to users.

## Layer 2: resource metrics on the box itself

```bash
# CPU, memory, disk — the classic three to watch
top -bn1 | head -15
free -h
df -h

# disk fill-rate is the one that silently kills servers (full disk = logs stop, DB writes fail)
df -h --output=pcent,target | tail -n +2
```

A simple cron-based disk-space alert with no extra tooling:

```bash
#!/usr/bin/env bash
# /usr/local/bin/check-disk.sh
THRESHOLD=85
usage=$(df / --output=pcent | tail -1 | tr -dc '0-9')
if [ "$usage" -ge "$THRESHOLD" ]; then
  echo "Disk usage at ${usage}% on $(hostname)" | mail -s "DISK ALERT $(hostname)" ops@example.com
fi
```

```cron
*/15 * * * * /usr/local/bin/check-disk.sh
```

This is deliberately low-tech — a `mail` command and a cron job — because
it works with zero new infrastructure, and "does the alert actually fire"
is easy to test end to end (module 9 covers a real metrics stack when you
outgrow this).

## Layer 3: watch the process itself

```bash
systemctl status myapp --no-pager    # is systemd's view of the service healthy?
journalctl -u myapp -p err --since "1 hour ago"   # any errors logged recently?
```

Pair systemd's own `Restart=on-failure` (module 1, level 1) with an
alert-on-restart check, since a service that keeps crash-looping is
"technically up" between crashes but clearly unhealthy:

```bash
systemctl show myapp -p NRestarts    # count of restarts since last reset — a rising number is a red flag
```

## Setting up a basic alert when a threshold is crossed

```bash
#!/usr/bin/env bash
# /usr/local/bin/check-health.sh — run from cron every minute on a SEPARATE monitoring host
URL="https://example.com/health"
code=$(curl -sf -o /dev/null -w "%{http_code}" --max-time 5 "$URL" || echo "000")
if [ "$code" != "200" ]; then
  echo "$(date -Iseconds) health check returned $code" >> /var/log/health-alerts.log
  curl -s -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"ALERT: $URL returned $code\"}" \
    "$SLACK_WEBHOOK_URL"
fi
```

```cron
* * * * * SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx /usr/local/bin/check-health.sh
```

This one script covers the two things that matter most at this level:
detect the failure, and get a human notified somewhere they'll actually see
it (a log file alone is not an alert — nobody's `tail -f`ing it at 2am).

## Avoiding alert fatigue from the start

- Alert on **symptoms users feel** (health check failing, high error rate,
  disk full) not every possible internal metric — a flood of low-value
  alerts trains people to ignore all of them, including the real ones.
- Require a threshold to be crossed for a sustained period (e.g. "health
  check failed 3 times in a row") rather than firing on a single blip,
  which is often just a transient network hiccup.

## Worked example: full loop, one host

```bash
# on the app host — health endpoint already returns 200/503
sudo mkdir -p /usr/local/bin
sudo tee /usr/local/bin/check-health.sh <<'EOF'
#!/usr/bin/env bash
code=$(curl -sf -o /dev/null -w "%{http_code}" --max-time 5 http://127.0.0.1:3000/health || echo "000")
[ "$code" != "200" ] && logger -t health-check "ALERT: health check returned $code"
EOF
sudo chmod +x /usr/local/bin/check-health.sh
(crontab -l 2>/dev/null; echo "* * * * * /usr/local/bin/check-health.sh") | crontab -
# simulate failure by stopping the app, then check the log
sudo systemctl stop myapp
sleep 65
journalctl -t health-check --since "2 min ago"
```

## Exercise

1. Write a `/health` endpoint for a toy app that fails (503) if a
   simulated dependency (e.g. a flag file `/tmp/db-down`) exists.
2. Write a cron-driven bash script that curls it every minute and logs a
   line via `logger` on failure.
3. Trigger the failure condition, confirm the alert fires within a minute
   via `journalctl`, then clear it and confirm alerting stops.
