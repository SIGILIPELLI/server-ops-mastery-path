# 09 · Platform Engineering & Internal Developer Experience

Everything in this program so far has been "how does an ops person build
and operate reliable systems." Platform engineering flips the lens: how
does an ops/platform team build **tools and paved paths** so that other
engineers can safely do parts of that work themselves, without needing to
re-learn every module in this program from scratch for every new service.

## The problem platform engineering solves

Without a platform, every team reinvents deployment, monitoring, and
provisioning independently — inconsistently, with varying quality, and
with the platform/ops team as a manual bottleneck for anything
infrastructure-related ("file a ticket to get a new service deployed").
This doesn't scale past a handful of teams, and it produces exactly the
kind of configuration drift (Level 3 module 06) and inconsistent security
posture (Level 4 module 04) that centralizing on a shared, well-built
platform is meant to prevent.

```
Without a platform:                  With a platform:
Team A hand-rolls their deploy      All teams use the same "golden path"
  script and monitoring              template, get monitoring/logging/
Team B copies Team A's script,       alerting for free, and provisioning
  slightly different, drifts         is self-service, not a ticket queue
Team C does something else
  entirely, nobody remembers why
```

## The "golden path": paved, not mandated

A golden path is a supported, pre-built way of doing something common
(deploying a new service, provisioning a database, setting up CI) that's
deliberately made the *easiest* option — teams choose it because it's less
work than doing it themselves, not because they're forced to.

```yaml
# Example: a service scaffolding template (cookiecutter/Backstage software template)
# `platform new-service orders-v2` generates:
orders-v2/
├── Dockerfile                 # already follows Level 3 module 4's best practices
├── .github/workflows/ci.yml   # test, build, scan (Level 4 module 4's Trivy gate), deploy
├── terraform/                 # provisions the service's infra, follows org tagging (module 6)
├── k8s/deployment.yaml        # includes readiness/liveness probes (Level 3 module 1) by default
└── docs/README.md             # runbook template pre-filled (Level 4 module 3)
```

```bash
$ platform new-service orders-v2
✓ Repository created: github.com/company/orders-v2
✓ CI pipeline configured (test → build → scan → deploy)
✓ Terraform module scaffolded (VPC, LB, autoscaling group pre-wired)
✓ Dashboards created (RED metrics, pre-wired to your service name)
✓ On-call rotation template ready to configure in PagerDuty
→ Next: fill in your business logic. Everything else already works.
```

A team running `platform new-service` starts with correct health checks,
CI/CD, monitoring, and security scanning already wired up — the golden
path's value is that doing the *right* thing (from the perspective of every
earlier module in this program) requires less effort than doing an ad-hoc
thing, so it wins by default rather than by mandate.

## Self-service, with guardrails

The platform team's job shifts from "provision things on request" to
"build the self-service tooling and the guardrails that make self-service
safe":

```hcl
# Terraform module exposed to application teams — they can provision their
# own database, but only within the constraints the platform team has set
module "app_database" {
  source        = "git::https://github.com/company/platform-modules//postgres"
  instance_size = "db.t3.medium"    # allowed values enforced by the module's variable validation
  environment   = "production"
  # backup retention, encryption, tagging, network placement are ALL fixed
  # by the module itself — not exposed as options the calling team can get wrong
}
```

```hcl
variable "instance_size" {
  type = string
  validation {
    condition     = contains(["db.t3.small", "db.t3.medium", "db.t3.large"], var.instance_size)
    error_message = "instance_size must be one of the approved sizes."
  }
}
```

This is the platform-engineering expression of "least privilege meets
convenience": an application team can provision a database in minutes
without filing a ticket, but cannot accidentally provision it unencrypted,
untagged, or outside the approved size range, because the module simply
doesn't expose those failure modes as options.

## Internal developer platform (IDP) components

```
A typical IDP is built from several layers, each mapping to earlier modules:

1. Service catalog (e.g. Backstage) — a directory of every service, its
   owner, its docs, its on-call rotation, its dependencies
2. Golden-path templates — scaffolding, as above
3. Self-service infra provisioning — Terraform modules with guardrails, as above
4. CI/CD platform — shared pipeline definitions (Level 2 module 7) that
   every service inherits, rather than each team writing their own from scratch
5. Observability platform — a shared Prometheus/Grafana/tracing stack
   (Level 3 module 9) that every service is automatically onboarded onto
6. Golden signals dashboards, generated automatically per-service from
   the RED metrics every service exposes via the golden path template
```

## Measuring developer experience, not just infrastructure health

