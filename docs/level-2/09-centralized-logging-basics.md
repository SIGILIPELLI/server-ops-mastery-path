# 09 · Centralized Logging Basics

Level 1 covered reading logs on a single box (`journalctl`, `/var/log`).
Once you have more than one server, "SSH into each box and grep" stops
scaling — you need logs shipped somewhere central where you can search
across every host at once. This module builds that pipeline with the free,
widely-used combination of **Vector** (or Filebeat) shipping into a
central store.

## Why centralize

- An incident spans multiple backends (module 3) — you need one place to
  correlate what happened on all of them at the same timestamp.
- Logs on a crashed/terminated box are gone with it — shipping them off-box
  in near-real-time is your only record if the host itself doesn't survive.
- Grepping 10 files by hand doesn't scale; a search index does.

## Architecture

```
app (stdout/stderr → journald)
     │
     ▼
log shipper (Vector/Filebeat) reads journald, forwards over the network
     │
     ▼
central log store (Loki / Elasticsearch / a hosted service)
     │
     ▼
you, searching/dashboarding (Grafana / Kibana)
```

## Shipping journald logs with Vector

```toml
# /etc/vector/vector.toml
[sources.app_logs]
type = "journald"
include_units = ["myapp.service", "nginx.service"]

[transforms.add_host]
type = "remap"
inputs = ["app_logs"]
source = '''
.host = get_hostname!()
.environment = "production"
'''

[sinks.loki]
type = "loki"
inputs = ["add_host"]
endpoint = "http://loki.internal:3100"
encoding.codec = "json"
labels.job = "myapp"
labels.host = "{{ host }}"
```

```bash
sudo systemctl enable --now vector
journalctl -u vector -f    # confirm it's shipping without errors
```

The `remap` transform adds fields (hostname, environment) to every log line
before it's shipped — this is what lets you later filter "show me only
production, only host web-3" in the central UI.

## Querying centralized logs (LogQL, Loki's query language)

```logql
{job="myapp"} |= "ERROR"
{job="myapp", host="web-2"} | json | status_code >= 500
{job="myapp"} |= "ERROR" [5m]  # count errors in the last 5 minutes, e.g. for an alert
```

The equivalent single-host `journalctl` command you already know from
level 1 (`journalctl -u myapp -p err`) only ever sees one machine —
`{job="myapp"} |= "ERROR"` across a Loki index sees every machine shipping
that label, which is the entire point of this module.

## Structured logging: log JSON, not free text

Free-text logs (`"user 42 logged in from 1.2.3.4"`) are hard to query
precisely. Structured JSON logs are trivial to filter and aggregate on
specific fields:

```python
import logging, json, sys

class JsonFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "user_id": getattr(record, "user_id", None),
            "request_id": getattr(record, "request_id", None),
        })

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JsonFormatter())
logging.getLogger().addHandler(handler)

logging.info("user logged in", extra={"user_id": 42, "request_id": "abc-123"})
# -> {"timestamp": "...", "level": "INFO", "message": "user logged in", "user_id": 42, "request_id": "abc-123"}
```

With structured logs, `{job="myapp"} | json | user_id="42"` finds every
event for that user across every host and every deploy — a query that's
essentially impossible to do reliably against free-text logs at scale.

## Log retention and cost control

Centralized logging has a real storage cost that grows with volume — set a
retention policy deliberately rather than by accident:

```yaml
# Loki config snippet
limits_config:
  retention_period: 720h   # 30 days
```

A common pattern: keep verbose (debug) logs for a short window (days), keep
error/warn-level logs longer (weeks/months) for incident postmortems and
compliance, and sample or drop extremely high-volume, low-value log lines
(e.g. per-request access logs on a busy endpoint) rather than shipping
100% of everything forever.

## Correlating a request across services with a request ID

```python
import uuid
request_id = request.headers.get("X-Request-Id", str(uuid.uuid4()))
# pass it through: log it locally, AND forward it as a header to any downstream service call
```

Generating (or propagating) one `request_id` per incoming request and
including it in every log line that request produces — across every
service it touches — is what turns "logs from five different backends"
into "the complete story of one user's failed checkout," searchable with a
single `request_id="..."` query.

## Exercise

1. Install Vector (or Filebeat) on a test VM and configure it to ship
   `journalctl` output for one unit to a local file or a free-tier Loki/
   Grafana Cloud instance.
2. Switch a toy app's logging to structured JSON with at least a
   `request_id` field, and confirm you can query for a single request's
   full log trail.
3. Write a LogQL (or equivalent) query that counts ERROR-level lines in the
   last 5 minutes — the kind of query you'd wire into an alert in the next
   level's observability work.
