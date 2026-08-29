# 02 · Initial Server Hardening Checklist

A freshly created server (especially one you got with root/password login)
is not safe to leave as-is for more than a few minutes on the public
internet — automated scanners will find port 22 and start trying credentials
almost immediately. This module is the checklist you run through on every
new box before doing anything else.

## 1. Create a non-root user with sudo

Never operate as `root` day-to-day. Create a dedicated administrative user:

```bash
# As root, on the server
adduser deploy
usermod -aG sudo deploy
```

`adduser` (Debian/Ubuntu) walks you through setting a password and optional
account info interactively. On RHEL/CentOS-family systems, use:

```bash
useradd -m -s /bin/bash deploy
passwd deploy
usermod -aG wheel deploy
```

(`wheel` is the sudo-equivalent group on RHEL-family distros.)

Verify sudo works before you do anything else, from a **second** terminal
(so you don't lock yourself out if something's wrong):

```bash
ssh deploy@203.0.113.10
sudo whoami   # should print: root
```

## 2. Install your SSH key for the new user

Repeat the key-installation step from module 1, but for `deploy` instead of
`root`:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub deploy@203.0.113.10
```

Confirm key-based login works for `deploy` before touching SSH config.

## 3. Harden the SSH daemon

Edit `/etc/ssh/sshd_config` (as root or via sudo):

```text
# /etc/ssh/sshd_config
Port 22
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
X11Forwarding no
MaxAuthTries 3
AllowUsers deploy
```

What each line buys you:

- `PermitRootLogin no` — root can no longer log in over SSH at all, even with
  a key. You escalate via `sudo` from `deploy` instead.
- `PasswordAuthentication no` — kills the entire class of password-guessing
  attacks against SSH. Only key-based auth is accepted.
- `MaxAuthTries 3` — caps how many auth attempts a single connection gets.
- `AllowUsers deploy` — an explicit allowlist; anyone else can't even attempt
  to authenticate, regardless of credentials.

Validate the config syntax **before** restarting the daemon — a typo here can
lock you out permanently:

```bash
sudo sshd -t
```

If that prints nothing, the config is syntactically valid. Then restart:

```bash
sudo systemctl restart ssh      # Debian/Ubuntu service name
# sudo systemctl restart sshd   # RHEL/CentOS service name
```

**Keep your current SSH session open** while you test a brand-new connection
in another terminal. Only close the original session once you've confirmed
the new one logs in cleanly as `deploy` with your key. This is the single
most important safety habit in server hardening — never close your only
working connection until you've proven the new config works.

## 4. Set up a firewall

### ufw (Debian/Ubuntu)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH        # or: sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

`ufw` ("uncomplicated firewall") is a friendly frontend over `iptables`/
`nftables`. The order matters: allow SSH **before** you `enable`, or you'll
cut yourself off the moment the firewall activates.

### iptables (lower-level, portable to more distros)

The equivalent raw `iptables` rules:

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```

Raw `iptables` rules don't persist across reboot by default — on Debian/
Ubuntu install `iptables-persistent` (`apt install iptables-persistent`) or,
more simply, just use `ufw`, which handles persistence for you.

### firewalld (RHEL/CentOS/Fedora)

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

## 5. Install and configure fail2ban

fail2ban watches log files for repeated auth failures and temporarily bans
the offending IP at the firewall level — useful defense-in-depth even with
password auth already disabled (it also covers other services like web app
login forms if you configure a filter for them).

```bash
sudo apt update && sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
```

A minimal SSH jail, `/etc/fail2ban/jail.local`:

```ini
[sshd]
enabled = true
port = 22
maxretry = 5
bantime = 1h
findtime = 10m
```

Restart to apply: `sudo systemctl restart fail2ban`.

## 6. Keep the system patched

```bash
sudo apt update && sudo apt upgrade -y      # Debian/Ubuntu
# sudo dnf upgrade -y                        # RHEL/Fedora
```

Consider unattended security upgrades (Debian/Ubuntu):

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

## Worked example: the full checklist, in order

```bash
# 1-2. user + key (as root)
adduser deploy
usermod -aG sudo deploy
ssh-copy-id -i ~/.ssh/id_ed25519.pub deploy@203.0.113.10

# --- switch to a NEW terminal, log in as deploy, confirm sudo works ---

# 3. SSH hardening (edit /etc/ssh/sshd_config as shown above, then:)
sudo sshd -t && sudo systemctl restart ssh

# --- open ANOTHER new terminal, confirm deploy@host still logs in ---

# 4. firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# 5. fail2ban
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban

# 6. patch
sudo apt update && sudo apt upgrade -y
```

!!! warning "Order matters"
    Always confirm the *new* access path works before you close off the
    *old* one. That applies to SSH config changes, firewall rules, and user
    permissions alike.

## Exercise

Using the VM from module 1:

1. Create a `deploy` user with sudo access and install your SSH key for it.
2. Harden `sshd_config` per this module, validate with `sshd -t`, and
   restart — verifying a fresh connection works before closing your original
   session.
3. Enable `ufw` (or `firewalld`) allowing only SSH, HTTP, and HTTPS.
4. Install and enable `fail2ban` with the jail above.
5. Run `sudo ufw status verbose` and `sudo systemctl status fail2ban` and
   confirm both show active/running.
