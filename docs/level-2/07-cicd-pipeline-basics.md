# 07 · CI/CD Pipeline Basics

CI (continuous integration) means every code change is automatically
built and tested. CD (continuous delivery/deployment) extends that to
automatically packaging and shipping the result. This module builds a real
pipeline with GitHub Actions that ties together everything from modules
1-6: build once, test it, deploy the same artifact through staging then
production.

## Anatomy of a pipeline

```
push to git
   │
   ▼
CI: install deps → lint → run tests → build artifact
   │ (only if all green)
   ▼
CD: deploy artifact to staging → smoke test
   │ (only if smoke test passes, and/or manual approval)
   ▼
CD: deploy same artifact to production → smoke test
```

Each stage only runs if the previous stage succeeded — that ordering is
the entire point. A pipeline that deploys regardless of test results isn't
CI/CD, it's just automation of the risky part.

## A GitHub Actions workflow, end to end

```yaml
# .github/workflows/deploy.yml
name: build-test-deploy

on:
  push:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build
      - name: Package artifact
        run: tar czf release.tar.gz dist/ package.json package-lock.json
      - uses: actions/upload-artifact@v4
        with:
          name: release
          path: release.tar.gz

  deploy-staging:
    needs: build-and-test
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: release
      - name: Deploy to staging
        env:
          SSH_KEY: ${{ secrets.STAGING_SSH_KEY }}
        run: |
          echo "$SSH_KEY" > deploy_key && chmod 600 deploy_key
          scp -i deploy_key -o StrictHostKeyChecking=no release.tar.gz deploy@staging.internal:/opt/releases/
          ssh -i deploy_key -o StrictHostKeyChecking=no deploy@staging.internal "/opt/myapp/bin/install-release.sh release.tar.gz"
      - name: Smoke test staging
        run: |
          sleep 5
          curl -sf https://staging.example.com/health

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production   # GitHub Environments can require manual approval here
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: release
      - name: Deploy to production
        env:
          SSH_KEY: ${{ secrets.PROD_SSH_KEY }}
        run: |
          echo "$SSH_KEY" > deploy_key && chmod 600 deploy_key
          scp -i deploy_key -o StrictHostKeyChecking=no release.tar.gz deploy@prod1.internal:/opt/releases/
          ssh -i deploy_key -o StrictHostKeyChecking=no deploy@prod1.internal "/opt/myapp/bin/install-release.sh release.tar.gz"
      - name: Smoke test production
        run: curl -sf https://example.com/health
```

Key design points:

- `needs: build-and-test` / `needs: deploy-staging` — each job is gated on
  the previous one succeeding. If tests fail, `deploy-staging` never runs.
- The **same artifact** (`upload-artifact`/`download-artifact`) flows
  through both deploy jobs — it's built exactly once, matching module 4's
  "build once, promote" principle.
- `environment: production` in GitHub Actions can be configured (in repo
  settings → Environments) to require a human reviewer to click "approve"
  before that job runs — a manual gate on the highest-stakes step, same
  idea as the `read -p` confirmation from module 4's script, but enforced
  by the platform instead of a shell prompt.
- Secrets (`STAGING_SSH_KEY`, `PROD_SSH_KEY`) live in GitHub's encrypted
  secrets store, scoped per-environment — the production key is never
  exposed to a staging-only job.

## Fail fast: lint and unit tests before anything expensive

Order steps cheapest-and-fastest-to-fail first. A lint error should fail in
seconds, not after a 10-minute build. This also holds true within a single
job — `npm run lint` before `npm test` before `npm run build`.

## Rollback: keep the last N artifacts around

```bash
# on the deploy target, install-release.sh keeps the previous release for fast rollback
#!/usr/bin/env bash
set -euo pipefail
RELEASE="$1"
mkdir -p /opt/myapp/releases
tar xzf "/opt/releases/${RELEASE}" -C "/opt/myapp/releases/${RELEASE%.tar.gz}"
ln -sfn "/opt/myapp/releases/${RELEASE%.tar.gz}" /opt/myapp/current
sudo systemctl restart myapp
# keep only the 5 most recent releases on disk
cd /opt/myapp/releases && ls -t | tail -n +6 | xargs -r rm -rf
```

```bash
# rollback = just repoint the symlink to a previous release directory and restart
ln -sfn /opt/myapp/releases/release-abc1234 /opt/myapp/current
sudo systemctl restart myapp
```

This symlink-swap pattern is what makes rollback fast (seconds, not a full
rebuild-and-redeploy) — it's the same idea CI/CD pipelines rely on when a
production smoke test fails right after deploy.

## Exercise

1. Set up a GitHub Actions workflow for a small repo that runs lint + tests
   on every push, and only proceeds to a "deploy" job (even a fake one that
   just echoes) if both pass.
2. Add a second, dependent job (`needs:`) that only runs after the first
   succeeds, simulating the staging→production promotion.
3. Break a test on purpose, push, and confirm in the Actions UI that the
   deploy job is skipped rather than run.
