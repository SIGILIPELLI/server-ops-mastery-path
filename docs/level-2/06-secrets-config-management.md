# 06 · Secrets & Configuration Management

Level 1 covered environment variables for basic config. This module is
about the harder half: secrets — database passwords, API keys, TLS private
keys — which need everything regular config needs (per-environment values,
easy rotation) plus protection from being read by the wrong person or
committed to git by accident.

## Rule 1: secrets never go in git

```bash
# .gitignore
*.env
secrets/
*.pem
*.key
```

If a secret was ever committed, rotating it is mandatory — removing the
file in a later commit does **not** remove it from git history. Anyone
with clone access can still retrieve it from an old commit:

```bash
git log --all --full-history -- secrets.env    # find when it was added
```

Rewriting history (`git filter-repo`, BFG) to strip it out is a real
option but treat it as cleanup *after* rotating the leaked secret — the
history rewrite alone doesn't un-leak something that may already be cloned
elsewhere or cached by a CI system.

## Rule 2: separate "config" from "secret" operationally

Non-secret config (log level, feature flags, timeouts) can reasonably live
in a checked-in file per environment. Secrets need a different delivery
path with tighter access control. A simple, effective baseline with no
extra infrastructure:

```bash
sudo mkdir -p /etc/myapp
sudo touch /etc/myapp/secrets.env
sudo chown deploy:deploy /etc/myapp/secrets.env
sudo chmod 600 /etc/myapp/secrets.env    # only the deploy user can read it, not group/other
```

```ini
# /etc/myapp/secrets.env — never in git, deployed out-of-band (scp over SSH, config management)
DATABASE_PASSWORD=s3cr3t-value
STRIPE_API_KEY=sk_live_xxx
```

```ini
[Service]
User=deploy
EnvironmentFile=/etc/myapp/config.env      # checked into git, non-secret
EnvironmentFile=/etc/myapp/secrets.env     # not in git, mode 600, deploy-owned
```

`chmod 600` plus running the service as the same non-root user that owns
the file means even other unprivileged users on the box can't read it —
and if the app itself is compromised, the blast radius is "this app's
secrets," not "everything on the machine" (assuming other apps run as
different users with their own 600 files).

## Rotating a leaked or routinely-rotated secret

```bash
# 1. Generate/obtain the new secret from the provider (DB, Stripe, etc.)
# 2. Update the secrets file
sudo sed -i 's/^DATABASE_PASSWORD=.*/DATABASE_PASSWORD=new-value-here/' /etc/myapp/secrets.env
# 3. Restart the service to pick up the new value (env vars are read at process start)
sudo systemctl restart myapp
# 4. Confirm the app is healthy, THEN revoke/expire the old secret at the source
curl -sf http://127.0.0.1:3000/health
```

Order matters: update-and-restart-and-verify *before* revoking the old
value at the source, so a mistake in step 2/3 doesn't lock you out of a
now-broken app with no working credential to fall back to during rollback.

## Using a dedicated secrets manager (when file-based isn't enough)

File-based secrets (above) are a fine starting point for a small fleet, but
don't scale to: auditing who accessed which secret when, automatic
rotation, or dynamically-generated short-lived credentials. Once you need
those, tools like **HashiCorp Vault** or a cloud provider's secret manager
(AWS Secrets Manager, GCP Secret Manager) become worth the operational
overhead. The pattern doesn't change conceptually — the app still gets
secrets injected at startup or fetched at runtime — but the source of
truth moves from a file on disk to an API call, gated by a short-lived
auth token instead of filesystem permissions.

```bash
# conceptual shape of a Vault-backed fetch in a startup wrapper script
export DATABASE_PASSWORD=$(vault kv get -field=password secret/myapp/database)
exec /opt/myapp/bin/server
```

## Auditing what's currently exposed

```bash
# check for accidentally-world-readable secret files across the system
sudo find /etc /opt -name "*.env" -o -name "secrets*" 2>/dev/null | xargs -I{} sh -c 'ls -la {}'

# check process environment isn't leaking secrets into logs
sudo systemctl show myapp -p Environment    # shows literal values — treat this output itself as sensitive
```

Never `echo` or log full environment variables from application code —
"debug logging" that includes `process.env` or `os.environ` is one of the
most common accidental-leak vectors, especially once logs are shipped to a
third-party aggregator (module 9).

## Worked example: locking down a secrets file end to end

```bash
sudo useradd -r -s /usr/sbin/nologin deploy || true
sudo mkdir -p /etc/myapp
printf 'DATABASE_PASSWORD=changeme123\n' | sudo tee /etc/myapp/secrets.env > /dev/null
sudo chown deploy:deploy /etc/myapp/secrets.env
sudo chmod 600 /etc/myapp/secrets.env
sudo -u deploy cat /etc/myapp/secrets.env    # works — same user
sudo -u nobody cat /etc/myapp/secrets.env    # Permission denied — confirms isolation
```

## How It Actually Works

**Why secrets in environment variables are visible to more than you'd
think.** An environment variable set on a process is readable by anything
with access to `/proc/<pid>/environ` on that host (typically root, or the
same UID) and is included verbatim in crash dumps, some logging middleware
that dumps `process.env`, and child-process environments unless explicitly
stripped. It is *not* automatically exposed over the network or to other
users, but it is far more exposed than a value fetched at runtime from a
secrets manager and held only in application memory, never touching disk,
shell history, or a source-controlled `.env` file.

**How a secrets manager (Vault, AWS Secrets Manager) actually restricts
access.** These systems store secrets encrypted at rest and only decrypt
them when a request presents valid, scoped credentials (an IAM role, a
Vault token tied to a policy) — access is enforced by the secrets manager's
own authorization check on every read, and every read is logged centrally.
This is qualitatively different from a file on disk with restrictive
permissions: file permissions only stop *other users on that host*; a
secrets manager can also grant/revoke/rotate an individual application's
access without touching any other consumer of the same secret, and produces
a full audit trail of who fetched what and when.

**Why rotating a secret is hard in practice.** Rotation isn't just
"generate a new value" — every consumer holding the old value in memory or a
connection pool needs to pick up the new one before the old one is
invalidated, or you get a brief window of failed auth. This is why
production secrets rotation is usually staged: both old and new credentials
are valid simultaneously for an overlap window, consumers are bounced (or
poll for updates) to pick up the new value, and only then is the old value
revoked — a mechanism directly analogous to blue-green deployment, applied
to credentials instead of application code.

## Exercise

1. Create a `secrets.env` file with mode `600` owned by a dedicated
   non-root user, and confirm another unprivileged user on the same box
   cannot read it.
2. Wire it into a systemd unit via `EnvironmentFile=` alongside a separate,
   git-tracked `config.env` for non-secret settings.
3. Simulate a rotation: change the value, restart the service, confirm via
   a health/debug endpoint that the new value is in effect, and only then
   "revoke" the old one (e.g. rename it) to prove the app no longer needs
   it.
