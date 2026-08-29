# 04 · Process Management (systemd)

Every modern mainstream Linux distribution uses **systemd** as its init
system and service manager — it starts services at boot, restarts them if
they crash, and gives you one consistent interface (`systemctl`) for
controlling all of them. This module is your working knowledge of it.

## The core commands

```bash
sudo systemctl start nginx      # start a service now
sudo systemctl stop nginx       # stop it
sudo systemctl restart nginx    # stop then start
sudo systemctl reload nginx     # re-read config without dropping connections (if supported)
sudo systemctl enable nginx     # start automatically on boot
sudo systemctl disable nginx    # don't start automatically on boot
systemctl status nginx          # current state + recent log lines
systemctl is-active nginx       # prints active/inactive/failed
systemctl is-enabled nginx      # prints enabled/disabled
systemctl list-units --type=service --state=running
```

`restart` vs `reload`: `reload` asks a well-behaved service to re-read its
config in place (nginx does this by gracefully finishing in-flight
connections on old workers while starting new ones with the new config) —
prefer it over `restart` for zero-downtime config changes when the service
supports it. Not all services implement `reload`; check with
`systemctl show nginx -p CanReload`.

## Writing your own systemd service unit

Say you have a small app — a Python/Node/Go binary — that you want managed
like any other system service (auto-restart on crash, start on boot, log
capture via journald). Create a unit file:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App
After=network.target

[Service]
Type=simple
User=deploy
Group=deploy
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/venv/bin/python /opt/myapp/main.py
Restart=on-failure
RestartSec=5
Environment=APP_ENV=production
EnvironmentFile=-/etc/myapp/myapp.env

[Install]
WantedBy=multi-user.target
```

Field by field:

- `After=network.target` — a soft ordering hint: try to start this after
  networking is up. It does **not** mean the network is fully ready in every
  case; use `Wants=network-online.target` + `After=network-online.target` if
  your app genuinely fails without working DNS/routes at startup.
- `Type=simple` — the most common type: systemd considers the service
  "started" as soon as `ExecStart`'s process launches. Use `Type=forking` for
  old-style daemons that fork and exit the parent.
- `User=`/`Group=` — never run app processes as root unless there's a
  specific reason to.
- `Restart=on-failure` + `RestartSec=5` — automatically restart the process
  5 seconds after a non-zero exit, but not after a clean `stop`.
- `EnvironmentFile=-/etc/myapp/myapp.env` — load `KEY=value` pairs from this
  file into the process environment; the leading `-` means "don't fail unit
  activation if the file is missing."
- `WantedBy=multi-user.target` — which boot target pulls this unit in when
  enabled; `multi-user.target` is the normal "system is up, non-graphical"
  target most server services attach to.

After creating or editing a unit file, you must tell systemd to reload its
configuration before the changes take effect:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
```

`enable --now` is shorthand for `enable` (start on boot) plus `start`
(start immediately) in one command.

## Reading service logs via the unit

```bash
journalctl -u myapp                 # all logs for this unit
journalctl -u myapp -f              # follow, like tail -f
journalctl -u myapp --since "10 min ago"
journalctl -u myapp -p err          # only error-level and above
```

Anything the service writes to stdout/stderr is automatically captured by
journald — no separate log-shipping setup needed for basic cases. (Full
`journalctl` usage is covered in module 8.)

## Inspecting and managing plain processes

Below the systemd layer, the everyday process tools still apply:

```bash
ps aux | grep nginx          # list processes, filter by name
ps -ef --forest              # process tree view
top                          # live resource usage, interactive
htop                         # nicer live view (if installed)
kill -TERM 4821              # ask a process to terminate gracefully
kill -9 4821                 # force-kill (SIGKILL) — last resort
pkill -f "main.py"           # kill by matching command line
pgrep -a nginx               # list PIDs + command line for matches
```

Prefer `systemctl stop <unit>` over manually `kill`ing a process that's
managed by systemd — a raw `kill` can race with systemd's own restart logic
and leave things in a confusing state.

## Worked example: a managed "hello" service

```bash
sudo mkdir -p /opt/hello
cat <<'EOF' | sudo tee /opt/hello/hello.sh > /dev/null
#!/usr/bin/env bash
while true; do
  echo "hello from $(hostname) at $(date -Iseconds)"
  sleep 5
done
EOF
sudo chmod +x /opt/hello/hello.sh

cat <<'EOF' | sudo tee /etc/systemd/system/hello.service > /dev/null
[Unit]
Description=Hello demo service
After=network.target

[Service]
Type=simple
User=deploy
ExecStart=/opt/hello/hello.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now hello
systemctl status hello --no-pager
journalctl -u hello -n 5 --no-pager
```

Expected `status` output includes `Active: active (running)` and the most
recent journal lines showing the "hello from ..." messages every 5 seconds.

## Exercise

1. Write the `hello.service` unit above and get it running with
   `systemctl status` showing `active (running)`.
2. Kill the underlying process directly with `pkill -f hello.sh` and confirm
   systemd notices and restarts it automatically (check `systemctl status`
   again — note the new PID and `Restart` count).
3. `systemctl disable hello`, reboot the VM (`sudo reboot`), and confirm
   after it comes back that the service is **not** running — then re-enable
   it and reboot again to confirm it now starts automatically.
