# 02 · DNS & Load Balancer HA Patterns

Module 01 made the load balancer itself highly available with a floating IP
on the same subnet. That doesn't help once you need redundancy *across*
subnets, data centers, or regions — DNS is the tool that operates above the
network layer and can point clients at entirely different IPs depending on
health, location, or load.

## DNS as a (slow, coarse) load balancer

A DNS record can list multiple IPs:

```
app.example.com.  300  IN  A  203.0.113.10
app.example.com.  300  IN  A  203.0.113.20
```

Resolvers typically return them in rotation (round-robin DNS) or clients
pick one. This distributes *new connections* across IPs, but DNS has no idea
whether `203.0.113.20` is actually healthy — a plain `A` record is "dumb":
it answers the same way whether the backend is up or on fire. That's why
production setups use **health-checked DNS** from a provider that supports
it (Route 53, Cloudflare, NS1): the provider polls each endpoint and stops
handing out IPs that fail.

```
# Conceptual Route 53 health-checked record (via console/Terraform, not zone file syntax)
app.example.com → failover routing policy
  primary:   203.0.113.10  (health check: GET /healthz every 30s)
  secondary: 203.0.113.20  (only answered if primary's health check fails)
```

## Why DNS failover is slow: TTL and caching

```
app.example.com.  300  IN  A  203.0.113.10
```

`300` is the TTL in seconds — how long resolvers and clients are allowed to
cache this answer. Even with instant server-side detection, a client that
already cached the old IP for 300 seconds keeps trying it until the TTL
expires. Some resolvers and OS stub resolvers also ignore TTL or cache
longer than they should ("TTL disrespect").

Practical implications:

- Lower the TTL (e.g. to 60s or less) **before** a planned migration or
  failover event, not during it — the lowered TTL itself needs to propagate
  first.
- Never rely on DNS failover alone for sub-minute recovery; pair it with a
  fast mechanism underneath (keepalived VIP, anycast) for the failures that
  need to disappear in seconds, and use DNS failover for the coarser
  case — an entire region or data center going down.
- Don't set TTLs extremely low (e.g. 5s) permanently "just in case" — it
  multiplies query volume against your DNS provider and adds latency for
  every cache miss, for a benefit you'll rarely use.

## GeoDNS and latency-based routing

Beyond simple failover, DNS can route different clients to different,
equally-healthy endpoints based on where they are:

- **Geolocation routing** — clients in the EU get the EU region's IP,
  clients in the US get the US region's IP. Good for data-residency
  requirements as well as latency.
- **Latency-based routing** — the DNS provider measures round-trip time
  from each of its edge locations to each of your regions and answers with
  whichever is fastest for that resolver, not a fixed map.
- **Weighted routing** — split traffic e.g. 90/10 between two regions for a
  canary rollout, independent of health.

These all still combine with health checks: a region can be "correct for
this client" and "excluded because unhealthy" at the same time, and the
provider picks the next-best healthy option.

## Load balancer HA patterns beyond one VIP pair

**Anycast** — the same IP address is announced from multiple physical
locations via BGP; the network itself routes each client to the topologically
nearest healthy announcer. This is how large CDNs and DNS providers (e.g.
`8.8.8.8`) achieve HA without relying on DNS failover at all — it requires
BGP control, which is typically only available to you via a cloud provider's
anycast IP product, not something you run yourself on a couple of VMs.

**Layered LB** — a common real-world shape:

```
Internet
   │
   ▼
DNS (health-checked, geo/latency routing)
   │
   ▼
Regional load balancer (cloud LB or keepalived pair, HA within the region)
   │
   ▼
App server pool (health-checked by the regional LB, per module 01)
```

Each layer handles a different blast radius: DNS handles "a whole region is
gone," the regional LB handles "one backend is gone," and health checks
within each layer decide what's currently eligible to receive traffic.

## Worked example: simulating DNS failover with dnsmasq

You can rehearse the failover *logic* locally without owning real DNS
infrastructure, using `dnsmasq` as a stand-in authoritative resolver and a
script that rewrites its records based on health checks.

```bash
sudo apt install -y dnsmasq
```

`/etc/dnsmasq.d/app.conf`:

```
address=/app.test/203.0.113.10
```

A small "failover controller" that polls the primary and rewrites the
record to the secondary if it goes unhealthy:

```bash
#!/usr/bin/env bash
# dns-failover.sh — poll primary; if it fails 3 times, repoint app.test at the secondary
set -euo pipefail
PRIMARY=203.0.113.10
SECONDARY=203.0.113.20
CONF=/etc/dnsmasq.d/app.conf
FAILS=0

while true; do
  if curl -sf --max-time 2 "http://$PRIMARY/healthz" > /dev/null; then
    FAILS=0
    CURRENT=$PRIMARY
  else
    FAILS=$((FAILS + 1))
    CURRENT=$([ "$FAILS" -ge 3 ] && echo "$SECONDARY" || echo "$PRIMARY")
  fi

  echo "address=/app.test/$CURRENT" | sudo tee "$CONF" > /dev/null
  sudo systemctl reload dnsmasq
  sleep 10
done
```

This is a toy — real DNS failover uses your provider's health-check feature
— but it demonstrates the exact mechanism: something has to detect the
failure, then rewrite what DNS answers, and every client is bound by the
TTL until it re-resolves.

## Exercise

1. Set up `dnsmasq` as above and confirm `dig app.test @127.0.0.1` (or
   `nslookup app.test 127.0.0.1`) returns the primary IP.
2. Run the `dns-failover.sh` script against two toy HTTP servers (like the
   `python3 -m http.server` trick from Level 2). Kill the primary and time
   how long it takes for `dig` to start returning the secondary.
3. Add a TTL to the `dnsmasq` record (`local-ttl=30`) and, using a small
   client script that resolves once and caches for the TTL, demonstrate that
   the client keeps hitting the (now-dead) primary until its cached TTL
   expires — even though the DNS record itself already changed.
