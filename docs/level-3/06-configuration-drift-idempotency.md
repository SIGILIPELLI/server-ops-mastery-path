# 06 · Configuration Drift & Idempotency

Module 05 introduced IaC's promise: describe desired state, apply it, done.
That promise only holds if two properties are true — the tool's runs are
**idempotent** (safe to repeat) and you actually detect when reality has
**drifted** away from what's declared. This module is about both.

## What is drift

Drift is any difference between what your IaC/config-management code says
should exist and what's actually running. It happens constantly, in
ordinary operation:

- Someone SSHes in during an incident and hand-edits `nginx.conf` to stop
  the bleeding, then forgets to port the fix back into the Ansible
  playbook/Terraform file.
- A cloud console click (someone opens the AWS console and bumps an
  instance's size) bypasses Terraform entirely.
- An auto-scaling event or a manually-run one-off script changes something
  the IaC tool doesn't know about.
- A package gets security-patched by unattended-upgrades (Level 1 module
  2) to a version different from what's pinned in the playbook.

Left unnoticed, drift means your IaC code is no longer a truthful
description of your infrastructure — the DR promise from module 03/05
("rebuild from these files") quietly stops being true, and nobody finds out
until the rebuild is attempted during an actual disaster.

## Detecting drift

```bash
# Terraform: compare real infrastructure against declared state, without changing anything
terraform plan
```

A clean `plan` with "No changes" means no drift. Any unexpected `~ update
in-place` or `- destroy`/`+ create` line for something nobody touched via
Terraform is drift — investigate before deciding whether to `apply` (pull
reality back to match code) or update the code (accept the manual change
as the new desired state).

```bash
# Ansible: --check runs in dry-run mode, --diff shows what would change
ansible-playbook -i inventory.ini site.yml --check --diff
```

```bash
# ad-hoc drift check on package versions across a fleet
ansible web -i inventory.ini -m shell -a "dpkg -l nginx | tail -1"
```

Neither tool watches continuously by default — drift detection means
*running the check regularly* (a scheduled CI job hitting `terraform plan`
nightly, or `ansible-playbook --check` on a cron), not a one-time thing.

```yaml
# .github/workflows/drift-check.yml (conceptual — CI job, not applied automatically)
name: nightly-drift-check
on:
  schedule:
    - cron: "0 6 * * *"
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: terraform init
      - run: terraform plan -detailed-exitcode || echo "DRIFT DETECTED" # exit code 2 means changes pending
```

## Idempotency: why it matters more than it sounds like it should

An operation is **idempotent** if running it once has the same effect as
running it many times. This is the property that makes "just re-run the
playbook" a safe default action instead of a gamble.

```bash
# NOT idempotent: appends a line every single run
echo "127.0.0.1 app.local" >> /etc/hosts

# Idempotent: only adds the line if it's not already there
grep -qxF "127.0.0.1 app.local" /etc/hosts || echo "127.0.0.1 app.local" >> /etc/hosts
```

```bash
# NOT idempotent: fails the second time (directory already exists)
mkdir /opt/app/releases

# Idempotent
mkdir -p /opt/app/releases
```

Ansible modules are (mostly) idempotent by design — `apt: {name: nginx,
state: present}` checks whether nginx is already installed before doing
anything, and reports `changed: false` if it is. This is exactly what
"convergent" means: no matter the starting state, applying the same
playbook repeatedly converges on the same end state, and stops making
changes once it's reached.

```yaml
# Idempotent Ansible task: only "changed" the first time it actually installs
- name: nginx present
  apt:
    name: nginx
    state: present
```

```yaml
# NON-idempotent trap: a raw shell command that always reports "changed"
# and may fail or duplicate work on re-run
- name: add cron entry
  shell: "echo '0 2 * * * /usr/local/bin/backup.sh' >> /etc/crontab"
  # BAD — appends a new line every run

# Fixed with the dedicated, idempotent module
- name: add cron entry
  cron:
    name: "nightly backup"
    minute: "0"
    hour: "2"
    job: "/usr/local/bin/backup.sh"
```

The `cron` module tracks the entry by name and only touches it if the
schedule/command actually changed — the `shell` version has no memory of
what it did last time, so it just keeps acting.

## Convergence vs. one-shot scripts

A one-shot deployment script (Level 1 module 6) runs once per deploy and is
fine to be somewhat imperative — its job is over quickly, and if it fails
partway you re-run the whole deploy. Config-management code is different:
it's meant to be run *repeatedly, forever*, against machines whose state
you don't fully control between runs (manual fixes, drift, partial
failures). That's why idempotency is a hard requirement there, not a nice
extra — the tool must be safe to converge from any starting point, not just
"empty machine."

## Worked example: turning a non-idempotent script into an idempotent Ansible task

Starting point — a shell script someone wrote to set up a deploy user:

```bash
#!/usr/bin/env bash
useradd deploy
mkdir /home/deploy/.ssh
echo "ssh-ed25519 AAAA... deploy-key" >> /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
```

Run this twice: `useradd` errors on the second run (user exists), the
`authorized_keys` line gets duplicated, and the whole script aborts on the
first error before `chown` even reruns. This is exactly the "not safe to
re-run" problem.

Idempotent Ansible equivalent:

```yaml
- name: deploy user exists
  user:
    name: deploy
    shell: /bin/bash
    create_home: true

- name: .ssh directory exists with correct permissions
  file:
    path: /home/deploy/.ssh
    state: directory
    owner: deploy
    group: deploy
    mode: "0700"

- name: deploy key present in authorized_keys
  authorized_key:
    user: deploy
    key: "ssh-ed25519 AAAA... deploy-key"
    state: present
```

Every one of these modules checks current state before acting and reports
`changed: false` on a second run against an already-converged host —
`ansible-playbook site.yml` becomes something you can run nightly via cron
as a drift-correction job, not just a one-time setup script.

## Exercise

1. Take a shell script you (or the examples above) wrote that isn't
   idempotent, and rewrite it as an Ansible playbook using proper modules
   (`user`, `file`, `authorized_key`, `cron`, `apt`, etc.) instead of `shell`
   /`command` wherever a dedicated module exists.
2. Run the playbook twice against a scratch VM and confirm `changed=0` on
   the second run.
3. Manually introduce drift on the VM (e.g. `sudo systemctl stop nginx`,
   or hand-edit a config file the playbook manages) and run
   `ansible-playbook --check --diff` to see the drift reported without
   changing anything, then run it for real and confirm it's corrected.
4. Set up a nightly cron job (or CI schedule) that runs the drift check and
   emails/logs a warning when it reports pending changes.
