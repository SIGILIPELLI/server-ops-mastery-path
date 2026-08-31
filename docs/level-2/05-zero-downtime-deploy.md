# 05 · Zero-Downtime Deploy Patterns

A "restart the service to deploy" approach drops every in-flight request
for the seconds the process is down. Zero-downtime deployment means users
never see an error during a release. This module covers the two most
common patterns you can implement with tools you already have: nginx +
systemd.

## Pattern 1: rolling restart across multiple backends

If you already load-balance across N backends (module 3), you can deploy
one at a time: take it out of rotation, update it, health-check it, put it
back, repeat.

```bash
#!/usr/bin/env bash
set -euo pipefail
BACKENDS=(10.0.0.11 10.0.0.12 10.0.0.13)
ARTIFACT="$1"

for host in "${BACKENDS[@]}"; do
  echo "Draining $host from nginx upstream..."
  ssh lb.internal "sudo sed -i 's/server ${host}:3000;/server ${host}:3000 down;/' /etc/nginx/sites-available/app.conf"
  ssh lb.internal "sudo nginx -t && sudo systemctl reload nginx"
  sleep 5   # let in-flight requests to this backend finish

  echo "Deploying to $host..."
  scp "$ARTIFACT" "deploy@${host}:/opt/releases/"
  ssh "deploy@${host}" "/opt/myapp/bin/install-release.sh $ARTIFACT && sudo systemctl restart myapp"

  echo "Health-checking $host..."
  for i in $(seq 1 10); do
    ssh "deploy@${host}" "curl -sf http://127.0.0.1:3000/health" && break
    [[ $i -eq 10 ]] && { echo "FAILED health check on $host, aborting rollout"; exit 1; }
    sleep 2
  done

  echo "Re-adding $host to rotation..."
  ssh lb.internal "sudo sed -i 's/server ${host}:3000 down;/server ${host}:3000;/' /etc/nginx/sites-available/app.conf"
  ssh lb.internal "sudo nginx -t && sudo systemctl reload nginx"
  echo "$host done."
done
echo "Rolling deploy complete."
```

At any point during this loop, 2 of the 3 backends are serving live
traffic on the new *or* old version — clients never hit a backend that's
mid-restart. Aborting on a failed health check (rather than plowing ahead)
means a bad build takes down at most one-third of your capacity, not all
of it.

## Pattern 2: socket handoff on a single host (systemd socket activation)

When you only have one host, you can still avoid downtime by having the
*old* process keep serving until the *new* process is confirmed ready,
using two separate ports behind nginx and swapping which one nginx points
at — this is effectively blue-green on one box, and is expanded into the
full pattern in the module 10 capstone. The minimal version:

```bash
# old process on :3000 keeps running
# start the new version on :3001
sudo systemctl start myapp-new@3001

# health check the new one directly
curl -sf http://127.0.0.1:3001/health || { echo "new version unhealthy, not switching"; exit 1; }

# flip nginx's upstream to point at :3001
sudo sed -i 's/127.0.0.1:3000/127.0.0.1:3001/' /etc/nginx/sites-available/app.conf
sudo nginx -t && sudo systemctl reload nginx

# only now stop the old process
sudo systemctl stop myapp@3000
```

`nginx reload` (not `restart`) is itself zero-downtime for nginx's own
process — it spawns new worker processes with the new config and lets old
workers finish in-flight requests before exiting. That property is what
makes flipping the upstream target safe: there's no window where nginx
itself is down.

## Graceful shutdown on the app side

Zero-downtime deploys require the app to actually finish in-flight requests
before dying, not drop them when it receives `SIGTERM`. Systemd sends
`SIGTERM` on stop/restart by default:

```ini
[Service]
ExecStart=/opt/myapp/bin/server
KillSignal=SIGTERM
TimeoutStopSec=30
```

Your app's code needs to listen for `SIGTERM` and stop accepting *new*
connections while letting existing ones complete, e.g. (Node.js):

```js
process.on('SIGTERM', () => {
  server.close(() => process.exit(0));  // stop accepting new conns, finish existing ones, then exit
  setTimeout(() => process.exit(1), 25000); // hard exit if graceful shutdown hangs
});
```

`TimeoutStopSec=30` in the unit must be longer than your app's own graceful
shutdown budget, or systemd sends `SIGKILL` before the app finishes
cleanly.

## Database migrations: the part that actually breaks zero-downtime

Schema changes are the usual reason "zero-downtime" deploys still cause an
outage: if the new code expects a column the old code's still-running
instances don't know about (or vice versa, during a rolling deploy where
old and new code run simultaneously), requests fail. The rule: make
migrations **backward-compatible with the previous version of the code**,
and split breaking changes into multiple deploys:

1. Deploy 1: add the new column as nullable; old code ignores it, new code
   (not yet deployed) will use it.
2. Deploy 2: ship code that writes to the new column, still reading/falling
   back to the old one.
3. Deploy 3 (after backfill): ship code that only uses the new column.
4. Deploy 4: drop the old column.

Never combine "add column" and "drop old column" in the same deploy as the
code change that depends on both being done — that's the single most common
cause of "zero-downtime" deploys that weren't.

## Exercise

1. Extend the rolling-restart script above (or the load-balancing exercise
   from module 3) to loop over your 3 toy backends, draining, restarting,
   and health-checking each one in turn.
2. While the rollout is running, hit the load balancer in a tight loop from
   a second terminal (`while true; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/; sleep 0.2; done`)
   and confirm you see zero non-200 responses during the whole rollout.
3. Add a `SIGTERM` handler to a toy app that sleeps 3 seconds before exiting
   (simulating in-flight request draining), and confirm systemd's
   `TimeoutStopSec` is set high enough that `systemctl restart` doesn't kill
   it mid-drain.
