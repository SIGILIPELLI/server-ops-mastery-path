# 10 · Capstone — Blue-Green Deploy Pipeline

Blue-green deployment runs two complete, identical production environments
("blue" and "green"). At any time, one is live (serving all traffic) and
the other is idle. You deploy the new version to the idle one, test it for
real, then switch traffic over atomically. If anything's wrong, switching
back is instant. This capstone wires together modules 1-9 into one
end-to-end pipeline.

## Topology

```
                    ┌─────────────┐
  clients ────────▶ │ nginx (LB)  │
                    └──────┬──────┘
                    active pool ──▶ blue  (v1, currently live)
                    idle pool   ──▶ green (v2, being deployed/tested)
```

```nginx
# /etc/nginx/sites-available/app.conf
upstream blue {
    server 127.0.0.1:3001;
}
upstream green {
    server 127.0.0.1:3002;
}
upstream active {
    server 127.0.0.1:3001;   # this line is what we flip: points at blue OR green
}

server {
    listen 80;
    server_name app.example.com;
    location / {
        proxy_pass http://active;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## The deploy script

```bash
#!/usr/bin/env bash
# blue-green-deploy.sh — deploys $1 (a release tarball) to the currently-idle color
set -euo pipefail

ARTIFACT="$1"
ACTIVE_CONF="/etc/nginx/sites-available/app.conf"

# 1. figure out which color is currently live by reading the nginx config
if grep -q "server 127.0.0.1:3001;" <(sed -n '/upstream active/,/}/p' "$ACTIVE_CONF"); then
  LIVE_COLOR=blue; LIVE_PORT=3001
  IDLE_COLOR=green; IDLE_PORT=3002
else
  LIVE_COLOR=green; LIVE_PORT=3002
  IDLE_COLOR=blue; IDLE_PORT=3001
fi
echo "Live: $LIVE_COLOR (:$LIVE_PORT)  Idle target: $IDLE_COLOR (:$IDLE_PORT)"

# 2. deploy the new artifact to the idle color only — live traffic is untouched
sudo systemctl stop "myapp-${IDLE_COLOR}"
sudo rm -rf "/opt/myapp-${IDLE_COLOR}/current"
sudo mkdir -p "/opt/myapp-${IDLE_COLOR}/current"
sudo tar xzf "$ARTIFACT" -C "/opt/myapp-${IDLE_COLOR}/current"
sudo systemctl start "myapp-${IDLE_COLOR}"

# 3. health-check the idle color directly, BEFORE it gets any real traffic
echo "Health-checking $IDLE_COLOR..."
for i in $(seq 1 15); do
  if curl -sf "http://127.0.0.1:${IDLE_PORT}/health"; then
    echo "OK"
    break
  fi
  [ "$i" -eq 15 ] && { echo "FAILED: $IDLE_COLOR never became healthy, aborting"; exit 1; }
  sleep 2
done

# 4. run a real smoke test against it (not just /health — hit a couple of real endpoints)
curl -sf "http://127.0.0.1:${IDLE_PORT}/api/version" | grep -q "$(git rev-parse --short HEAD)" \
  || { echo "FAILED: version mismatch on $IDLE_COLOR, aborting"; exit 1; }

# 5. flip nginx's "active" upstream to the now-verified idle color
sudo sed -i "/upstream active/,/}/ s/server 127.0.0.1:${LIVE_PORT};/server 127.0.0.1:${IDLE_PORT};/" "$ACTIVE_CONF"
sudo nginx -t && sudo systemctl reload nginx
echo "Switched live traffic to $IDLE_COLOR."

# 6. give it a few seconds under real traffic, then confirm it's still healthy
sleep 10
curl -sf "http://127.0.0.1:${IDLE_PORT}/health" || { echo "UNHEALTHY under real traffic — rolling back"; \
  sudo sed -i "/upstream active/,/}/ s/server 127.0.0.1:${IDLE_PORT};/server 127.0.0.1:${LIVE_PORT};/" "$ACTIVE_CONF"; \
  sudo nginx -t && sudo systemctl reload nginx; exit 1; }

echo "Deploy complete. $IDLE_COLOR is now live; $LIVE_COLOR is idle and can be updated next time."
```

Everything from earlier modules shows up here on purpose:

- **module 1** — nginx upstream/reverse proxy is the switch mechanism.
- **module 3** — `upstream` blocks, same primitive as load balancing.
- **module 4** — one artifact promoted, never rebuilt per target.
- **module 5** — zero traffic ever hits a half-restarted process; the old
  color keeps serving until the new one is proven healthy.
- **module 6** — secrets for both colors come from per-color env files, not
  baked into the artifact.
- **module 7** — this whole script is what a CI/CD "deploy" job calls.
- **module 8/9** — the `/health` check and post-switch verification are the
  monitoring hooks that make step 6's automatic rollback possible.

## Instant rollback

Because the previous color is still fully running (just idle, not
destroyed) immediately after a switch, rollback is the same one-line
`sed` + `nginx reload` used in step 6's automatic case — no redeploying
old code, no waiting for a build:

```bash
sudo sed -i "/upstream active/,/}/ s/server 127.0.0.1:${NEW_PORT};/server 127.0.0.1:${OLD_PORT};/" /etc/nginx/sites-available/app.conf
sudo nginx -t && sudo systemctl reload nginx
```

## Trade-offs to know

- Blue-green needs 2x the running capacity of a single environment (both
  colors are fully provisioned, even though only one serves traffic at a
  time) — costlier than the rolling-restart pattern from module 5, but
  simpler to reason about and instantly reversible.
- Database migrations must still follow the backward-compatible,
  multi-step pattern from module 5 — blue and green share one database, so
  a breaking schema change affects both colors regardless of which is
  "live."

## Exercise

1. Set up two systemd services (`myapp-blue`, `myapp-green`) on different
   ports, an nginx config with the `upstream active` indirection above, and
   deploy a "v1" build to blue and switch active to it.
2. Run the full script to deploy "v2" to green, watch it health-check,
   flip, and verify.
3. Force a failure in the post-switch health check (e.g. crash the new
   process right after cutover) and confirm the script's automatic
   rollback correctly repoints `active` back to the previous color.
