# 03 · Load Balancing Basics

Once one server can't (or shouldn't, for availability reasons) handle all
your traffic alone, you put a load balancer in front of multiple identical
backend instances. nginx can do this itself with its `upstream` block —
no separate LB product needed for a basic setup.

## The `upstream` block

```nginx
upstream app_backend {
    server 10.0.0.11:3000;
    server 10.0.0.12:3000;
    server 10.0.0.13:3000;
}

server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

By default nginx uses **round robin**: request 1 to server A, request 2 to
B, request 3 to C, request 4 back to A, and so on.

## Balancing algorithms

```nginx
upstream app_backend {
    least_conn;                 # send to the backend with fewest active connections
    server 10.0.0.11:3000;
    server 10.0.0.12:3000;
}
```

- **round robin** (default) — simplest, fine when requests are roughly
  uniform in cost.
- **least_conn** — better when request processing times vary a lot (e.g.
  some requests are slow reports); avoids piling more work onto a backend
  that's already busy.
- **ip_hash** — routes the same client IP to the same backend every time.
  Useful for session affinity ("sticky sessions") when the app keeps
  in-memory session state instead of a shared store — but see the note
  below on why this is a workaround, not a fix.

```nginx
upstream app_backend {
    ip_hash;
    server 10.0.0.11:3000;
    server 10.0.0.12:3000;
}
```

**Prefer stateless backends over sticky sessions.** `ip_hash` breaks down
behind carrier-grade NAT (many users share one IP) and doesn't rebalance
well when a backend is added/removed. The durable fix is to keep session
state in a shared store (Redis, a database) so *any* backend can serve
*any* request — treat `ip_hash` as a stopgap, not the design goal.

## Weighting and marking backends down

```nginx
upstream app_backend {
    server 10.0.0.11:3000 weight=3;   # gets ~3x the traffic of a weight=1 server
    server 10.0.0.12:3000;
    server 10.0.0.13:3000 backup;      # only used if all non-backup servers are down
    server 10.0.0.14:3000 down;        # manually taken out of rotation (e.g. for maintenance)
}
```

`weight` is handy when backends have different capacity (e.g. one bigger
instance during a migration); `down` is a manual, config-file way to drain
a node before you do maintenance on it — reload nginx after adding it,
finish the maintenance, remove it, reload again.

## Passive health checks (open source nginx)

Open-source nginx doesn't have active health checks (polling `/health` on a
timer) built in — that's an nginx Plus feature. What it does have is
**passive** health checking: if a backend fails to connect or times out,
nginx marks it temporarily unavailable and stops sending it traffic for a
cooldown period.

```nginx
upstream app_backend {
    server 10.0.0.11:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.12:3000 max_fails=3 fail_timeout=30s;
}
```

`max_fails=3 fail_timeout=30s` means: after 3 failed attempts within 30
seconds, mark this backend down for 30 seconds, then try it again. For real
active health checks (proactively probing `/health` even with zero live
traffic) you'd reach for HAProxy, nginx Plus, or a dedicated LB/service
mesh — know this is the boundary of what stock nginx gives you for free.

## Why load balancing = one form of high availability

If backend `10.0.0.11` crashes, requests keep flowing to `.12` and `.13`
with (at worst) the in-flight requests to `.11` failing once, then
`max_fails` kicking in to stop sending it new traffic. This is the same
mechanism behind zero-downtime deploys (module 5) — you can take one
backend out of the pool, deploy to it, health-check it, and put it back,
one at a time, with no visible downtime.

## Worked example: three backends, one balancer

```bash
# on the LB host, simulate three backends locally on different ports
for p in 3001 3002 3003; do
  (echo "server $p"; python3 -m http.server $p --bind 127.0.0.1) &
done

sudo tee /etc/nginx/sites-available/lbdemo.conf <<'EOF'
upstream demo_backend {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}
server {
    listen 8080;
    location / {
        proxy_pass http://demo_backend;
        proxy_set_header Host $host;
    }
}
EOF
sudo ln -sf /etc/nginx/sites-available/lbdemo.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

for i in $(seq 1 6); do curl -s http://localhost:8080/ -o /dev/null -w "%{http_code}\n"; done
kill %1 %2 %3   # stop the background python servers when done
```

Watching nginx's access log (`sudo tail -f /var/log/nginx/access.log`) while
curling repeatedly shows requests distributed across backends.

## Exercise

1. Run three instances of a toy HTTP server on different local ports and
   put an nginx `upstream` in front with `least_conn`.
2. Kill one backend process mid-test and confirm (via repeated `curl`) that
   nginx stops sending it traffic after `max_fails`/`fail_timeout`, and that
   overall requests keep succeeding.
3. Switch the upstream to `ip_hash` and confirm from a single client that
   all requests land on the same backend every time.
