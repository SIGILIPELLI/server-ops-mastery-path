---
title: "Server Ops Interview Questions and Answers (179 Questions)"
description: "179 Server Ops interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Server Ops Mastery Path."
---

# Server Ops Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Server Ops Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Server Basics &amp; SSH Access](level-1/01-server-basics-ssh-access.md)

**Q1.** Which SSH authentication method is preferred over passwords on servers?

**Answer:** Public-key authentication with a protected private key

Keys resist guessing and can be revoked per user.

**Q2.** What does ssh-copy-id do?

**Answer:** Appends your public key to the remote user's ~/.ssh/authorized_keys

Never share the private key.

**Q3.** What permission bits (octal) do these produce?

```bash
touch f; chmod 600 f; stat -c %a f 2>/dev/null || stat -f %Lp f
chmod u+x,g-r f; stat -c %a f 2>/dev/null || stat -f %Lp f
```

**Answer:** 600
700

chmod 600 leaves rw for the owner only; adding u+x gives 700 and removing group read changes nothing. Verified by running it in bash.

**Q4.** Which file must have strict permissions or ssh refuses to use it?

**Answer:** The private key (for example mode 600)

Keep ~/.ssh at 700.

### [02 · Initial Server Hardening Checklist](level-1/02-initial-hardening-checklist.md)

**Q5.** Which sshd setting stops direct root login?

**Answer:** PermitRootLogin no

Use a normal user with sudo.

**Q6.** Why disable password authentication after keys work?

**Answer:** It removes the brute-force password attack surface

Test key login in a second session before closing the first.

**Q7.** What is the first thing to do on a fresh server?

**Answer:** Apply updates, create a non-root admin user and secure SSH

Harden before exposing services.

**Q8.** What does a host firewall such as ufw or nftables do?

**Answer:** Allows only the needed inbound ports and blocks the rest

Default deny.

### [03 · Package Management](level-1/03-package-management.md)

**Q9.** What does apt update do compared with apt upgrade?

**Answer:** update refreshes package lists; upgrade installs newer versions

Run update first.

**Q10.** Why pin or hold critical package versions in production?

**Answer:** To avoid unplanned behaviour changes during automatic upgrades

Upgrade deliberately, tested in staging.

**Q11.** What does a package manager verify when installing?

**Answer:** Signatures of packages from configured repositories

Only add trusted repos.

**Q12.** What is the safest way to install software on a server?

**Answer:** From the distribution or a vendor's signed repository, not curl | sh from unknown sources

Verify what you run as root.

### [04 · Process Management (systemd)](level-1/04-process-management-systemd.md)

**Q13.** What does systemctl enable myapp do?

**Answer:** Configures the service to start at boot

Use --now to also start it.

**Q14.** What does Restart=on-failure in a unit file do?

**Answer:** Restarts the service if it exits with an error

Add RestartSec to avoid tight loops.

**Q15.** Which commands show why a service failed?

**Answer:** systemctl status myapp and journalctl -u myapp

Logs live in the journal.

**Q16.** Why run a service as a dedicated non-root user?

**Answer:** Limits damage if the process is compromised

User= in the unit file.

### [05 · Basic Networking Concepts](level-1/05-basic-networking-concepts.md)

**Q17.** What is the difference between TCP and UDP?

**Answer:** TCP is reliable and connection-oriented; UDP is connectionless without delivery guarantees

DNS often uses UDP; HTTP uses TCP.

**Q18.** How many usable host addresses does a /24 IPv4 network have?

```python
print(2**(32-24)-2)
```

**Answer:** 254

256 addresses minus network and broadcast. Verified by running it.

**Q19.** What port does HTTPS use by default?

**Answer:** 443

HTTP is 80, SSH is 22.

**Q20.** What does DNS do?

**Answer:** Translates names to IP addresses

A, AAAA and CNAME records.

### [06 · Writing a Build/Deploy Script](level-1/06-build-deploy-script.md)

**Q21.** What does this deploy-script fragment print?

```bash
set -eu
release=app-v1
for step in build test ship; do echo "step: $step"; done
name=${1:-world}
echo "hello ${name} ${release#app-}"
```

**Answer:** step: build
step: test
step: ship
hello world v1

