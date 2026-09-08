# 03 · Backup & Disaster Recovery Strategy

High availability (module 01) protects against a single component failing.
It does not protect against data corruption, a bad deploy that silently
deletes rows, ransomware, or an entire region disappearing — for those you
need backups and a disaster recovery (DR) plan, which are a different
discipline with different math.

## RPO and RTO

Two numbers drive every backup/DR decision:

- **RPO (Recovery Point Objective)** — how much data you can afford to
  lose, measured in time. "RPO = 1 hour" means: after a disaster, you may
  lose up to the last hour of writes, but no more.
- **RTO (Recovery Time Objective)** — how long you can afford to be down
  while recovering. "RTO = 4 hours" means: from the moment disaster is
  declared, the system must be back up within 4 hours.

These are business decisions, not engineering ones — but engineering has to
translate them into a concrete backup schedule and restore procedure, and
prove the numbers are actually achievable (see "test your restores" below).

| Target | Implies |
|---|---|
| RPO = 24h | Nightly backup is enough |
| RPO = 15 min | Continuous replication or frequent incremental backups + WAL/binlog shipping |
| RTO = 4h | Manual restore from backup onto fresh infrastructure is fine |
| RTO = 5 min | Needs a warm/hot standby already running, not "restore from backup" |

## The 3-2-1 rule

A widely used baseline: keep **3** copies of data, on **2** different types
of media/storage, with **1** copy off-site (a different physical location,
ideally a different provider/region than production).

```
Copy 1: production database (the live data)
Copy 2: nightly snapshot on the same cloud provider, different volume
Copy 3: nightly dump shipped to a different region/provider's object storage
```

The point of "different media/location" is to survive failure modes that
take out more than one copy at once — a bad disk, an accidental `rm -rf` on
the wrong volume, a whole-region cloud outage, or a compromised admin
account that has access to only one of the storage backends.

## Backup types

- **Full backup** — a complete copy every time. Simple to restore from (one
  file), expensive in storage and time to produce.
- **Incremental backup** — only what changed since the *last* backup (full
  or incremental). Cheapest to produce, but restoring means replaying the
  full backup plus every incremental since, in order — more moving parts,
  more ways to have a broken chain.
- **Differential backup** — everything changed since the last *full*
  backup. Restoring needs only the last full + the last differential — a
  middle ground between the other two.

```bash
# Full pg_dump (logical backup, portable across postgres versions/OS)
pg_dump -Fc -f /backups/app_full_2026-08-31.dump app_db

# Restore
pg_restore -d app_db_restored /backups/app_full_2026-08-31.dump
```

For databases specifically, **point-in-time recovery (PITR)** — continuous
shipping of write-ahead logs (WAL in Postgres, binlogs in MySQL) alongside
periodic full backups — is what gets you a low RPO (minutes, not a full
day) without full backups running constantly.

## Worked example: nightly Postgres backup with retention, shipped off-site

```bash
#!/usr/bin/env bash
# /usr/local/bin/backup-db.sh — nightly full dump + off-site copy + retention
set -euo pipefail

DB=app_db
BACKUP_DIR=/var/backups/postgres
REMOTE=s3://company-backups-offsite/app_db/
RETAIN_DAYS=14
STAMP=$(date +%F)
FILE="$BACKUP_DIR/${DB}_${STAMP}.dump"

mkdir -p "$BACKUP_DIR"
pg_dump -Fc -f "$FILE" "$DB"

# verify the dump is restorable-shaped before trusting it (catches truncated/corrupt files)
pg_restore --list "$FILE" > /dev/null

# ship off-site
aws s3 cp "$FILE" "$REMOTE"

# prune local copies older than retention window (remote bucket has its own lifecycle policy)
find "$BACKUP_DIR" -name "${DB}_*.dump" -mtime +"$RETAIN_DAYS" -delete

logger -t backup-db "backup of $DB completed: $FILE"
```

```
# /etc/cron.d/backup-db
0 2 * * * postgres /usr/local/bin/backup-db.sh >> /var/log/backup-db.log 2>&1
```

The `pg_restore --list` line matters more than it looks: a backup job that
"succeeds" (exit 0) while silently writing a truncated or empty file is
worse than no backup, because it creates false confidence. Verify the
artifact, not just the exit code.

## Test your restores — the rule everyone skips

**An untested backup is a hypothesis, not a backup.** The only way to know
your RTO is achievable, and that the backup file actually contains
restorable data, is to actually restore it — on a schedule, not "whenever
there's time."