A platform team's success metric isn't "the platform exists," it's
"engineers on other teams can ship changes quickly and safely using it."
Standard measures (the DORA metrics, widely used as a baseline):

```
Deployment frequency:      how often does a team ship to production
Lead time for changes:     time from commit to running in production
Change failure rate:       % of deploys that cause a rollback/incident
Time to restore service:   MTTR when something does go wrong (Level 4 module 3)
```

```
# Practical signal, gathered directly rather than assumed:
Quarterly platform survey: "How easy was it to provision a new service /
                            deploy a change / debug a production issue
                            using the platform, on a 1-5 scale?"
Support ticket volume for platform-related requests, trending over time
  (rising volume for "how do I..." questions signals the golden path
   isn't self-explanatory enough, or documentation has gaps)
```

A platform that's technically excellent but that engineers route around
(because it's confusing, slow to iterate with, or missing an escape hatch
for legitimate edge cases) has failed its actual purpose — DevEx metrics
and direct feedback are how you'd know that's happening before it shows up
as teams quietly building their own parallel, undocumented infrastructure
again.

## Escape hatches: golden paths must not become golden cages

A golden path that's mandatory with no way to deviate for a legitimate
edge case pushes teams to either misuse the platform (bending a template
to do something it wasn't designed for) or abandon it entirely. Good
platform design includes an explicit, supported way to step outside the
default when justified:

```
Golden path covers: 95% of services (standard web app shape)
Escape hatch: a documented process to provision custom infrastructure
              directly via Terraform (not through the scaffolded module),
              with platform team review to ensure it still meets baseline
              requirements (tagging, security scanning, monitoring) even
              though it isn't using the template
```

The review step matters: an escape hatch that skips the golden path
*and* skips the baseline requirements it was enforcing (tagging, security
scanning) reintroduces exactly the inconsistency platform engineering was
built to eliminate.

## How It Actually Works

**Why Terraform's `validation` block enforces guardrails at plan time, not
review time.** A `variable` block's `validation { condition ... }` is
evaluated by Terraform itself during `terraform plan`/`apply`, before any
API call to the cloud provider is made — an invalid `instance_size` fails
immediately with the custom `error_message`, with no dependency on a
human reviewer noticing an out-of-range value in a PR diff. This is a
categorically stronger guarantee than a code-review checklist: a review
step can be skipped under time pressure or simply missed, while a
`validation` block runs deterministically on every single invocation,
which is why platform guardrails are built into the module's own schema
rather than documented as a convention teams are expected to follow.

**Why a golden path wins adoption through friction differential, not
mandate.** Teams don't choose infrastructure patterns by evaluating which
is "correct" in the abstract — they choose whichever path requires less
effort to ship the feature they actually care about. A scaffolded service
that already has CI, health checks, and monitoring wired up means the
*marginal* effort of doing the right thing is near zero, while the
ad-hoc alternative requires reproducing all of that from scratch. This
is the same behavioral mechanism behind Dependabot-style automated PRs
(Level 4 module 04) or a default-secure Terraform module here — making the
correct choice cheaper than the incorrect one changes behavior far more
reliably than policy alone, because it doesn't depend on anyone
remembering or enforcing the policy.

**Why DORA metrics specifically (not just "is the platform up") measure
platform success.** Deployment frequency and lead time for changes measure
how much friction the platform adds to the *path from commit to
production* — the exact surface a golden path is meant to shrink — while
change failure rate and MTTR measure whether the safety rails (CI gates,
staged rollouts, health checks the template wires up by default) actually
reduce the blast radius and detection time when something does go wrong.
A platform can have perfect uptime for its own control-plane components
while still failing its purpose if it makes shipping *application* changes
slower or riskier than going around it — which is why these four numbers,
measured on the teams *using* the platform rather than the platform's own
infrastructure, are the metrics that actually reveal whether it's working.

## Exercise

1. Pick one recurring task engineers on your team do manually today
   (provisioning a database, setting up CI for a new service, creating
   monitoring dashboards) and design a golden-path template or self-service
   module for it, listing what it fixes/enforces by default versus what it
   leaves configurable.
2. Add at least one guardrail (like the Terraform variable validation
   above) that prevents a common misconfiguration, without requiring a
   human reviewer to catch it manually every time.
3. Define 3-4 developer-experience metrics you'd track for this platform
   component (borrowing from DORA or the survey approach above), and state
   what a "the platform is failing its users" result would look like for
   each.
4. Design an escape hatch for a legitimate case your golden path doesn't
   cover, including what review or baseline requirements still apply even
   when the golden path itself is bypassed.
