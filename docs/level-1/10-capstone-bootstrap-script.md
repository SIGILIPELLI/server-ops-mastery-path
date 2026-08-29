# 10 · Capstone — Server Bootstrap Script

The capstone for Level 1: a single script that takes a brand-new server from
"just booted, root/password access only" to "hardened, running a web server
behind a firewall, managed by systemd" — combining every module in this
level into one repeatable, idempotent tool.

## What it needs to do

1. Create a non-root admin user and grant sudo (module 2).
2. Install the caller's SSH public key for that user (module 1).
3. Harden `sshd_config` — disable root login and password auth (module 2).
4. Install and enable `ufw`, allowing only SSH/HTTP/HTTPS (module 2).
5. Update packages and install nginx (module 3).
6. Confirm nginx is enabled and running via `systemctl` (module 4).
7. Write a systemd env file and a placeholder app service (modules 4, 7).
8. Set up log rotation awareness by confirming journald persistence
   (module 8).
9. Install a cron job (or timer) for nightly cleanup (module 9).
10. Print a final summary report.

## The script

```bash
#!/usr/bin/env bash
#
# bootstrap.sh — Level 1 capstone: harden a fresh server and stand up
# a basic web-serving stack, idempotently.
#
# Usage: sudo ./bootstrap.sh <admin_user> <path_to_ssh_pubkey>
# Example: sudo ./bootstrap.sh deploy /home/me/.ssh/id_ed25519.pub

set -euo pipefail

if [ "$EUID" -ne 0 ]; then
    echo "Run this script as root (or with sudo)." >&2
    exit 1
fi

ADMIN_USER="${1:?Usage: $0 <admin_user> <path_to_ssh_pubkey>}"
PUBKEY_PATH="${2:?Usage: $0 <admin_user> <path_to_ssh_pubkey>}"

if [ ! -f "$PUBKEY_PATH" ]; then
    echo "ERROR: public key file not found at $PUBKEY_PATH" >&2
    exit 1
fi

log() { echo "[bootstrap] $*"; }

# --- 1 & 2. Create admin user, grant sudo, install SSH key -----------------
if id "$ADMIN_USER" &>/dev/null; then
    log "User $ADMIN_USER already exists — skipping creation"
else
    log "Creating user $ADMIN_USER"
    adduser --disabled-password --gecos "" "$ADMIN_USER"
    usermod -aG sudo "$ADMIN_USER"
fi

USER_HOME="/home/$ADMIN_USER"
mkdir -p "$USER_HOME/.ssh"
cat "$PUBKEY_PATH" >> "$USER_HOME/.ssh/authorized_keys"
sort -u -o "$USER_HOME/.ssh/authorized_keys" "$USER_HOME/.ssh/authorized_keys"
chown -R "$ADMIN_USER:$ADMIN_USER" "$USER_HOME/.ssh"
chmod 700 "$USER_HOME/.ssh"
chmod 600 "$USER_HOME/.ssh/authorized_keys"
log "SSH key installed for $ADMIN_USER"

# --- 3. Harden SSH -----------------------------------------------------------
SSHD_CONFIG="/etc/ssh/sshd_config"
log "Hardening $SSHD_CONFIG"
cp "$SSHD_CONFIG" "${SSHD_CONFIG}.bak.$(date +%s)"

set_sshd_option() {
    local key="$1" value="$2"
    if grep -qE "^\s*#?\s*${key}\s+" "$SSHD_CONFIG"; then
        sed -i -E "s|^\s*#?\s*${key}\s+.*|${key} ${value}|" "$SSHD_CONFIG"
    else
        echo "${key} ${value}" >> "$SSHD_CONFIG"
    fi
}

set_sshd_option "PermitRootLogin" "no"
set_sshd_option "PasswordAuthentication" "no"
set_sshd_option "KbdInteractiveAuthentication" "no"
set_sshd_option "MaxAuthTries" "3"
set_sshd_option "AllowUsers" "$ADMIN_USER"

if sshd -t; then
    systemctl restart ssh || systemctl restart sshd
    log "SSH hardened and restarted"
else
    echo "ERROR: sshd_config failed validation — restoring backup" >&2
    cp "${SSHD_CONFIG}.bak."* "$SSHD_CONFIG"
    exit 1
fi

# --- 4. Firewall -------------------------------------------------------------
log "Configuring ufw"
if ! command -v ufw >/dev/null 2>&1; then
    apt-get update -y
    apt-get install -y ufw
fi
ufw --force default deny incoming
ufw --force default allow outgoing
ufw allow OpenSSH
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
log "ufw enabled: $(ufw status | head -1)"

# --- 5 & 6. Install nginx ----------------------------------------------------
log "Installing nginx"
apt-get update -y
apt-get install -y nginx
systemctl enable --now nginx

if systemctl is-active --quiet nginx; then
    log "nginx is active and enabled"
else
    echo "ERROR: nginx failed to start" >&2
    systemctl status nginx --no-pager
    exit 1
fi

# --- 7. App env + placeholder service ---------------------------------------
log "Writing baseline app config skeleton"
mkdir -p /etc/myapp
if [ ! -f /etc/myapp/myapp.env ]; then
    cat > /etc/myapp/myapp.env <<EOF
APP_ENV=production
LOG_LEVEL=info
EOF
fi
chown root:"$ADMIN_USER" /etc/myapp/myapp.env
chmod 640 /etc/myapp/myapp.env

# --- 8. Confirm journald persistence ----------------------------------------
log "Enabling persistent journald storage"
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald

# --- 9. Nightly cleanup timer ------------------------------------------------
log "Installing nightly cleanup timer"
cat > /etc/systemd/system/tmp-cleanup.service <<'EOF'
[Unit]
Description=Delete /tmp files older than 7 days

[Service]
Type=oneshot
ExecStart=/usr/bin/find /tmp -maxdepth 1 -type f -mtime +7 -delete
EOF

cat > /etc/systemd/system/tmp-cleanup.timer <<'EOF'
[Unit]
Description=Run tmp-cleanup daily at 4am

[Timer]
OnCalendar=*-*-* 04:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

systemctl daemon-reload
systemctl enable --now tmp-cleanup.timer

# --- 10. Summary --------------------------------------------------------------
echo
echo "======================================================================"
echo " Bootstrap complete"
echo "======================================================================"
echo " Admin user:        $ADMIN_USER (sudo, SSH key-only)"
echo " SSH:                root login disabled, password auth disabled"
echo " Firewall (ufw):     $(ufw status | head -1)"
echo " Web server:         nginx — $(systemctl is-active nginx), $(systemctl is-enabled nginx)"
echo " App config:         /etc/myapp/myapp.env (640, root:$ADMIN_USER)"
echo " Journal:             persistent storage enabled"
echo " Scheduled cleanup:   tmp-cleanup.timer — $(systemctl is-enabled tmp-cleanup.timer)"
echo "======================================================================"
echo " Next: log in as $ADMIN_USER from a NEW terminal to confirm access"
echo " BEFORE closing this session."
echo "======================================================================"
```

