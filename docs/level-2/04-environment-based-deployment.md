# 04 · Environment-Based Deployment (staging vs. prod)

Deploying straight to production with no intermediate check is how small
bugs become customer-facing incidents. This module is about running the
same app in at least two environments — staging (a production-like
rehearsal space) and production (the real thing) — with a deploy process
that promotes the exact same artifact through both, rather than rebuilding
per environment.

## The core principle: build once, promote the artifact

If staging and prod are built from source *separately* (even from the same
git commit), you can still end up with different dependency versions
resolved, different build-time environment leaking in, or a compiler/base
image drift between the two builds. The fix: build one artifact (a Docker
image, a tarball, a versioned package) and deploy that *exact same* binary
object to staging, verify it, then deploy that *exact same* object to prod.

```bash
# build once, tag with the git SHA
GIT_SHA=$(git rev-parse --short HEAD)
tar czf "release-${GIT_SHA}.tar.gz" dist/ package.json

# staging
scp "release-${GIT_SHA}.tar.gz" deploy@staging.internal:/opt/releases/
ssh deploy@staging.internal "cd /opt/releases && tar xzf release-${GIT_SHA}.tar.gz -C /opt/myapp"

# only after staging checks pass, same artifact to prod
scp "release-${GIT_SHA}.tar.gz" deploy@prod1.internal:/opt/releases/
ssh deploy@prod1.internal "cd /opt/releases && tar xzf release-${GIT_SHA}.tar.gz -C /opt/myapp"
```

## Separating config from code per environment

The app code is identical across environments; only configuration differs
(database host, API keys, log level, feature flags). Keep per-environment
config out of the artifact entirely — inject it at deploy/runtime:

```bash
# /etc/myapp/staging.env
DATABASE_URL=postgres://staging-db.internal/myapp
LOG_LEVEL=debug
FEATURE_NEW_CHECKOUT=true

# /etc/myapp/production.env
DATABASE_URL=postgres://prod-db.internal/myapp
LOG_LEVEL=warn
FEATURE_NEW_CHECKOUT=false
```

```ini
# systemd unit, same file used in both environments
[Service]
EnvironmentFile=/etc/myapp/%i.env
ExecStart=/opt/myapp/bin/server
```

Using a systemd template unit (`myapp@.service`) with `%i` substituted at
enable time (`systemctl enable myapp@production`) lets one unit file serve
both environments, differing only by which env file gets loaded.

## Why staging must resemble production

Staging is only useful if it catches real bugs before prod does. Common
ways staging silently stops doing that:

- Staging runs on a much smaller/bigger instance — masks resource-limit
  bugs (OOM kills, connection pool exhaustion) that only show at prod scale.
- Staging's database has stale or synthetic data with none of prod's messy
  edge cases (nulls, unicode, huge rows).
- Staging skips the reverse proxy / TLS layer that prod has — misses
  header-forwarding bugs (module 1).

Keep staging as close to a scaled-down clone of prod's topology as
practical: same OS version, same reverse proxy config (different
`server_name`/cert), same deploy mechanism.

## A deploy script that enforces the promotion order

```bash
#!/usr/bin/env bash
set -euo pipefail

SHA=$(git rev-parse --short HEAD)
ARTIFACT="release-${SHA}.tar.gz"

echo "Building ${ARTIFACT}..."
tar czf "$ARTIFACT" dist/

echo "Deploying to staging..."
scp "$ARTIFACT" deploy@staging.internal:/opt/releases/
ssh deploy@staging.internal "/opt/myapp/bin/install-release.sh $ARTIFACT"

echo "Running smoke tests against staging..."
curl -sf https://staging.example.com/health || { echo "staging smoke test failed"; exit 1; }

read -p "Staging looks good. Deploy ${SHA} to PRODUCTION? [y/N] " confirm
[[ "$confirm" == "y" ]] || { echo "Aborted."; exit 1; }

echo "Deploying to production..."
scp "$ARTIFACT" deploy@prod1.internal:/opt/releases/
ssh deploy@prod1.internal "/opt/myapp/bin/install-release.sh $ARTIFACT"
curl -sf https://example.com/health || { echo "prod smoke test failed — investigate immediately"; exit 1; }

echo "Deployed ${SHA} to production."
```

The `read -p` confirmation gate is a deliberate manual checkpoint — cheap
insurance until you trust the pipeline enough to automate the promotion
(module 7 covers making this fully automatic in CI/CD).

## Worked example: two environments on one host (for practice)

On a single practice VM you can simulate two environments with two ports
and two env files instead of two machines:

```bash
sudo mkdir -p /etc/myapp /opt/myapp-staging /opt/myapp-production
printf 'PORT=3001\nLOG_LEVEL=debug\n' | sudo tee /etc/myapp/staging.env
printf 'PORT=3002\nLOG_LEVEL=warn\n'  | sudo tee /etc/myapp/production.env
# deploy the same release tarball into both dirs, then start each with its own env file
```

## Exercise

1. Set up two systemd services (or two directories/ports on one VM)
   representing staging and production, each with its own env file.
2. Write a deploy script that builds one artifact, deploys it to "staging",
   curls a health endpoint, and only proceeds to "production" after that
   check passes and a manual confirmation.
3. Deliberately break the staging health check and confirm the script stops
   before touching "production".
