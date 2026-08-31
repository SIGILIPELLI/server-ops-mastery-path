# 09 · Observability at Scale (metrics, logs, traces)

Level 2 introduced basic monitoring and centralized logging — enough for a
handful of services. Once you have dozens of services, multiple hosts, and
requests that hop across several of them (module 07), you need the three
pillars working together: **metrics** (is something wrong, in aggregate),
**logs** (what exactly happened), and **traces** (where, across which
services, did it happen) — plus a way to move between all three quickly
during an incident.

## The three pillars, and what each is (and isn't) for

**Metrics** — numeric time series, cheap to store and query at scale, ideal
for dashboards, alerting thresholds, and long-term trend analysis. Bad for
"what exactly happened to this one request" — metrics are aggregates by
design.

```
http_requests_total{service="orders", status="500"}  1204
http_request_duration_seconds{service="orders", quantile="0.99"}  0.87
```

**Logs** — discrete events with full detail (a stack trace, a request
body, a specific error message). Expensive to store and search at high
volume; the right tool for root-causing one specific failure once metrics
have told you *something* failed *somewhere*.

**Traces** — the path a single request took across multiple services, with
timing for each hop (a "span" per service call). The only one of the three
that directly answers "which of these six services in the call chain is
actually slow" — metrics per-service can't show you the *relationship*.

```
trace_id: 7a3f9c...
├─ span: web (12ms)
│  └─ span: orders (340ms)          ← the slow one
│     ├─ span: inventory (15ms)
│     └─ span: payments (310ms)     ← root cause: payments, not orders
```

Without tracing, "orders is slow" (visible in orders' own metrics) doesn't
tell you the actual bottleneck is a downstream call to `payments` — you'd
have to correlate timestamps across separate services' logs by hand.

## Structured logging and correlation IDs

The connective tissue between all three pillars is a **correlation ID**
(often called a trace ID) generated at the edge and propagated through
every hop:

```python
import logging, json, uuid

def log_event(request_id, service, message, **fields):
    logging.info(json.dumps({
        "timestamp": "2026-08-31T10:00:00Z",
        "request_id": request_id,
        "service": service,
        "message": message,
        **fields,
    }))

# at the edge, generate once per incoming request
request_id = str(uuid.uuid4())
log_event(request_id, "web", "request received", path="/checkout")
# ... pass request_id through every downstream call (HTTP header, message queue attribute) ...
```

```
# downstream service receives and reuses the same request_id
X-Request-Id: 7a3f9c12-88e1-4b0a-9c2e-1f0a6d3b9e77
```

With every log line JSON-structured and carrying the same `request_id`,
you can pull the full story of one failed checkout across every service it
touched with a single query — `request_id = "7a3f9c..."` — instead of
grepping five services' logs by approximate timestamp.

## Metrics: the RED and USE methods

Two widely used checklists for "what should every service expose":

**RED** (for request-driven services):
- **Rate** — requests per second
- **Errors** — failed requests per second
- **Duration** — latency distribution (not just average — track p50/p95/p99)

**USE** (for resources — CPU, disk, memory, connection pools):
- **Utilization** — % time the resource was busy
- **Saturation** — how much queued work is waiting for it
- **Errors** — count of resource-level errors (disk I/O errors, OOM kills)

```yaml
# Prometheus scrape config: every service exposes /metrics in Prometheus format
scrape_configs:
  - job_name: orders
    static_configs:
      - targets: ["orders-01:9100", "orders-02:9100", "orders-03:9100"]
```

```
# Example PromQL: error rate over the last 5 minutes, per service
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/
sum(rate(http_requests_total[5m])) by (service)
```

## Distributed tracing: OpenTelemetry

OpenTelemetry (OTel) is the vendor-neutral standard for producing traces,
metrics, and logs from application code, sent to whatever backend you
choose (Jaeger, Tempo, a vendor's SaaS product) — write instrumentation
once, swap backends without touching application code.

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="otel-collector:4317"))
)
tracer = trace.get_tracer(__name__)

def handle_checkout(order_id):
    with tracer.start_as_current_span("handle_checkout") as span:
        span.set_attribute("order.id", order_id)
        with tracer.start_as_current_span("call_payments"):
            charge_payment(order_id)   # this span becomes a child, nested under handle_checkout
```

The `otel-collector` sits between instrumented services and the storage
backend, buffering and batching so applications don't take a direct
dependency (or added latency) on the tracing backend's availability.

## Cost control: sampling

At scale, tracing (and to a lesser extent, verbose logging) every single
request is expensive to store and often unnecessary — most requests are
boring. **Sampling** keeps observability useful without the cost exploding:

```yaml
# Conceptual OTel Collector config: sample 10% of traces, but ALWAYS keep errors
processors:
  probabilistic_sampler:
    sampling_percentage: 10

  tail_sampling:
    policies:
      - name: keep-all-errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: sample-the-rest
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }
```

**Tail-based sampling** (deciding whether to keep a trace *after* seeing
how it turned out) is what lets you keep 100% of error/slow traces while
discarding most of the routine ones — head-based sampling (deciding before
the request completes) can't make that distinction and either keeps too
much or discards traces you'd have wanted.

## Correlating dashboards to logs to traces during an incident

The practical payoff of building all three: during an incident, the flow
is usually

```
Alert fires on a metric (error rate for `orders` > threshold)
   → dashboard shows WHEN it started and roughly HOW BAD
   → click through to traces for that time window, filtered to errors
   → find the specific slow/failing span (e.g. payments call timing out)
   → jump from that trace's request_id to the full structured logs for
     that request, across every service it touched
   → root cause found in minutes, not by grepping five hosts' logs by hand
```

This chain only works if the tools are wired together (dashboards link to
trace search, traces carry the same `request_id` logs use) — building the
three pillars in isolation, without the cross-links, leaves you back to
manual correlation under pressure.

## Exercise

1. Add a `request_id` (generated at the edge, propagated via an HTTP
   header) to a small multi-service toy app (2-3 services calling each
   other), and make every service log JSON lines including that ID.
2. Instrument the same app with OpenTelemetry tracing and run a local
   Jaeger instance (`docker run -p 16686:16686 -p 4317:4317
   jaegertracing/all-in-one`) as the backend; confirm you can see the full
   trace for one request across all services in the Jaeger UI.
3. Expose Prometheus-format RED metrics (`http_requests_total`,
   `http_request_duration_seconds`) from each service and scrape them with
   a local Prometheus instance.
4. Deliberately introduce a slow downstream call in one service, then walk
   the full path above: notice it in a metric, find the trace, find the
   specific slow span, and pull the matching structured logs by
   `request_id`.