```bash
#!/usr/bin/env bash
# /usr/local/bin/test-restore.sh — run monthly against a scratch DB, alert on failure
set -euo pipefail
LATEST=$(ls -t /var/backups/postgres/app_db_*.dump | head -1)
TEST_DB=app_db_restore_test

dropdb --if-exists "$TEST_DB"
createdb "$TEST_DB"
if pg_restore -d "$TEST_DB" "$LATEST"; then
  ROWS=$(psql -d "$TEST_DB" -tAc "SELECT count(*) FROM users;")
  echo "restore ok, users table has $ROWS rows"
else
  logger -t test-restore "RESTORE FAILED for $LATEST — page on-call"
  exit 1
fi
```

Beyond "does it restore," periodically check that the *row counts and
recency* look sane — a backup that restores cleanly but is silently three
months stale (because the cron job quietly stopped running) passes a naive
restore test while still failing the actual goal.

## Disaster recovery plan structure

A DR plan is a document (kept somewhere that survives the disaster it
describes — not only on the server that might be the disaster) covering:

1. **Scenarios covered** — single-server loss, region loss, data
   corruption/ransomware, accidental deletion. Each has a different
   recovery path.
2. **Roles** — who declares a disaster, who executes the restore, who
   communicates status to stakeholders.
3. **Step-by-step recovery procedure** per scenario, specific enough that
   someone who didn't build the system could follow it under pressure —
   this overlaps heavily with Level 4's "Incident Response & Runbooks."
4. **Verified RPO/RTO** from the last DR drill, not the theoretical number
   from the architecture diagram.
5. **DR drill schedule** — e.g. quarterly, restoring into an isolated
   environment and timing the whole process end to end.

## How It Actually Works

**Why `pg_dump -Fc` produces a restorable file while a snapshot of the
data directory taken mid-write might not.** `pg_dump` doesn't copy raw
files — it opens a transaction at a consistent snapshot (via Postgres's
MVCC) and reads each table's rows through the normal query engine, writing
them out in a custom archive format with a built-in table of contents.
Because it goes through MVCC, concurrent writes during the dump don't
corrupt it — the dump reflects one consistent point-in-time view, the same
guarantee an ordinary `SELECT` gets. A raw filesystem copy of `/var/lib/postgresql`
taken without stopping the database, by contrast, can capture pages
mid-write and produce a physically inconsistent copy — which is exactly
why physical/PITR backups need either a filesystem snapshot with WAL
replay to reach consistency, or a tool (`pg_basebackup`) that coordinates
with the write-ahead log, rather than a plain `cp` or `tar`.

**Why WAL shipping is what turns "nightly backup" into "RPO of minutes."**
Postgres writes every change to the write-ahead log *before* applying it
to the actual data pages (write-ahead logging is what makes crash recovery
possible at all). PITR backup tooling archives each completed WAL segment
continuously as it's produced, on top of a periodic full base backup.
Restoring means: load the base backup, then replay WAL segments forward
from that point up to any target timestamp you choose — the RPO becomes
"however recent the last archived WAL segment is" (often seconds) instead
of "how long since the last full dump," because the full dump is now only
the recovery *starting point*, not the boundary of what's recoverable.

**Why `pg_restore --list` catches truncation that `pg_dump`'s exit code
doesn't.** A dump can be truncated by a disk-full condition, a killed
process, or a network interruption mid-transfer to S3, and still leave a
partial file on disk with `pg_dump`'s own exit code showing success if the
truncation happened *after* the local write completed (e.g. during the
`aws s3 cp` step). `pg_restore --list` has to actually parse the archive's
internal table-of-contents structure to enumerate its contents — a
truncated or corrupted file fails that parse immediately, catching the
class of failure ("the exit code lied") that motivates verifying the
artifact itself rather than trusting the producing command's return
status.

## Exercise

1. Write `backup-db.sh` against a local Postgres or MySQL instance, verify
   the dump with the appropriate list/check command, and schedule it via
   cron.
2. Intentionally corrupt or drop a table in a scratch copy of the database,
   then use only your backup script's output to restore it — time the whole
   process from "data is gone" to "data is back and verified."
3. Write down what your actual measured RTO was from step 2, compare it to
   a target RTO you pick for this exercise (e.g. 30 minutes), and list two
   concrete changes you'd make to close the gap if it's not met.
