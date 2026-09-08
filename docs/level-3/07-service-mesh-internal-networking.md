# 07 · Service Mesh & Internal Networking Concepts

As a deployment grows from "one app talking to one database" into a dozen
services calling each other, the *internal* (service-to-service) network
becomes as important to operate as the external one. This module covers the
concepts — service discovery, mutual TLS, retries/circuit breaking — and
where a service mesh fits, without requiring you to already run Kubernetes.

## The problem service meshes solve

With N services calling each other, every pair needs answers to the same
set of questions, and answering them independently in each service's code
doesn't scale:

- **Service discovery** — how does the `orders` service find a current,
  healthy address for the `payments` service, when instances come and go?
- **Encryption in transit** — is traffic between services on the internal
  network encrypted, or does anyone with access to that network segment see
  plaintext?
- **Retries and timeouts** — if `payments` is slow, does `orders` wait
  forever, retry blindly and pile on load, or fail fast?
- **Observability** — can you see the request graph (which service called
  which, how long each hop took) across the whole mesh, not just inside one
  service's logs?

A service mesh's core idea: pull all of this out of application code and
into a shared infrastructure layer, usually implemented as a small proxy
("sidecar") deployed next to every service instance, handling all of that
service's inbound and outbound traffic.

## Service discovery: the foundation, mesh or not

Even without a mesh, you need service discovery once you have more than a
hardcoded IP:

```
# DNS-based (simplest, works standalone): each service resolves a name
payments.internal.example.com  →  resolves to current healthy instances
```

```yaml
# Consul-style service registration (an instance announces itself on startup)
service:
  name: payments
  address: 10.0.2.15
  port: 8080
  check:
    http: http://10.0.2.15:8080/healthz
    interval: 10s
```

Other services then query Consul (or `etcd`, or Kubernetes' built-in
service objects) instead of a static config file, and the registry
automatically drops instances that fail their health check — the same
liveness/readiness idea from module 01, now used for internal routing
decisions instead of just external LB decisions.

## The sidecar proxy pattern

```
┌─────────────────────────┐      ┌─────────────────────────┐
│  orders service          │      │  payments service        │
│  ┌────────┐  ┌────────┐ │      │ ┌────────┐  ┌────────┐  │
│  │  app   │──│ sidecar│─┼──────┼─│sidecar │──│  app   │  │
│  │ (code) │  │(Envoy) │ │ mTLS │ │(Envoy) │  │ (code) │  │
│  └────────┘  └────────┘ │      │ └────────┘  └────────┘  │
└─────────────────────────┘      └─────────────────────────┘
```

