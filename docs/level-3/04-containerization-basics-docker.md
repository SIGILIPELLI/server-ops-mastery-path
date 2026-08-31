# 04 · Containerization Basics (Docker) as a Deployment Pattern

Everything up to this point deployed an app directly onto a server's OS —
install a runtime, copy code, run it under systemd. Containers package the
app *and* its runtime/dependencies into one portable image, so "works on my
machine" becomes "works in this exact environment, everywhere." This module
treats Docker as a deployment pattern, not a general Docker course.

## Why containers, from an ops angle

- **Reproducible environment** — the container image pins the OS packages,
  language runtime version, and dependencies. No more "prod has a different
  libssl than staging."
- **Immutable artifact** — you build an image once, tag it, and run the
  *same* bytes in staging and production (this is the "environment-based
  deployment" idea from Level 2, taken further — the artifact itself is
  now identical, not just the config pointed at it).
- **Density and isolation** — multiple containers share one kernel but get
  their own filesystem/process namespace, so you can pack more workloads
  per host than full VMs, with lighter-weight isolation than "just run
  everything directly on the host."
- **Portability** — the same image runs on a laptop, a bare-metal server,
  or any cloud, as long as a container runtime is present.

Trade-off to be honest about: containers add a layer (the runtime, image
registry, orchestration) — for a single app on a single server, plain
systemd (Level 1, module 4) is often simpler and has less to operate. Reach
for containers when you need portability across environments or multiple
apps/services per host with clean isolation.

## Anatomy of a Dockerfile

```dockerfile
FROM node:20-slim

WORKDIR /app

# copy dependency manifests first so this layer caches independently of source changes
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

COPY . .

# run as non-root — never run production containers as root
RUN useradd --uid 1000 --create-home appuser
USER appuser

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/healthz || exit 1

CMD ["node", "server.js"]
```

Key ops-relevant choices baked into this file:

- **Layer ordering** — dependency install before source copy means
  changing application code doesn't invalidate (and re-run) the `npm ci`
  layer, keeping rebuilds fast.
- **Slim base image** — smaller attack surface and faster pulls than a full
  `node:20` image with a whole Debian userland.
- **Non-root user** — if the app is compromised, the attacker doesn't
  automatically have root inside the container (and, depending on
  configuration, on the host).
- **`HEALTHCHECK`** — the same liveness/readiness idea from module 01,
  expressed at the container level so `docker ps` and orchestrators can see
  container health directly.

## Building, tagging, and running

```bash
docker build -t registry.example.com/app:1.4.2 .
docker push registry.example.com/app:1.4.2

docker run -d \
  --name app \
  --restart unless-stopped \
  -p 3000:3000 \
  -e DATABASE_URL="postgres://app:secret@db.internal:5432/app" \
  --memory=512m --cpus=1 \
  registry.example.com/app:1.4.2
```

Notes that matter operationally:

- **Tag by immutable version, never just `latest`** — `latest` makes
  rollbacks and "what's actually running in prod" impossible to answer
  reliably. Use a semantic version or the git commit SHA as the tag.
- `--restart unless-stopped` gives you the systemd-equivalent of
  auto-restart on crash/reboot, without systemd itself.
- `--memory`/`--cpus` set resource limits — without them, one runaway
  container can starve every other container on the host, the container
  equivalent of one process filling up a whole VM.

## docker-compose for multi-container apps

```yaml
# docker-compose.yml
services:
  app:
    image: registry.example.com/app:1.4.2
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/app
    depends_on:
      db:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"

  db:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  db_data:
```

```bash
docker compose up -d
docker compose ps
docker compose logs -f app
```

`depends_on: condition: service_healthy` ties back directly to the
`HEALTHCHECK` mechanism — `app` won't start until `db`'s healthcheck passes,
avoiding the classic "app crash-loops because the database wasn't ready
yet" race.

## Persisting data and avoiding the classic mistake

Containers are meant to be disposable — anything written inside the
container's own filesystem is lost when the container is removed. Data that
must survive belongs in a **named volume** (as `db_data` above) or a
bind-mounted host path, never in the container's writable layer.

```bash
# WRONG: data lives only inside the container, gone on `docker rm`
docker run -d postgres:16

# RIGHT: data lives in a named volume, survives container replacement
docker run -d -v db_data:/var/lib/postgresql/data postgres:16
```

This is the container-world version of "don't put your database on the same
disk you'll wipe during a redeploy" — the container itself is the
disposable unit; durable state has to be explicitly placed outside it.

## Logs and systemd-style operational habits

```bash
docker logs -f --tail 100 app          # equivalent to journalctl -u app -f
docker inspect app --format '{{.State.Health.Status}}'   # healthy / unhealthy / starting
docker stats --no-stream               # equivalent to a quick top for containers
```

In production, container logs are typically shipped off-host (Level 2's
centralized logging module) rather than read locally — a container that's
rescheduled onto a different host takes its logs with it unless they're
already centralized.

## Exercise

1. Write a `Dockerfile` for a small app (any language) that follows the
   pattern above: cached dependency layer, non-root user, `HEALTHCHECK`.
2. Write a `docker-compose.yml` running the app plus a database it depends
   on, using `depends_on: condition: service_healthy` and a named volume
   for the database's data directory.
3. `docker compose up -d`, then kill the database container (`docker kill
   <db-container>`) and observe: does the app container crash-loop, retry,
   or hang? Decide whether that's the behavior you'd want in production and
   what you'd change (e.g. retry logic in the app, or `restart:
   unless-stopped` on the db) if not.
4. Tag and rebuild the image with a code change, and demonstrate a rollback
   by re-running the previous version's tag — confirm the previous tag
   still exists and works, which is why "never overwrite `latest`" matters.
