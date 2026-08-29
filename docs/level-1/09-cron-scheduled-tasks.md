# 09 · Cron Jobs & Scheduled Tasks

Backups, cleanup jobs, periodic health checks, certificate renewals — a lot
of server ops work is "run this on a schedule and leave it alone." This
module covers `cron`, the classic Unix scheduler, and systemd timers, its
modern alternative.

## Cron syntax

A crontab line has five time fields followed by the command:

```text
# ┌───────────── minute (0-59)
# │ ┌───────────── hour (0-23)
# │ │ ┌───────────── day of month (1-31)
# │ │ │ ┌───────────── month (1-12)
# │ │ │ │ ┌───────────── day of week (0-6, Sunday=0)
# │ │ │ │ │
# * * * * *  command-to-run
```

Common patterns:

```text
0 3 * * *          every day at 3:00 AM
*/15 * * * *       every 15 minutes
0 * * * *          every hour, on the hour
0 0 * * 0          every Sunday at midnight
30 2 1 * *         2:30 AM on the 1st of every month
0 9-17 * * 1-5     every hour from 9am-5pm, Monday-Friday
```

`*/15` means "every 15th value" — for minutes, that's `0,15,30,45`. Ranges
(`9-17`) and lists (`1,3,5`) both work in any field.

## Editing a user's crontab

```bash
crontab -e          # edit the current user's crontab (opens $EDITOR)
crontab -l           # list the current user's crontab
crontab -r           # remove the current user's entire crontab (careful!)
sudo crontab -u deploy -e   # edit another user's crontab
```

A real crontab entry, with the two habits that save you the most debugging
time later — an absolute `PATH` and output redirected to a log file:

```text
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

0 3 * * * /opt/myapp/scripts/nightly-backup.sh >> /var/log/myapp-backup.log 2>&1
```

Cron runs jobs with a minimal environment — no shell profile, no login
`PATH`. A script that works fine when you run it manually can silently fail
under cron simply because `python3`, `node`, or a custom binary isn't found.
Always set `PATH` explicitly in the crontab, or better, use absolute paths
inside the script itself (`/usr/bin/python3` rather than `python3`).

`>> file 2>&1` appends both stdout and stderr to a log file — without this,
cron normally emails output to the user (assuming mail is even configured,
which on a bare cloud VM it usually isn't) and you'd never see failures.

## System-wide cron locations

Beyond per-user crontabs, system jobs commonly live in:

```text
/etc/crontab               # system crontab — has an extra "user" field
/etc/cron.d/*               # drop-in files, same format as /etc/crontab
/etc/cron.daily/            # scripts run once a day (via anacron/cron)
/etc/cron.hourly/
/etc/cron.weekly/
/etc/cron.monthly/
```

`/etc/crontab` and `/etc/cron.d/*` entries include a username field between
the schedule and the command (because, unlike a per-user crontab, there's no
implicit "owner"):

```text
# /etc/cron.d/myapp-backup
0 3 * * * deploy /opt/myapp/scripts/nightly-backup.sh >> /var/log/myapp-backup.log 2>&1
```

## systemd timers — the modern alternative

A systemd timer pairs with a regular service unit and gives you better
logging (through journald), dependency handling, and the ability to catch up
on missed runs (`Persistent=true`) if the machine was off at the scheduled
time — none of which plain cron gives you.

```ini
# /etc/systemd/system/myapp-backup.service
[Unit]
Description=Nightly backup for myapp

[Service]
Type=oneshot
User=deploy
ExecStart=/opt/myapp/scripts/nightly-backup.sh
```

```ini
# /etc/systemd/system/myapp-backup.timer
[Unit]
Description=Run myapp-backup nightly at 3am

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

- `Type=oneshot` — the service is expected to run once and exit, not stay
  running (as opposed to `Type=simple` for long-lived daemons in module 04).
- `Persistent=true` — if the machine was powered off (or the timer service
  wasn't running) at 3:00 AM, run the missed job as soon as it's back up.
  Plain cron has no equivalent — a missed cron run is just gone.
- `RandomizedDelaySec=300` — jitter the actual start time by up to 5 minutes,
  useful across a fleet so many machines don't all hit a shared resource
  (like a backup target or database) at the exact same second.

Enable and check it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp-backup.timer
systemctl list-timers --all | grep myapp-backup
journalctl -u myapp-backup.service        # logs from the last run(s)
```

`systemctl list-timers` shows you exactly when each timer last ran and when
it's next scheduled — something plain cron gives you no easy way to inspect.

## Worked example: a nightly cleanup job, both ways

**Via cron:**

```bash
cat <<'EOF' | sudo tee /etc/cron.d/tmp-cleanup > /dev/null
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
0 4 * * * deploy find /opt/myapp/tmp -type f -mtime +7 -delete >> /var/log/tmp-cleanup.log 2>&1
EOF
sudo chmod 644 /etc/cron.d/tmp-cleanup
```

**Via systemd timer** (same job, better observability):

```bash
cat <<'EOF' | sudo tee /etc/systemd/system/tmp-cleanup.service > /dev/null
[Unit]
Description=Delete tmp files older than 7 days

[Service]
Type=oneshot
User=deploy
ExecStart=/usr/bin/find /opt/myapp/tmp -type f -mtime +7 -delete
EOF

cat <<'EOF' | sudo tee /etc/systemd/system/tmp-cleanup.timer > /dev/null
[Unit]
Description=Run tmp-cleanup daily at 4am

[Timer]
OnCalendar=*-*-* 04:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now tmp-cleanup.timer
```

## Exercise

1. Write a small script that appends a timestamp to `/var/log/heartbeat.log`
   (`date -Iseconds >> /var/log/heartbeat.log`).
2. Schedule it via cron to run every 5 minutes (`*/5 * * * *`), with an
   explicit `PATH` and output redirected.
3. Wait ~10-15 minutes and confirm multiple timestamps appear in the log.
4. Reimplement the same job as a systemd service + timer pair with
   `OnCalendar=*:0/5` (every 5 minutes) and `Persistent=true`, and confirm
   via `systemctl list-timers` that it's scheduled correctly and via
   `journalctl -u <name>.service` that it's actually running.