The application talks to `localhost` — its own sidecar — which handles
finding the real destination, encrypting the connection, retrying on
transient failure, and reporting metrics. The application code never needs
a retry/TLS library for internal calls; it's centralized in the mesh's data
plane (the sidecars) and configured from a control plane (e.g. Istio's
`istiod`, Linkerd's control plane).

## Mutual TLS (mTLS) between services

Regular TLS (Level 2 module 2) proves the *server's* identity to the
client. **Mutual** TLS proves both directions — the client also presents a
certificate, so the server knows which service is calling, not just that
the connection is encrypted.

```yaml
# Conceptual Istio PeerAuthentication: require mTLS for all traffic in a namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

Why this matters operationally: internal networks are often assumed
"trusted" because they're not internet-facing, but that assumption fails
the moment any single host or container on that network is compromised —
mTLS means a compromised service can't silently impersonate another one or
read traffic between two services it's not part of. This is the "zero
trust" principle applied to the internal network, not just the perimeter.

## Retries, timeouts, and circuit breaking

Without care, a slow downstream service causes cascading failure: `orders`
calls `payments`, `payments` is slow, `orders`' threads/connections pile up
waiting, and `orders` itself becomes unresponsive to *its* callers — one
slow service takes down the whole call chain.

```yaml
# Conceptual Envoy/Istio VirtualService: bound the blast radius of a slow downstream
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payments
spec:
  hosts: [payments]
  http:
    - timeout: 2s
      retries:
        attempts: 2
        perTryTimeout: 500ms
        retryOn: 5xx,reset,connect-failure
```

- **Timeout** — never wait indefinitely; fail the call after a bound and
  let the caller decide what to do (return a degraded response, queue for
  later, surface an error).
- **Retries** — only for safe-to-retry (idempotent) requests, with a small
  attempt count and per-try timeout — see the idempotency discussion in
  module 06; retrying a non-idempotent write can duplicate it.
- **Circuit breaking** — after enough consecutive failures to a backend,
  stop sending it traffic for a cooldown period (the internal-network
  version of module 01's `max_fails`/`fail_timeout`), so a struggling
  service gets a chance to recover instead of being hit harder by retries
  from everyone calling it.

## Do you actually need a service mesh

A mesh (Istio, Linkerd, Consul Connect) is real operational overhead —
another control plane to run, upgrade, and understand, and a real learning
curve for anyone debugging why a request failed (was it the app, or the
sidecar's retry/timeout policy?). Rules of thumb:

- **A handful of services, one team, one cluster** — a service registry
  (Consul, or Kubernetes' built-in DNS) plus TLS termination at each
  service, and retry/timeout logic in a shared HTTP client library, gets
  you most of the benefit with far less infrastructure.
- **Dozens of services, multiple teams, need for consistent mTLS/policy
  enforcement across all of them without every team implementing it in
  code** — this is where a mesh's centralization starts paying for its
  overhead.
- Don't adopt a mesh because it's the state of the art; adopt it when the
  concrete pain (inconsistent retry behavior across teams, no visibility
  into cross-service latency, can't enforce mTLS uniformly) is already
  costing more than the mesh would.

## How It Actually Works

**Why the sidecar sees traffic at all without the app knowing.** A sidecar
like Envoy doesn't wrap library calls inside the application — it's a
separate process in the same Pod/network namespace, and traffic reaches it
via `iptables` (or `nftables`) rules injected at container startup (in
Istio, by an init container running `iptables -t nat`) that transparently
redirect all outbound and inbound TCP traffic on the Pod's network
namespace through the sidecar's listener ports before it ever reaches the
real destination. The application still connects to what it thinks is
`payments:8080` directly — the kernel's NAT table silently rewrites the
destination to `localhost:<envoy-port>` first. This is why mesh adoption
needs zero application code changes: the interception happens below the
socket layer the app code ever touches.

**Why mTLS certificate rotation doesn't require restarting services.**
Each sidecar holds a short-lived certificate (often valid for 24 hours or
less) issued by the mesh's control-plane certificate authority (Istio's
`istiod`), and the sidecar itself — not the application — handles
renewal: it requests a new cert before the old one expires and swaps it
into its TLS listener/originator config via the same dynamic
configuration channel (xDS in Istio/Envoy) it uses for routing updates,
with no socket-level interruption to in-flight connections. Short-lived
certs bound the damage window if a workload identity is ever compromised
— an attacker with a stolen cert loses access within hours, not until
someone remembers to rotate it — which is the actual security argument
for automated short-lived mTLS over long-lived static certs.

**Why a retry budget without a circuit breaker can make an outage worse,
not better.** `retries: attempts: 2` means every caller of a struggling
`payments` instance sends up to 3x the request volume at it (the original
plus 2 retries) — under real load, this is a retry storm: the added
retry traffic is exactly what pushes an already-slow backend from
"degraded" into "completely saturated." Circuit breaking (Envoy's outlier
detection, ejecting a host after N consecutive 5xx responses) exists
specifically to break this feedback loop — once a backend is ejected,
callers fail fast instead of retrying against it, giving it a
retry-traffic-free cooldown window to actually recover, which is the
internal-network analog of an LB's `max_fails` pulling a backend from
rotation instead of continuing to hammer it with health checks.

## Exercise

1. Sketch (on paper or in a diagram tool) a 4-service system (e.g. `web` →
   `orders` → `payments`, and `orders` → `inventory`) and, for each edge,
   write down: what happens today if the callee is slow? Unavailable?
   Returns a 5xx?
2. Pick one edge and design a concrete timeout + retry policy for it
   (values, and *why* those values), being explicit about whether the
   request is safe to retry.
3. Without installing a full mesh, add basic service discovery via DNS (or
   a tool like Consul) between two toy services running locally, and
   confirm that killing/restarting one instance doesn't require
   hardcoding a new IP anywhere.
4. Write one paragraph arguing for or against adopting a full service mesh
   for this 4-service system, referencing the trade-off discussion above.