## Design notes

- **Idempotency throughout** — `id "$ADMIN_USER" &>/dev/null` before
  creating the user, `sort -u` on `authorized_keys` to avoid duplicate key
  entries, checking for an existing env file before overwriting it. Running
  the script a second time should be safe and mostly a no-op.
- **Backup before mutating `sshd_config`**, and restore it automatically if
  `sshd -t` fails validation — this is the single highest-consequence file
  in the whole script, and a bad edit could lock out all future SSH access.
- **Fail loudly, not silently** — every risky step (`sshd -t`, the nginx
  active check) has an explicit failure branch that prints diagnostics and
  exits non-zero, rather than continuing into an inconsistent state.
- **The final summary block** gives you, at a glance, exactly what the
  script changed — useful both for you right after running it and for
  whoever reads the deploy log later.

## Worked example: running it on a fresh VM

```bash
scp bootstrap.sh root@203.0.113.10:/root/
scp ~/.ssh/id_ed25519.pub root@203.0.113.10:/root/mykey.pub
ssh root@203.0.113.10
chmod +x bootstrap.sh
./bootstrap.sh deploy ./mykey.pub
```

Expected tail of output:

```text
======================================================================
 Bootstrap complete
======================================================================
 Admin user:        deploy (sudo, SSH key-only)
 SSH:                root login disabled, password auth disabled
 Firewall (ufw):     Status: active
 Web server:         nginx — active, enabled
 App config:         /etc/myapp/myapp.env (640, root:deploy)
 Journal:             persistent storage enabled
 Scheduled cleanup:   tmp-cleanup.timer — enabled
======================================================================
 Next: log in as deploy from a NEW terminal to confirm access
 BEFORE closing this session.
======================================================================
```

At that point — from a **new** terminal — `ssh deploy@203.0.113.10` should
work with your key, `sudo whoami` should print `root`, and
`curl -I http://203.0.113.10` should return `HTTP/1.1 200 OK` from nginx.
Only then close the original root session.

## Exercise

1. Run the full script against a fresh VM, following the worked example.
2. Verify each summary line manually: SSH as `deploy`, check `sudo ufw
   status verbose`, `systemctl status nginx`, `ls -l /etc/myapp/myapp.env`,
   and `systemctl list-timers | grep tmp-cleanup`.
3. Run the script a **second** time against the same server and confirm it
   completes cleanly with no errors and no duplicated `authorized_keys`
   entries — proving it's idempotent.
4. Extend the script yourself: add a step that installs `fail2ban` (module
   2) with a minimal SSH jail, and add its status to the final summary
   block.
