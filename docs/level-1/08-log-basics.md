# 08 · Log Basics (journalctl, /var/log)

When something goes wrong on a server, logs are almost always the first
place you look. This module covers the two main log sources on a modern
Linux box: the systemd journal, and the traditional flat files under
`/var/log`.

## journalctl: the systemd journal

Every service managed by systemd has its stdout/stderr automatically
captured into the journal — a structured, binary log store — with no extra
configuration needed.

### Core queries

```bash
journalctl                          # entire journal, oldest first
journalctl -e                       # jump to the end (like less +G)
journalctl -f                       # follow live, like tail -f
journalctl -u nginx                 # only this unit's logs
journalctl -u nginx -f               # follow only this unit
journalctl --since "1 hour ago"
journalctl --since "2026-08-29 09:00" --until "2026-08-29 10:00"
journalctl -p err                   # only priority "err" and above
journalctl -p warning -u nginx --since today
journalctl -k                       # kernel messages only (like dmesg)
journalctl --disk-usage             # how much space the journal is using
```

Priority levels, from most to least severe: `emerg`, `alert`, `crit`, `err`,
`warning`, `notice`, `info`, `debug`. `-p err` means "err or more severe" —
so it also includes `crit`, `alert`, and `emerg`.

### Filtering by time and unit together

```bash
journalctl -u myapp --since "10 min ago" --no-pager
```

`--no-pager` is worth adding whenever you're piping journalctl output into
another command or a script — otherwise it opens in `less` and blocks.

### Journal persistence and size limits

By default on many distros the journal is only kept in memory and lost on
reboot, unless persistent storage is enabled:

```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

Cap how much disk the journal can use, in `/etc/systemd/journald.conf`:

```ini
[Journal]
Storage=persistent
SystemMaxUse=500M
```

Then `sudo systemctl restart systemd-journald`.

## /var/log: traditional flat-file logs

Not everything logs through journald — many services (especially anything
not managed as a systemd unit, or configured explicitly to log to a file)
write plain text log files under `/var/log`:

```text
/var/log/syslog          # general system log (Debian/Ubuntu)
/var/log/messages        # general system log (RHEL-family)
/var/log/auth.log        # authentication attempts, sudo usage (Debian/Ubuntu)
/var/log/secure          # same, RHEL-family
/var/log/nginx/access.log
/var/log/nginx/error.log
/var/log/dpkg.log        # package install/remove history (Debian/Ubuntu)
```

Standard tools for reading them:

```bash
tail -f /var/log/nginx/access.log        # follow live
tail -n 100 /var/log/nginx/error.log     # last 100 lines
less /var/log/syslog                      # paged, searchable (/ to search)
grep "Failed password" /var/log/auth.log  # find failed SSH login attempts
zcat /var/log/nginx/access.log.2.gz       # read a rotated, gzipped log
```

## Log rotation with logrotate

Left unmanaged, flat-text logs grow forever and eventually fill the disk.
`logrotate` (installed by default on nearly every distro, run automatically
via cron/systemd timer) handles rotating, compressing, and eventually
deleting old logs.

Most packages (nginx included) ship their own rotation config under
`/etc/logrotate.d/`:

```text
# /etc/logrotate.d/nginx (simplified, typical shape)
/var/log/nginx/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

Reading this: rotate daily, keep 14 rotated copies, gzip them (but delay
compressing the most recent rotated file by one cycle so anything still
mid-write to it isn't corrupted), don't error if the log file is missing,
skip rotation if the file is empty, and after rotating, signal nginx
(`USR1`) to reopen its log file handles pointing at the new empty file.

Test a logrotate config without waiting for the schedule:

```bash
sudo logrotate -d /etc/logrotate.d/nginx     # dry run, shows what it would do
sudo logrotate -f /etc/logrotate.d/nginx     # force rotation now
```

## Worked example: chasing down a 502 error

```bash
# 1. Check the app service itself for crashes
journalctl -u myapp --since "30 min ago" -p err --no-pager

# 2. Check nginx's error log for what it saw from the upstream
sudo tail -n 50 /var/log/nginx/error.log

# 3. Cross-reference timestamps between the two
journalctl -u myapp --since "2026-08-29 09:55:00" --until "2026-08-29 10:00:00"

# 4. Confirm the service is actually up now
systemctl status myapp --no-pager
```

This sequence — application log, proxy log, then correlate by timestamp —
is the standard shape of most "why did this request fail" investigations.

## Exercise

1. On your VM, run `journalctl -u nginx --since "1 hour ago"` (or whatever
   service you've set up) and identify at least one `info`-level startup
   message.
2. Deliberately cause an nginx error (e.g., point `proxy_pass` at a port
   nothing is listening on, per module 06's app, and hit the site) and find
   the corresponding line in `/var/log/nginx/error.log`.
3. Write a `logrotate` config for a log file your `hello` service produces
   (redirect its output to a file instead of just stdout for this exercise),
   rotating daily and keeping 7 copies compressed.
4. Test it with `logrotate -d` and then force one rotation with `logrotate -f`.