${1:-world} supplies a default; ${var#prefix} strips a prefix. Verified by running it in bash.

**Q22.** What does set -e do here?

```bash
set -e
echo one
false
echo two
echo three
```

**Answer:** one

set -e exits at the first failing command, so two and three never print. Verified by running it in bash.

**Q23.** Why make deploy scripts idempotent?

**Answer:** Re-running them after a partial failure must leave the system in the same correct state

Check before create; use mkdir -p, ln -sfn.

**Q24.** Why use set -euo pipefail at the top of a script?

**Answer:** Stop on errors, undefined variables and failed pipeline stages instead of continuing silently

Fail fast.

### [07 · Environment Variables &amp; Config Management](level-1/07-environment-variables-config.md)

**Q25.** What does this print?

```bash
export APP_PORT=8080
env_file=$(mktemp)
printf 'DB_HOST=db1\nDB_PORT=5432\n' > $env_file
set -a; . $env_file; set +a
echo "$DB_HOST:$DB_PORT $APP_PORT"
echo "${MISSING:-default}"
```

**Answer:** db1:5432 8080
default

set -a exports sourced variables; ${VAR:-x} supplies a fallback. Verified by running it in bash.

**Q26.** Where should secrets never be stored?

**Answer:** In version control

Use secret stores or protected files.

**Q27.** What is the twelve-factor guidance on config?

**Answer:** Store config in the environment, separate from code

Same artifact for every environment.

**Q28.** Why not print environment variables in logs?

**Answer:** They may contain secrets

Log keys, not values.

### [08 · Log Basics (journalctl, /var/log)](level-1/08-log-basics.md)

**Q29.** Where do systemd services typically log?

**Answer:** To the journal, viewed with journalctl

Some also write to /var/log.

**Q30.** Which command follows a service's logs live?

**Answer:** journalctl -u myapp -f

Like tail -f.

**Q31.** What does log rotation do?

**Answer:** Archives and compresses old logs and starts new ones so disks do not fill

logrotate.

**Q32.** What does this print?

```bash
printf 'INFO ok\nERROR bad\nINFO ok\nERROR worse\n' | grep -c ERROR
printf 'a\nb\na\n' | sort | uniq -c | sort -rn | head -1 | awk '{print $1, $2}'
```

**Answer:** 2
2 a

grep -c counts matching lines; uniq -c prefixes counts. Verified by running it in bash.

### [09 · Cron Jobs &amp; Scheduled Tasks](level-1/09-cron-scheduled-tasks.md)

**Q33.** Which of these five-field schedules means 'at 02:30 every day'? (prints the matching hour:minute fields)

```bash
echo '30 2 * * *' | awk '{printf "minute=%s hour=%s\n", $1, $2}'
```

**Answer:** minute=30 hour=2

The fields are minute, hour, day of month, month, day of week. Verified by running it in bash.

**Q34.** What does */15 in the minute field mean?

**Answer:** Every 15 minutes

Step values.

**Q35.** Why do cron jobs often fail when copied from a terminal?

**Answer:** Cron runs with a minimal environment, so use absolute paths and set variables

Redirect output to a log.

**Q36.** How do you prevent overlapping runs of a long cron job?

**Answer:** Use a lock (flock) so a new run exits if the old one is still running

Avoid pile-ups.

### [10 · Capstone — Server Bootstrap Script](level-1/10-capstone-bootstrap-script.md)

**Q37.** What is the purpose of the bootstrap capstone script?

**Answer:** Harden a fresh server and stand up a web stack repeatably and idempotently

Automation beats hand typing.

**Q38.** Which command is idempotent when run twice?

```bash
d=$(mktemp -d)
mkdir -p $d/a/b
mkdir -p $d/a/b && echo ok1
ln -sfn $d/a $d/cur && ln -sfn $d/a $d/cur && echo ok2
```

**Answer:** ok1
ok2

mkdir -p and ln -sfn can be run repeatedly. Verified by running it in bash.

**Q39.** Why validate SSH config before restarting sshd?

**Answer:** A broken config can lock you out; use sshd -t

Keep a second session open.

**Q40.** What belongs in the script output?

**Answer:** Clear steps and failures, never secrets

Helps debugging.

## Level 2 · Intermediate

### [01 · Reverse Proxies (nginx basics)](level-2/01-reverse-proxies-nginx.md)

**Q41.** What does a reverse proxy do?

**Answer:** Accepts client requests and forwards them to backend servers

nginx in front of an app.

**Q42.** Why put nginx in front of an application server?

**Answer:** TLS termination, static files, buffering and load balancing

Offloads work.

**Q43.** What does proxy_set_header X-Forwarded-For $remote_addr do?

**Answer:** Passes the original client address to the backend

Otherwise the backend sees the proxy's IP.

**Q44.** What command checks an nginx config before reload?

**Answer:** nginx -t

Then reload without dropping connections.

### [02 · TLS/SSL Certificates (Let's Encrypt / certbot)](level-2/02-tls-ssl-certs.md)

**Q45.** What does a TLS certificate prove?

**Answer:** That the server controls the domain, as vouched for by a trusted CA

It authenticates identity, with encryption.

**Q46.** What does certbot do?

**Answer:** Obtains and renews Let's Encrypt certificates

Automate renewal.

**Q47.** Why must you test certificate renewal?

**Answer:** Expired certificates cause outages

certbot renew --dry-run.

**Q48.** How long do Let's Encrypt certificates last?

**Answer:** About 90 days, so renewal must be automatic

Short lifetimes limit exposure.

### [03 · Load Balancing Basics](level-2/03-load-balancing-basics.md)

**Q49.** What is the aim of a load balancer?

**Answer:** Distribute requests across healthy backends

Improves capacity and availability.

**Q50.** Round-robin across 3 backends: which backend index (0-based) serves request number 10 (starting at request 0)?

```python
print(10 % 3)
```

**Answer:** 1

Index = request number modulo backend count. Verified by running it.

**Q51.** What do health checks do?

**Answer:** Remove failing backends from rotation

Active or passive.

**Q52.** What is sticky sessions risk?

**Answer:** Uneven load and lost sessions when a backend dies

Prefer stateless apps or shared session stores.

### [04 · Environment-Based Deployment (staging vs. prod)](level-2/04-environment-based-deployment.md)

**Q53.** What does 'build once, deploy many' mean?

**Answer:** The same artifact is promoted from staging to production; only configuration differs

Avoid 'works in staging' surprises.

**Q54.** Where should environment differences live?

**Answer:** In configuration (environment variables or files), not in code

Same artifact everywhere.

**Q55.** What should staging resemble?

**Answer:** Production in topology and configuration, with safe data

Realism finds problems.

**Q56.** Why tag artifacts with the git SHA?

**Answer:** Traceability: know exactly what is running

Rollback by SHA.

### [05 · Zero-Downtime Deploy Patterns](level-2/05-zero-downtime-deploy.md)

**Q57.** What is a zero-downtime deploy?

**Answer:** Replacing the running version without dropping requests

Start new, check health, switch, drain old.

**Q58.** Why drain connections before stopping the old process?

**Answer:** In-flight requests can finish

Graceful shutdown.

**Q59.** What breaks zero-downtime database changes?

**Answer:** Non-backward-compatible schema changes

Use expand/contract migrations.

**Q60.** What is a rolling deploy?

**Answer:** Updating instances in small batches behind a load balancer

Capacity must cover the batch.

### [06 · Secrets &amp; Configuration Management](level-2/06-secrets-config-management.md)

**Q61.** Where do secrets belong?

**Answer:** In a secrets manager or protected file deployed out of band, never in git

Rotate regularly.

**Q62.** What should you do if a secret leaks to git?

**Answer:** Rotate it immediately; removing the commit is not enough

Assume it is compromised.

**Q63.** What does least privilege mean for secrets?

**Answer:** Each service gets only the credentials it needs

Limits blast radius.

**Q64.** Why rotate secrets?

**Answer:** Limits the window an exposed secret is useful

Automate where possible.

### [07 · CI/CD Pipeline Basics](level-2/07-cicd-pipeline-basics.md)

**Q65.** What is continuous integration?

**Answer:** Merging changes frequently with automated build and tests

Fast feedback.

**Q66.** What does a pipeline stage failing do?

**Answer:** Stops the pipeline so broken builds do not progress

Gates protect production.

**Q67.** Why keep secrets out of pipeline logs?

**Answer:** Logs are widely readable

Use masked secrets.

**Q68.** What should production deploys require?

**Answer:** Passing tests and, for sensitive systems, an approval step

Protect with environments.

### [08 · Basic Monitoring (uptime checks, resource metrics)](level-2/08-basic-monitoring.md)

**Q69.** Which three resources are the classic ones to watch?

**Answer:** CPU, memory and disk (plus network)

Disk filling silently kills servers.

**Q70.** A disk is 400 GB, uses 300 GB and grows 5 GB/day. How many days until it is full?

```python
print((400-300)/5)
```

**Answer:** 20.0

Alert on fill-rate, not just percent used. Verified by running it.

**Q71.** What does an uptime check measure?

**Answer:** Whether the service responds correctly from outside

Run it from a separate location.

**Q72.** Why alert on symptoms rather than every metric?

**Answer:** Fewer false alarms and focus on user impact

Page for real problems.

### [09 · Centralized Logging Basics](level-2/09-centralized-logging-basics.md)

**Q73.** Why centralise logs?

**Answer:** So you can search across hosts and keep them after a server dies

Ship to a central store.

**Q74.** What does a log shipper such as Vector do?

**Answer:** Collects, transforms and forwards logs to a destination

Agent on each host.

**Q75.** What helps correlate a request across services?

**Answer:** A request or correlation ID in every log line

Generate at the edge.

**Q76.** What should logs never contain?

**Answer:** Passwords, tokens and sensitive personal data

Scrub them.

### [10 · Capstone — Blue-Green Deploy Pipeline](level-2/10-capstone-blue-green-deploy.md)

**Q77.** What is blue-green deployment?

**Answer:** Two identical environments; deploy to the idle one, test, then switch traffic

Instant rollback by switching back.

**Q78.** Why health-check the idle environment before switching?

**Answer:** Prevent sending users to a broken release

Smoke tests too.

**Q79.** What is the main cost of blue-green?

**Answer:** Running duplicate capacity

Trade cost for safety.

**Q80.** What makes rollback easy in blue-green?

**Answer:** The previous environment is still intact; switch back

Keep it until confident.

## Level 3 · Advanced

### [01 · High Availability Concepts (redundancy, failover, health checks)](level-3/01-ha-concepts.md)

**Q81.** What is high availability?

**Answer:** Designing so the service survives component failures with little or no downtime

Redundancy plus failover.

**Q82.** What is a single point of failure?

**Answer:** A component whose failure stops the whole service

Remove or replicate it.

**Q83.** Two independent components each 99.9% available in series give what availability?

```python
print(round(0.999*0.999*100,4))
```

**Answer:** 99.8001

Serial availabilities multiply; redundancy in parallel improves it. Verified by running it.

**Q84.** Two independent 99% replicas in parallel give what availability?

```python
print(round((1-0.01*0.01)*100,2))
```

**Answer:** 99.99

Fail only if both fail: 1 - 0.01^2. Verified by running it.

### [02 · DNS &amp; Load Balancer HA Patterns](level-3/02-dns-lb-ha-patterns.md)

**Q85.** What do DNS health-checked records provide?

**Answer:** Failover by removing unhealthy endpoints from answers

TTL affects failover time.

**Q86.** Why is low TTL important for failover?

**Answer:** Clients re-resolve sooner, but it increases query load

Balance.

**Q87.** What is active-passive?

**Answer:** One serves traffic; the standby takes over on failure

Active-active uses both.

**Q88.** What is the risk of split brain?

**Answer:** Two nodes both believe they are primary and diverge

Use quorum or fencing.

### [03 · Backup &amp; Disaster Recovery Strategy](level-3/03-backup-disaster-recovery.md)

**Q89.** What are RPO and RTO?

**Answer:** RPO is acceptable data loss in time; RTO is acceptable time to restore service

Set by the business.

**Q90.** What is the 3-2-1 backup rule?

**Answer:** 3 copies, 2 media types, 1 offsite

Resilience to different failures.

**Q91.** What proves a backup is good?

**Answer:** A successful restore test

Test regularly.

**Q92.** If backups run every 6 hours, what is the worst-case RPO in hours?

```python
print(6)
```

**Answer:** 6

You can lose up to one interval of data. Verified by running it.

### [04 · Containerization Basics (Docker) as a Deployment Pattern](level-3/04-containerization-basics-docker.md)

**Q93.** Why containerise an application?

**Answer:** Package code and dependencies for consistent runs across environments

Same image everywhere.

**Q94.** Why run containers as non-root?

**Answer:** Limits damage if the container is compromised

USER in the Dockerfile.

**Q95.** Why copy dependency manifests before source in a Dockerfile?

**Answer:** Layer caching: dependencies rebuild only when they change

Faster builds.

**Q96.** Where does state live for containers?

**Answer:** Outside them, in volumes or external services

Containers are disposable.

### [05 · Infrastructure as Code Overview](level-3/05-infrastructure-as-code-overview.md)

**Q97.** What is infrastructure as code?

**Answer:** Defining infrastructure in version-controlled files that tools apply

Reviewable and repeatable.

**Q98.** What is a benefit of declarative IaC?

**Answer:** You describe the desired state and the tool computes the changes

Plan before apply.

**Q99.** Why review a plan before applying?

**Answer:** See what will change, preventing surprises

Treat apply as production change.

**Q100.** Where should IaC state be stored?

**Answer:** In a shared, locked, backed-up remote backend

Avoid conflicts.

### [06 · Configuration Drift &amp; Idempotency](level-3/06-configuration-drift-idempotency.md)

**Q101.** What is configuration drift?

**Answer:** Real systems diverging from their declared configuration over time

Detect and reconcile.

**Q102.** What does idempotent mean for configuration tools?

**Answer:** Applying twice has the same result as once

Safe to re-run.

**Q103.** What does this idempotent step print on two runs?

```bash
d=$(mktemp -d); f=$d/conf
for i in 1 2; do grep -q 'port=80' $f 2>/dev/null || echo 'port=80' >> $f; done
cat $f | wc -l | tr -d ' '
```

**Answer:** 1

Check-then-append keeps one line however many times it runs. Verified by running it in bash.

**Q104.** How do you detect drift?

**Answer:** terraform plan or Ansible --check --diff regularly

Alert on differences.

### [07 · Service Mesh &amp; Internal Networking Concepts](level-3/07-service-mesh-internal-networking.md)

**Q105.** What is service discovery?

**Answer:** How services find each other's addresses dynamically

DNS or a registry.

**Q106.** What does a service mesh add?

**Answer:** Consistent mTLS, retries, routing and telemetry between services

At the cost of complexity.

**Q107.** Why is mTLS useful internally?

**Answer:** Services authenticate each other and traffic is encrypted

Zero-trust networking.

**Q108.** When might you not need a mesh?

**Answer:** With few services, where simple DNS and libraries suffice

Start simple.

### [08 · Chaos Engineering Basics](level-3/08-chaos-engineering-basics.md)

**Q109.** What is chaos engineering?

**Answer:** Deliberately injecting controlled failures to test resilience

Learn before real incidents.

**Q110.** What should precede a chaos experiment?

**Answer:** A hypothesis, a small blast radius and an abort plan

Start in staging.

**Q111.** What does stopping one backend verify?

**Answer:** That health checks and the load balancer route around it

Redundancy must be proven.

**Q112.** What do you do with the findings?

**Answer:** Fix weaknesses and rerun the experiment

Improve.

### [09 · Observability at Scale (metrics, logs, traces)](level-3/09-observability-at-scale.md)

**Q113.** What are the three pillars of observability?

**Answer:** Metrics, logs and traces

Each answers different questions.

**Q114.** What does a trace show?

**Answer:** The path and timing of a single request across services

Find slow hops.

**Q115.** What is high-cardinality risk in metrics?

**Answer:** Too many label values explode storage and cost

Avoid user IDs as labels.

**Q116.** What makes alerts actionable?

**Answer:** They indicate user impact and link to a runbook

Page on symptoms.

### [10 · Capstone — Highly Available Web Service Design](level-3/10-capstone-ha-web-service-design.md)

**Q117.** What is a readiness check?

**Answer:** Whether an instance can serve traffic, including dependencies

Liveness differs.

**Q118.** Which design removes a single point of failure for a web tier?

**Answer:** Multiple instances behind a load balancer across zones

Plus a replicated database.

**Q119.** If one of 4 web servers can fail while still serving peak load of 300 req/s and each handles 100 req/s, are 4 enough?

```python
print(3*100>=300)
```

**Answer:** True

N-1 capacity: three remaining servers still handle 300. Verified by running it.

**Q120.** What belongs in the HA design doc?

**Answer:** Failure modes, recovery steps and tested assumptions

Prove by testing.

## Level 4 · Master

### [01 · Multi-Region / Multi-AZ Architecture Concepts](level-4/01-multi-region-multi-az-concepts.md)

**Q121.** What does multi-AZ protect against?

**Answer:** Failure of a single data centre (availability zone)

Latency stays low.

**Q122.** What does multi-region protect against?

**Answer:** Regional outages, at higher cost and complexity

Needs data replication strategy.

**Q123.** What is the hardest part of multi-region?

**Answer:** Data consistency and failover of state

Choose consistency model deliberately.

**Q124.** When is multi-region justified?

**Answer:** When the business impact of a regional outage exceeds the cost

Match architecture to SLA.

### [02 · Capacity Planning](level-4/02-capacity-planning.md)

**Q125.** Each instance safely handles 25 concurrent requests and peak is 400. With N+1 redundancy and 20% headroom, how many instances? (round up)

```python
import math
needed=math.ceil(400/25)
with_headroom=math.ceil(needed*1.2)
print(with_headroom+1)
```

**Answer:** 21

16 for peak, 20 with headroom (19.2 rounds up), plus one for N+1. Verified by running it.

**Q126.** What is the purpose of capacity planning?

**Answer:** Provision enough resources ahead of demand without overspending

Use trends and tests.

**Q127.** Why use percentiles rather than averages for latency?

**Answer:** Averages hide the slow tail users experience

p95 and p99.

**Q128.** What does load testing give you?

**Answer:** Evidence of where the system breaks, to ground capacity numbers

Test before launch.

### [03 · Incident Response &amp; Runbooks](level-4/03-incident-response-runbooks.md)

**Q129.** What is a runbook?

**Answer:** A documented procedure for diagnosing and fixing a known problem

Reduces time to recovery.

**Q130.** What is the first priority in an incident?

**Answer:** Mitigate user impact, then find the root cause

Stop the bleeding.

**Q131.** What is a blameless postmortem?

**Answer:** A review focused on systems and learning, not individual fault

Improves safety.

**Q132.** What should an incident have?

**Answer:** A clear incident commander and communication channel

Coordination.

### [04 · Security Patching Strategy at Scale](level-4/04-security-patching-at-scale.md)

**Q133.** Why patch security issues promptly?

**Answer:** Known vulnerabilities are quickly exploited

Prioritise by severity and exposure.

**Q134.** What does staged patching mean?

**Answer:** Patch a small group first, watch, then roll out widely

Limit blast radius.

**Q135.** What does a tool like Dependabot do?

**Answer:** Opens automated pull requests for dependency updates

Review and test them.

**Q136.** What do you do when you cannot patch immediately?

**Answer:** Apply compensating controls such as isolation or WAF rules

Document the exception.

### [05 · Choosing the Right HA Architecture for a Given SLA](level-4/05-choosing-ha-architecture-for-sla.md)

**Q137.** A 99.9% SLA allows how many minutes of downtime in a 30-day month? (rounded)

```python
print(round(30*24*60*0.001))
```

**Answer:** 43

43,200 minutes x 0.1% = 43.2. Verified by running it.

**Q138.** A 99.99% SLA allows about how many minutes of downtime per 30-day month? (1 decimal)

```python
print(round(30*24*60*0.0001,1))
```

**Answer:** 4.3

Each extra nine cuts allowed downtime tenfold. Verified by running it.

**Q139.** Why not always choose the highest-availability architecture?

**Answer:** Each extra nine costs disproportionately more; match it to business need

Cost versus risk.

**Q140.** What is an error budget?

**Answer:** The allowed unreliability implied by the SLO, spent on changes and incidents

Balance velocity and reliability.

### [06 · Cost-Aware Infrastructure Design](level-4/06-cost-aware-infrastructure-design.md)

**Q141.** What is rightsizing?

**Answer:** Matching instance sizes to actual utilisation

Review sustained usage.

**Q142.** What are reserved or committed-use discounts for?

**Answer:** Predictable baseline workloads

Use on-demand for spikes.

**Q143.** What is a hidden cost in cloud architectures?

**Answer:** Data transfer between zones and regions

Design for locality.

**Q144.** What should you tag resources with?

**Answer:** Owner, environment and cost centre

Cost allocation.

### [07 · Compliance &amp; Audit Considerations for Ops](level-4/07-compliance-audit-considerations.md)

**Q145.** What is the principle of least privilege in IAM?

**Answer:** Grant only the permissions needed for the task

Scope deploy roles narrowly.

**Q146.** Why keep audit logs?

**Answer:** To show who did what and when, for compliance and forensics

Protect them from tampering.

**Q147.** What is separation of duties?

**Answer:** No single person can complete a sensitive process alone

Reduces fraud and error.

**Q148.** What is evidence-as-code?

**Answer:** Automated, repeatable collection of compliance evidence

Cheaper audits.

### [08 · On-Call Culture &amp; Escalation Design](level-4/08-oncall-culture-escalation-design.md)

**Q149.** What makes a good alert?

**Answer:** Actionable, tied to user impact and with an owner

Avoid alert fatigue.

**Q150.** Why page on sustained error rates, not every 5xx?

**Answer:** Isolated errors are noise; sustained ones indicate ongoing impact

Threshold plus duration.

**Q151.** What is a healthy on-call practice?

**Answer:** Rotation, escalation paths, handoffs and retros to reduce toil

Sustainable.

**Q152.** What does an escalation policy define?

**Answer:** Who is paged next if the first person does not respond

Avoid unacknowledged pages.

### [09 · Platform Engineering &amp; Internal Developer Experience](level-4/09-platform-engineering-devex.md)

**Q153.** What does platform engineering provide?

**Answer:** Self-service paved roads for developers (templates, pipelines, environments)

Reduce cognitive load.

**Q154.** What does a service template give new services?

**Answer:** Good defaults: CI, logging, health checks and security

Golden paths.

**Q155.** How do you measure platform success?

**Answer:** Developer adoption, lead time and satisfaction

Treat devs as customers.

**Q156.** What is a risk of a platform team?

**Answer:** Building what developers do not need without feedback

Product mindset.

### [10 · Capstone — Designing a Resilient Global Platform](level-4/10-capstone-resilient-global-platform.md)

**Q157.** What is the capstone's goal?

**Answer:** Design a resilient global platform meeting stated SLOs, cost and compliance needs

Trade-offs explicit.

**Q158.** What belongs in the design doc?

**Answer:** Requirements, architecture, failure modes, runbooks and costs

Defend the decisions.

**Q159.** What evidence shows resilience?

**Answer:** Failure tests, restore drills and measured recovery times

Prove it.

**Q160.** What should you do about residual risks?

**Answer:** Document them and decide explicitly to accept, mitigate or transfer

Risk register.

## Mixed questions across all levels

**Q161.** What does this print?

```bash
set -eu
x=5
[ "$x" -gt 3 ] && echo big || echo small
name=""
echo "${name:-anon}"
n=0; for i in 1 2 3; do n=$((n + i)); done; echo $n
```

**Answer:** big
anon
6

${var:-default} substitutes when empty; arithmetic expansion sums the loop. Verified by running it in bash.

**Q162.** What does this print?

```bash
printf 'a 1\nb 2\na 3\n' | awk '{s[$1]+=$2} END {for (k in s) print k, s[k]}' | sort
```

**Answer:** a 4
b 2

awk accumulates per key. Verified by running it in bash.

**Q163.** What does this print?

```bash
set -e
f() { echo start; false; echo not-reached; }
f || echo handled
echo after
```

**Answer:** start
not-reached
after

A well-known gotcha: set -e is ignored inside a function that is called as part of an || list, so false does not stop f, f returns the status of its last command (0) and handled never prints. Verified by running it in bash.

**Q164.** How many minutes of downtime does a 99.95% monthly SLA allow in 30 days? (1 decimal)

```python
print(round(30*24*60*(1-0.9995),1))
```

**Answer:** 21.6

43,200 minutes x 0.05% = 21.6. Verified by running it.

**Q165.** A service needs 450 req/s peak; each instance handles 60 req/s. With N+1 how many instances? (round up for peak, then add one)

```python
import math
print(math.ceil(450/60)+1)
```

**Answer:** 9

8 for peak (7.5 rounded up) plus one spare. Verified by running it.

**Q166.** You disable SSH password login and close your only session; key login then fails. What should you have done?

**Answer:** Test key login in a second session first, validate with sshd -t and keep the first session open

Lock-out is the classic hardening mistake.

**Q167.** A deploy script fails halfway and is re-run. What property makes that safe?

**Answer:** Idempotency: every step checks state first so re-running reaches the same end state

mkdir -p, ln -sfn, check-then-append.

**Q168.** Why is renaming a symlink over 'current' used for deploys?

**Answer:** It switches releases atomically, so requests never see a half-deployed directory

Rollback is the same swap in reverse.

**Q169.** Your secrets file was committed to git by mistake. What is the correct response?

**Answer:** Rotate the exposed secrets immediately; removing the commit is not enough

Assume exposure.

**Q170.** Why should the same build artifact go to staging and production?

**Answer:** What was tested is exactly what ships; only configuration differs

Build once, promote.

**Q171.** What does a blue-green deploy let you do that an in-place deploy does not?

**Answer:** Test the new version fully before real traffic and roll back instantly by switching back

Cost: duplicate capacity.

**Q172.** Which alert is better: 'any 5xx' or 'error rate above 5% for 5 minutes'?

**Answer:** The sustained rate: it indicates ongoing user impact and avoids paging on noise

Alert fatigue hides real incidents.

**Q173.** What is the RPO of a nightly backup at best?

**Answer:** Up to 24 hours of data loss

RPO is bounded by backup frequency.

**Q174.** Why test restores rather than only checking that backups ran?

**Answer:** A backup that was never restored is unproven

Restore drills find broken backups.

**Q175.** Two independent web servers sit behind one load balancer and one database. What is the single point of failure?

**Answer:** The load balancer and the database (unless they are redundant too)

Redundancy must exist at every layer.

**Q176.** What is configuration drift and how do you detect it?

**Answer:** Reality diverging from declared config; detect with terraform plan or Ansible --check --diff on a schedule

Alert on unexpected diffs.

**Q177.** What is the first priority during an incident?

**Answer:** Mitigate user impact, then diagnose the root cause

Stop the bleeding first.

**Q178.** Why run containers as non-root and keep state outside them?

**Answer:** Limits compromise impact and keeps containers disposable and replaceable

Volumes or external services hold state.

**Q179.** What belongs in a runbook?

**Answer:** Symptoms, checks, safe fix steps, escalation contacts and how to verify recovery

Written before the incident.

## Practical coding questions

### Atomic releases with health check and rollback

Write a bash release manager: each release is a directory, 'current' is a symlink swapped atomically, a failed health check rolls back automatically, and old releases are pruned. It runs in a temp directory so it needs no root and touches no real server; real deployments also need service restarts and drained connections.

Starter code:

```
#!/usr/bin/env bash
# Atomic release switching with health check and automatic rollback (runs in a temp dir, no root needed)
set -euo pipefail
ROOT=$(mktemp -d)
mkdir -p "$ROOT/releases"

# TODO: switch_to <version>   point "$ROOT/current" at the release by creating a fresh symlink and renaming it over "current"
#       (rename is atomic; BSD mv uses -h, GNU mv uses -T)
# TODO: deploy <version> <healthy|broken>   create the release (write the mode to STATUS), remember the previous release,
#       switch, then "health check" by reading current/STATUS; if it is not "healthy" print "deploy <v>: health check FAILED",
#       switch back to the previous release and print "rolled back to <prev>", and return 1; otherwise print "deploy <v>: ok"
# TODO: prune <n>   keep the newest n releases (sorted by name), printing "pruned <name>" for each one removed (portable: no head -n -N)

deploy v1 healthy
deploy v2 healthy
deploy v3 broken || true
echo "current -> $(basename "$(readlink "$ROOT/current")")"
deploy v4 healthy
prune 2
echo "releases: $(ls -1 "$ROOT/releases" | tr '\n' ' ')"
echo "current -> $(basename "$(readlink "$ROOT/current")")"
```

Expected output:

```text
deploy v1: ok
deploy v2: ok
deploy v3: health check FAILED
rolled back to v2
current -> v2
deploy v4: ok
pruned v1
pruned v2
releases: v3 v4 
current -> v4
```

??? success "Show a solution"

    ```
    #!/usr/bin/env bash
    # Atomic release switching with health check and automatic rollback (runs in a temp dir, no root needed)
    set -euo pipefail
    ROOT=$(mktemp -d)
    mkdir -p "$ROOT/releases"

    switch_to() {                    # point "current" at a release by renaming a fresh symlink over it
      ln -sfn "$ROOT/releases/$1" "$ROOT/current.tmp"
      mv -fh "$ROOT/current.tmp" "$ROOT/current" 2>/dev/null || mv -fT "$ROOT/current.tmp" "$ROOT/current"   # BSD -h, GNU -T
    }

    deploy() {                       # deploy <version> <healthy|broken>
      local ver=$1 mode=$2 prev=""
      mkdir -p "$ROOT/releases/$ver"
      echo "$mode" > "$ROOT/releases/$ver/STATUS"
      [ -L "$ROOT/current" ] && prev=$(basename "$(readlink "$ROOT/current")")
      switch_to "$ver"
      if [ "$(cat "$ROOT/current/STATUS")" != healthy ]; then
        echo "deploy $ver: health check FAILED"
        if [ -n "$prev" ]; then switch_to "$prev"; echo "rolled back to $prev"; fi
        return 1
      fi
      echo "deploy $ver: ok"
    }

    prune() {                        # keep the newest $1 releases (portable: no head -n -N)
      ls -1 "$ROOT/releases" | sort | awk -v keep="$1" '{a[NR]=$0} END {for (i = 1; i <= NR - keep; i++) print a[i]}' |
      while read -r old; do rm -rf "$ROOT/releases/$old"; echo "pruned $old"; done
    }

    deploy v1 healthy
    deploy v2 healthy
    deploy v3 broken || true
    echo "current -> $(basename "$(readlink "$ROOT/current")")"
    deploy v4 healthy
    prune 2
    echo "releases: $(ls -1 "$ROOT/releases" | tr '\n' ' ')"
    echo "current -> $(basename "$(readlink "$ROOT/current")")"
    ```

    Creating the new symlink beside 'current' and renaming it over the old one is atomic, so readers never see a half-switched state; rollback is the same swap. Note that pruning by name keeps the broken v3 here, which is why real pipelines also delete or mark failed releases. Verified by running the script in bash on macOS.

### An access-log health report

Parse a small access log with awk: count statuses, compute the server-error rate, find the slowest path by average latency and decide whether to raise an alert. The log is generated inside the script, so no real server logs are used.

Starter code:

```
#!/usr/bin/env bash
# Access-log health report: status counts, error rate, slowest path and an alert decision
set -euo pipefail
LOG=$(mktemp)
cat > "$LOG" <<'LOGS'
10.0.0.1 GET /home 200 35
10.0.0.2 GET /api/items 200 120
10.0.0.3 POST /api/pay 500 910
10.0.0.1 GET /home 200 40
10.0.0.4 GET /api/items 503 30
10.0.0.2 POST /api/pay 500 880
10.0.0.5 GET /missing 404 12
10.0.0.3 GET /api/items 200 150
10.0.0.9 GET /home 200 33
10.0.0.6 POST /api/pay 200 420
LOGS

# TODO: print the status counts (sorted), then the totals line, the slowest path by average latency and the alert decision:
#   status counts:
#   <status> <count> ...
#   requests=<n> server_errors=<5xx count> rate=<percent with 1 decimal>%
#   slowest=<path> avg=<rounded>ms
#   ALERT: error rate above 15%   (or OK when the rate is 15% or less)
# Columns are: client method path status milliseconds
```

Expected output:

```text
status counts:
200 6
404 1
500 2
503 1
requests=10 server_errors=3 rate=30.0%
slowest=/api/pay avg=737ms
ALERT: error rate above 15%
```

??? success "Show a solution"

    ```
    #!/usr/bin/env bash
    # Access-log health report: status counts, error rate, slowest path and an alert decision
    set -euo pipefail
    LOG=$(mktemp)
    cat > "$LOG" <<'LOGS'
    10.0.0.1 GET /home 200 35
    10.0.0.2 GET /api/items 200 120
    10.0.0.3 POST /api/pay 500 910
    10.0.0.1 GET /home 200 40
    10.0.0.4 GET /api/items 503 30
    10.0.0.2 POST /api/pay 500 880
    10.0.0.5 GET /missing 404 12
    10.0.0.3 GET /api/items 200 150
    10.0.0.9 GET /home 200 33
    10.0.0.6 POST /api/pay 200 420
    LOGS

    echo "status counts:"
    awk '{c[$4]++} END {for (s in c) print s, c[s]}' "$LOG" | sort

    awk -v limit=15 '
      {total++; if ($4 >= 500) err++; sum[$3] += $5; n[$3]++}
      END {
        rate = 100 * err / total
        printf "requests=%d server_errors=%d rate=%.1f%%\n", total, err, rate
        worst = ""; wavg = 0
        for (p in sum) { avg = sum[p] / n[p]; if (avg > wavg) { wavg = avg; worst = p } }
        printf "slowest=%s avg=%.0fms\n", worst, wavg
        print (rate > limit ? "ALERT: error rate above " limit "%" : "OK")
      }' "$LOG"
    ```

    One awk pass accumulates totals and per-path sums, then END computes the rate and picks the path with the highest average. Alerting on a rate over a threshold is the same idea as the sustained-error alert in the monitoring lessons. Verified by running the script in bash on macOS.

---

More free practice: the [Server Ops Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
