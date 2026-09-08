# 06 · Writing a Build/Deploy Script

Manually SSHing in and typing commands every time you ship a change doesn't
scale and isn't repeatable. This module builds a real, defensive shell
script that pulls code, builds it, and restarts the service — the kind of
script that later evolves into full CI/CD (Level 2).

## Script safety basics

Every non-trivial shell script in this course starts with the same header:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

- `set -e` — exit immediately if any command fails (non-zero exit code),
  instead of plowing ahead with a broken state.
- `set -u` — treat use of an undefined variable as an error, catching typos
  like `$DEPOY_DIR` instead of `$DEPLOY_DIR`.
- `set -o pipefail` — in a pipeline (`a | b | c`), fail if *any* stage fails,
  not just the last one (by default, bash only looks at the last command's
  exit status).

## A basic deploy script

Assume a simple setup: a git repo holding the app, deployed to
`/opt/myapp`, run as a systemd service called `myapp` (from module 4).

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_DIR="/opt/myapp"
REPO_URL="git@github.com:example/myapp.git"
BRANCH="main"
SERVICE_NAME="myapp"
LOG_FILE="/var/log/myapp-deploy.log"

log() {
    echo "[$(date -Iseconds)] $*" | tee -a "$LOG_FILE"
}

log "Starting deploy of $SERVICE_NAME (branch: $BRANCH)"

if [ ! -d "$APP_DIR/.git" ]; then
    log "No existing checkout found — cloning fresh"
    git clone --branch "$BRANCH" "$REPO_URL" "$APP_DIR"
else
    log "Existing checkout found — fetching latest"
    cd "$APP_DIR"
    git fetch origin "$BRANCH"
    git reset --hard "origin/$BRANCH"
fi

cd "$APP_DIR"
CURRENT_SHA="$(git rev-parse --short HEAD)"
log "Deploying commit $CURRENT_SHA"

log "Installing dependencies"
if [ -f "requirements.txt" ]; then
    python3 -m venv venv --upgrade-deps
    ./venv/bin/pip install -r requirements.txt
elif [ -f "package.json" ]; then
    npm ci --omit=dev
fi

log "Restarting service"
sudo systemctl restart "$SERVICE_NAME"

sleep 2
if systemctl is-active --quiet "$SERVICE_NAME"; then
    log "Deploy succeeded — $SERVICE_NAME is active (commit $CURRENT_SHA)"
else
    log "ERROR: $SERVICE_NAME failed to start after deploy!"
    systemctl status "$SERVICE_NAME" --no-pager | tee -a "$LOG_FILE"
    exit 1
fi
```

Walking through the design choices:

- **Idempotent clone-or-update** — the script works correctly whether this
  is the first deploy ever or the hundredth; it checks for an existing `.git`
  directory rather than assuming.
- **`git reset --hard origin/$BRANCH`** — forces the working tree to exactly
  match the remote branch, discarding any local drift (which shouldn't exist
  on a deploy target, but defends against it if it does).
- **Logging with a timestamp to both stdout and a file** (`tee -a`) — so you
  can watch it live during a manual deploy and still have a record after the
  fact.
- **Post-deploy health check** — `systemctl is-active --quiet` after a short
  sleep, so the script actually fails loudly (`exit 1`) if the restart didn't
  result in a running service, rather than reporting false success.

## Detecting the runtime and picking a build step

Real projects vary in how they're built. A slightly more general pattern:

```bash
detect_and_build() {
    if [ -f "package.json" ]; then
        npm ci --omit=dev
        [ -f "webpack.config.js" ] && npm run build
    elif [ -f "requirements.txt" ]; then
        python3 -m venv venv --upgrade-deps
        ./venv/bin/pip install -r requirements.txt
    elif [ -f "go.mod" ]; then
        go build -o bin/app .
    else
        log "WARNING: no recognized build manifest found, skipping build step"
    fi
}
```

## Worked example: running the deploy script end-to-end

```bash
sudo mkdir -p /var/log
sudo touch /var/log/myapp-deploy.log
sudo chown deploy:deploy /var/log/myapp-deploy.log

chmod +x deploy.sh
./deploy.sh
```

Expected log output on a successful first-time run:

```text
[2026-08-29T10:02:11+00:00] Starting deploy of myapp (branch: main)
[2026-08-29T10:02:11+00:00] No existing checkout found — cloning fresh
[2026-08-29T10:02:14+00:00] Deploying commit 9f3a1c2
[2026-08-29T10:02:14+00:00] Installing dependencies
[2026-08-29T10:02:19+00:00] Restarting service
[2026-08-29T10:02:21+00:00] Deploy succeeded — myapp is active (commit 9f3a1c2)
```

If the service fails to come up, you get the failure line plus a full
`systemctl status` dump right in the log — no need to SSH back in separately
to find out why.

## How It Actually Works

**`set -euo pipefail` and shell exit-status propagation.** By default, a
shell script keeps running after a failing command — each command sets `$?`
but nothing checks it automatically. `set -e` makes the shell exit
immediately if any simple command returns non-zero (with documented
exceptions: commands in an `if`/`while` test, or before `&&`/`||`). `set -u`
turns a reference to an unset variable into a hard error instead of silently
expanding to an empty string — the classic case is a typo'd variable in
`rm -rf $DIR/` becoming `rm -rf /` when `$DIR` is empty. `set -o pipefail`
fixes a subtler gap: without it, `false | true` exits 0 because only the
*last* command in a pipeline determines `$?`; with it, the pipeline's exit
status is the rightmost non-zero command, so a failing step upstream of a
pipe isn't masked.

**Why deploy scripts should be idempotent at the OS level.** Re-running
`mkdir dir` errors if `dir` exists, but `mkdir -p dir` calls the same
`mkdir()` syscall guarded by a existence check, succeeding either way. This
distinction matters for deploy scripts specifically because they're expected
to run repeatedly (retries after partial failure, redeploys) — the OS-level
side effects (files, symlinks, systemd units) need operations that converge
to the same end state regardless of starting state, rather than commands
that assume a pristine target.

## Exercise

1. Take the `hello.service` app from module 4 and turn it into a tiny git
   repo (`git init`, commit `hello.sh` and the unit file).
2. Adapt the deploy script above to clone that repo to `/opt/hello`, copy
   the unit file into place if it changed, and restart the `hello` service.
3. Run it once against an empty target directory (first deploy) and once
   again after committing a small change (e.g., changing the sleep interval)
   to confirm the "existing checkout" branch works too.
4. Deliberately break the app (e.g., a syntax error in `hello.sh`) and
   confirm the script's post-deploy health check catches it and exits
   non-zero instead of silently reporting success.
