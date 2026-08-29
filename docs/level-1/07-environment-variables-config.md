# 07 · Environment Variables & Config Management

Hardcoding database passwords, API keys, and per-environment settings
directly into source code is one of the most common ways secrets end up
leaked in a git history. This module covers the standard pattern: keep
config out of code, injected at runtime via environment variables and config
files.

## Why environment variables

The [twelve-factor app](https://12factor.net/config) methodology's config
principle, in short: **anything that varies between environments (dev,
staging, prod) — credentials, hostnames, feature flags — belongs in the
environment, not in the codebase.** The same built artifact should run
correctly in any environment purely by changing what's injected around it.

Benefits in practice:

- The same code (and the same deploy script) works in every environment.
- Secrets never need to be committed to version control.
- Different engineers/environments can have different values without
  touching code.

## Setting environment variables for a systemd service

Two supported patterns, and they compose:

**1. Inline in the unit file** — fine for non-secret, rarely-changing values:

```ini
[Service]
Environment=APP_ENV=production
Environment=LOG_LEVEL=info
```

**2. An external env file** — better for secrets and per-host values, kept
out of the unit file (which often lives in version control alongside your
infra config):

```ini
[Service]
EnvironmentFile=/etc/myapp/myapp.env
```

```bash
# /etc/myapp/myapp.env — KEY=value, one per line, no quotes needed
APP_ENV=production
DATABASE_URL=postgresql://myapp:s3cr3t@localhost:5432/myapp
API_KEY=sk_live_abc123
LOG_LEVEL=info
```

Lock this file down — it holds secrets:

```bash
sudo chown root:deploy /etc/myapp/myapp.env
sudo chmod 640 /etc/myapp/myapp.env
```

`640` = owner (`root`) can read/write, group (`deploy`) can read, others get
nothing. The service's `User=deploy` (or whichever group you set) can then
read it via `EnvironmentFile=`, but random other users on the box can't.

After changing the env file, reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
```

## Reading environment variables in your app

```bash
# Shell script
echo "Running in ${APP_ENV:-development}"     # default if unset

# Python
import os
db_url = os.environ["DATABASE_URL"]            # raises if missing (fail fast)
log_level = os.environ.get("LOG_LEVEL", "info")  # default if missing

# Node.js
const dbUrl = process.env.DATABASE_URL;
const logLevel = process.env.LOG_LEVEL || "info";
```

Prefer failing loudly (`os.environ[...]`, no default) for values the app
genuinely cannot run without — a missing `DATABASE_URL` should crash on
startup with a clear error, not silently connect to `undefined` and fail
confusingly three requests later.

## `.env` files for local development

Outside of systemd, `.env` files loaded by a library (`python-dotenv`,
`dotenv` for Node) are the common local-dev equivalent:

```bash
# .env — NEVER commit this file
DATABASE_URL=postgresql://myapp:localpass@localhost:5432/myapp_dev
API_KEY=sk_test_dummy
LOG_LEVEL=debug
```

Always add it to `.gitignore` immediately, and commit a `.env.example` with
the same keys and placeholder/dummy values so teammates know what's needed:

```bash
# .env.example — safe to commit
DATABASE_URL=postgresql://user:pass@localhost:5432/dbname
API_KEY=
LOG_LEVEL=debug
```

```text
# .gitignore
.env
*.env
!.env.example
```

## Per-environment config layering

A common pattern for anything beyond a handful of variables: a small,
non-secret YAML/JSON per environment, with secrets still injected via env
vars layered on top.

```yaml
# config/production.yaml
log_level: info
request_timeout_seconds: 30
feature_flags:
  new_dashboard: true
```

```yaml
# config/staging.yaml
log_level: debug
request_timeout_seconds: 10
feature_flags:
  new_dashboard: true
```

The app picks the file based on `APP_ENV`, then still pulls secrets
(`DATABASE_URL`, `API_KEY`) from the environment rather than from these
files — because these files *are* safe to commit, and secrets never should
be.

## Worked example: wiring config end-to-end

```bash
# 1. Create the secret env file
sudo mkdir -p /etc/myapp
cat <<'EOF' | sudo tee /etc/myapp/myapp.env > /dev/null
APP_ENV=production
DATABASE_URL=postgresql://myapp:s3cr3t@localhost:5432/myapp
LOG_LEVEL=info
EOF
sudo chown root:deploy /etc/myapp/myapp.env
sudo chmod 640 /etc/myapp/myapp.env

# 2. Point the unit file at it (add this line under [Service])
#    EnvironmentFile=/etc/myapp/myapp.env

sudo systemctl daemon-reload
sudo systemctl restart myapp

# 3. Confirm the process actually received the variables
sudo systemctl show myapp -p Environment
# or, given the PID:
sudo cat /proc/$(pgrep -f myapp)/environ | tr '\0' '\n' | grep APP_ENV
```

## Exercise

1. Create `/etc/myapp/myapp.env` with at least three variables, lock it down
   to `640` owned by `root:deploy`.
2. Add `EnvironmentFile=/etc/myapp/myapp.env` to the `hello.service` unit
   from module 4 and modify `hello.sh` to print one of those variables each
   loop iteration.
3. `daemon-reload` + `restart`, then confirm via `journalctl -u hello` that
   the variable's value is showing up in the output.
4. Change the value in the env file, restart, and confirm the new value
   takes effect — without touching `hello.sh` or the unit file at all.
