# 07 · Compliance & Audit Considerations for Ops

Everything built so far — IaC, backups, patching, incident response — turns
out to also be most of what auditors ask for when a system needs to meet a
compliance framework (SOC 2, ISO 27001, PCI-DSS, HIPAA, or an internal
security review). This module reframes those existing practices through a
compliance/audit lens, and covers the parts that are genuinely new: access
control, audit logging, and evidence collection.

## Compliance is mostly "prove you do what you already claim to do"

The biggest misconception about compliance work: it's not usually about
adopting brand-new practices, it's about **evidencing** practices that
should already exist for good operational reasons. Most frameworks map
cleanly onto earlier modules:

| Control area | Framework asks (paraphrased) | Where this is already covered |
|---|---|---|
| Change management | Changes are reviewed and tracked | IaC + PR review (Level 3 module 05) |
| Backup & recovery | Backups exist, are tested, are recoverable within a stated time | Level 3 module 03 |
| Access control | Access is least-privilege, reviewed, and revocable | This module, below |
| Vulnerability management | Known vulnerabilities are tracked and remediated on a schedule | Level 4 module 04 |
| Incident response | A documented process exists and incidents are logged | Level 4 module 03 |
| Monitoring | Systems are monitored, and alerts reach a responsible party | Level 3 module 09 |
| Audit logging | Who did what, when, is recorded and tamper-resistant | This module, below |

If a team already runs the earlier modules' practices well, most of a
compliance audit becomes "collect the evidence that's already being
generated," rather than building new capability under deadline pressure.

## Least-privilege access control

**Principle of least privilege**: every identity (human or service) gets
only the permissions it needs to do its job, no more, and access is
reviewed and revoked when no longer needed.

```hcl
# Terraform IAM policy: scoped to exactly what the deploy role needs,
# not a blanket AdministratorAccess grant
resource "aws_iam_policy" "deploy_role_policy" {
  name = "deploy-role-policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "ecs:UpdateService",
        "ecs:DescribeServices",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ]
      Resource = "arn:aws:ecs:*:123456789012:service/production/orders"
    }]
  })
}
```

```yaml
# Kubernetes RBAC: an on-call engineer can read logs/describe pods,
# but cannot delete resources or read secrets, in production
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { namespace: production, name: oncall-readonly }
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "events"]
    verbs: ["get", "list", "watch"]
  # deliberately no "secrets" resource, no "delete"/"create"/"update" verbs
```

**Access reviews** — a recurring (usually quarterly) process where access
grants are checked against "does this person/service still need this,"
and revoked if not:

```bash
# quarterly access review: list everyone with production database access,
# cross-reference against current team roster and role
psql -c "SELECT rolname FROM pg_roles WHERE rolcanlogin AND rolname LIKE 'prod_%';"
# → manually or via script, confirm each against HR/team data; revoke stale grants
```

Access that was correctly granted (an engineer needed prod DB access for a
migration) becomes a compliance gap the moment it's no longer needed and
nobody revokes it — this is exactly the kind of drift Level 3 module 06
covers, applied to permissions instead of infrastructure config.

## Audit logging: who did what, when, immutably

An audit log records security-relevant actions — logins, permission
changes, data access, infrastructure changes — in a way that's tamper-
resistant (a compromised account, or even a malicious insider, shouldn't be
able to quietly edit their own trail).

```bash
# CloudTrail (AWS): logs every API call, shipped to a separate account's
# S3 bucket with write-once semantics so the source account can't delete it
aws cloudtrail create-trail \
  --name org-audit-trail \
  --s3-bucket-name company-audit-logs-locked-account \
  --is-multi-region-trail \
  --enable-log-file-validation
```

```sql
-- Database audit logging: who read/modified sensitive tables, not just
-- application-level logs (which a compromised app can't be trusted to log honestly)
-- (Postgres pgaudit extension)
CREATE EXTENSION pgaudit;
ALTER SYSTEM SET pgaudit.log = 'read, write, role';
ALTER SYSTEM SET pgaudit.log_relation = on;
-- then filter to sensitive tables via pgaudit.role settings
```

The **separate account/bucket with write-once/immutable storage** detail
matters: audit logs stored in the same account/system they're auditing can
be altered or deleted by whoever compromises that system, defeating the
purpose. Shipping them to an isolated, restricted-access destination (and
enabling log file integrity validation) is what makes the trail trustworthy
as *evidence*, not just a debugging convenience.

## Evidence collection: build it into the pipeline, don't scramble at audit time

The single highest-leverage compliance habit: generate audit evidence as a
byproduct of normal operations, continuously, rather than manually
assembling it under deadline when an auditor asks.

```yaml
# CI: every production deploy automatically produces an evidence artifact —
# who approved it, what changed, when, linked to the PR
- name: record deploy evidence
  run: |
    cat <<EOF >> deploy-evidence.log
    timestamp: $(date -Iseconds)
    deployed_by: ${{ github.actor }}
    approved_by: $(gh pr view ${{ github.event.pull_request.number }} --json reviews -q '.reviews[0].author.login')
    commit: ${{ github.sha }}
    service: orders
    EOF
    aws s3 cp deploy-evidence.log s3://company-audit-logs/deploys/$(date +%s).log
```

```bash
# quarterly automated evidence pull, instead of manual screenshots the week of the audit
./scripts/collect-evidence.sh --quarter 2026-Q3
# → pulls: backup test results (Level 3 module 03), patch compliance %
#   (Level 4 module 04), access review sign-offs, incident postmortems
#   (Level 4 module 03), all into one dated evidence package
```

An organization that can run `collect-evidence.sh` and produce a complete,
already-true evidence package at any time is in a fundamentally different
(and far less stressful) position than one that starts building evidence
from scratch when the auditor's request lands.

## Data classification and handling

Not all data needs the same controls — classify it, and apply controls
proportional to sensitivity:

```
Public:        marketing content — no special controls
Internal:      internal docs, non-sensitive logs — access limited to employees
Confidential:  customer PII, financial data — encryption at rest + in
               transit, access logged (pgaudit above), least-privilege
Restricted:    payment card data (PCI scope), health records (HIPAA) —
               additional controls per the specific framework (e.g. PCI's
               network segmentation requirements, tokenization instead of
               storing raw card numbers at all where possible)
```

Getting this classification wrong in the "under-classify" direction
(treating confidential data as merely internal) is the more common and more
dangerous mistake — it means weaker controls are applied to data that
needed stronger ones, discovered only during an incident or an audit
finding.

## Exercise

1. Pick one compliance framework relevant to a system you know (or SOC 2
   as a generic default) and map three of its control areas to practices
   from earlier modules, identifying what evidence each practice already
   produces (or would need to start producing).
2. Write an IAM policy or RBAC role scoped to the minimum permissions a
   specific role (on-call engineer, CI deploy pipeline) actually needs, and
   list what a broader "just give them admin" grant would have exposed
   unnecessarily.
3. Design an audit logging setup for one sensitive action in your system
   (e.g. reading a customer PII table, or changing a production
   permission) — specify what's logged, where it's stored, and why that
   storage location is resistant to tampering by whoever's being audited.
4. Write a `collect-evidence.sh` outline (pseudocode is fine) that would
   assemble, on demand, evidence for backup testing, patch compliance, and
   access reviews from the practices in earlier modules.
