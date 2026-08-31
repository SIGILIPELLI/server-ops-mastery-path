# 04 · Security Patching Strategy at Scale

Level 1's hardening checklist covered `unattended-upgrades` on one server.
At fleet scale — dozens or hundreds of hosts, multiple services, staged
environments — patching needs a deliberate strategy: how urgently different
classes of vulnerabilities get patched, how patches are rolled out without
causing an outage, and how you prove the fleet is actually patched rather
than assuming it.

## Classify before you patch: not every CVE is equally urgent

```
CVSS score + exploitability + exposure = actual urgency

Critical (CVSS 9.0-10.0) + known exploited in the wild + internet-facing
  → patch within hours, out-of-band, don't wait for the next scheduled window

High (CVSS 7.0-8.9) + internet-facing
  → patch within days, next scheduled window if one is imminent

Medium/Low, or internal-only service with no public exposure
  → normal patch cadence (e.g. weekly/monthly maintenance window)
```

A CVE's raw CVSS score without context about *exposure* leads to both
over-reaction (dropping everything for a critical bug in a library you
don't even use in an exposed path) and under-reaction (treating a
"medium" bug as low-priority when it's in your public-facing login
endpoint). Maintain an actual inventory of what's exposed where (module
"Compliance & Audit Considerations" builds on this) so triage is based on
real exposure, not just the headline score.

## Patch tiers and rollout cadence

```
Tier 1 — OS/kernel security patches:      weekly automated (unattended-upgrades,
                                            Level 1 module 2), reboot via
                                            kured/maintenance window if kernel
Tier 2 — Language runtime / framework:    monthly, tested in staging first
                                            (npm/pip/gem security advisories)
Tier 3 — Application dependencies:        continuous via automated PRs
                                            (Dependabot/Renovate) + CI
Tier 4 — Critical/actively-exploited:     out-of-band, immediate, any tier
```

```yaml
# Renovate/Dependabot-style config: auto-PR security patches, group by risk
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule: { interval: "daily" }
    open-pull-requests-limit: 10
    groups:
      security-patches:
        applies-to: security-updates
        update-types: ["patch", "minor"]
```

Automating the *proposal* of patches (a bot opens the PR) while keeping a
human/CI gate on merging is the practical middle ground between "patch
everything instantly, unreviewed" (risk of a bad patch breaking prod) and
"patch manually when someone remembers" (the far more common actual
failure mode).

## Rolling patches across a fleet without an outage

Patching N hosts by SSHing into each one by hand doesn't scale and risks
patching them all simultaneously — if the patch itself breaks something,
you've just taken down 100% of capacity at once. Use the same
rolling/canary discipline as a deploy (Level 2 module 5):

```yaml
# Ansible playbook: patch and reboot one batch at a time, health-check between batches
- name: rolling OS patch
  hosts: web
  serial: "25%"          # patch a quarter of the fleet at a time
  max_fail_percentage: 0  # stop the whole rollout if any host in a batch fails
  tasks:
    - name: apply security updates
      apt:
        upgrade: safe
        update_cache: true

    - name: check if reboot required
      stat: { path: /var/run/reboot-required }
      register: reboot_required

    - name: remove this host from the load balancer pool
      when: reboot_required.stat.exists
      uri:
        url: "http://lb-admin.internal/api/pool/{{ inventory_hostname }}/drain"
        method: POST

    - name: reboot if needed
      when: reboot_required.stat.exists
      reboot:
        reboot_timeout: 300

    - name: wait for health check to pass before re-adding to pool
      when: reboot_required.stat.exists
      uri:
        url: "http://{{ inventory_hostname }}/healthz"
      register: health
      until: health.status == 200
      retries: 10
      delay: 10

    - name: re-add to the load balancer pool
      when: reboot_required.stat.exists
      uri:
        url: "http://lb-admin.internal/api/pool/{{ inventory_hostname }}/undrain"
        method: POST
```

`serial: "25%"` combined with `max_fail_percentage: 0` is the key
safety property: if the patch itself causes a problem (a kernel update that
breaks a driver, a library update with a breaking change), it surfaces on
the first batch and the rollout halts automatically before it reaches the
whole fleet — this is the patching equivalent of a canary deploy.

## Staging first, for anything above Tier 1

```
1. Apply the patch in staging (same OS/package versions as prod)
2. Run the full test suite / a smoke test against staging
3. Apply to a canary batch in prod (module "rolling" above), watch metrics
   for an agreed soak period (e.g. 30 min)
4. Roll out to the rest of the fleet in batches
```

Skipping staging is sometimes justified for a Tier 4 (actively exploited,
critical) patch where the risk of *not* patching immediately outweighs the
risk of an untested patch — but that's a deliberate, documented exception,
not the default path.

## Proving the fleet is patched: don't trust, verify

A patching *process* existing doesn't mean it's actually keeping the fleet
current — hosts get missed (new hosts added outside the automation's
inventory, a host with `unattended-upgrades` silently broken, a host
excluded during an incident and never re-included). Verify with an
independent scan, not just by trusting the automation ran:

```bash
# osquery: ask every host what it actually has installed, fleet-wide
# (via Fleet/osquery-manager, or ad-hoc via Ansible for a smaller fleet)
ansible web -i inventory.ini -m shell \
  -a "dpkg-query -W -f='${Package} ${Version}\n' openssl"
```

```bash
# Trivy: scan a container image or filesystem for known CVEs directly
trivy image registry.example.com/app:1.4.2 --severity HIGH,CRITICAL
```

```yaml
# CI gate: fail the build if the image has unpatched critical CVEs
- name: scan image for critical vulnerabilities
  run: trivy image --exit-code 1 --severity CRITICAL registry.example.com/app:${{ github.sha }}
```

Running a vulnerability scanner as a **CI gate** (fails the build/deploy on
a critical finding) rather than only a periodic report is what turns
"we scan for vulnerabilities" into "we can't ship a known-critical
vulnerability" — a report nobody's required to act on tends to accumulate
unread findings.

## Tracking patch compliance as a metric

```
# Fleet-wide dashboard metric, reviewed weekly (or as a compliance report, module 07)
% of fleet with all Tier-1 patches applied within SLA (e.g. 7 days of release)
% of fleet with any actively-exploited CVE present
mean time to patch, by tier, over the last quarter
```

Treating "patch compliance %" as a tracked, reviewed metric — the same way
error rate or latency is tracked (Level 3 module 09) — is what keeps
patching from silently regressing after the initial rollout of a patching
process; without a metric, "we patch regularly" is unfalsifiable until an
audit or an incident proves otherwise.

## Exercise

1. Write a triage rule (like the classification block above) for your own
   environment: what counts as Tier 1-4, and what's the target time-to-
   patch for each.
2. Write an Ansible playbook (or equivalent for your config-management
   tool) that rolls out an OS patch to a fleet in batches, drains each host
   from a load balancer pool before rebooting it, and confirms a health
   check passes before re-adding it and moving to the next batch.
3. Run Trivy (or an equivalent scanner) against a real container image you
   use, and classify the findings using your Tier system from step 1.
4. Design one dashboard panel or scheduled report that shows patch
   compliance % across your fleet, and state what threshold would trigger
   an alert or escalation.
