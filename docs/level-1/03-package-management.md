# 03 · Package Management

Almost everything you install on a Linux server — a web server, a database,
a runtime — comes through the distribution's package manager rather than a
manual download-and-compile. This module covers the two families you'll meet
most often: `apt` (Debian/Ubuntu) and `yum`/`dnf` (RHEL/CentOS/Fedora/Amazon
Linux).

## apt (Debian, Ubuntu)

### Core commands

```bash
sudo apt update                 # refresh the local package index
sudo apt upgrade                # upgrade all installed packages
sudo apt install nginx          # install a package
sudo apt remove nginx           # remove a package, keep config files
sudo apt purge nginx            # remove a package AND its config files
sudo apt autoremove             # remove packages no longer needed as dependencies
apt list --installed | grep nginx
apt-cache search "web server"   # search available packages by keyword
apt show nginx                  # show details/version/description for a package
```

`apt update` does **not** install anything — it just refreshes the list of
what's available from configured repositories (`/etc/apt/sources.list` and
`/etc/apt/sources.list.d/*.list`). You need to run it before `install` or
`upgrade` picks up new versions, but you don't need it before every single
command.

### Pinning a specific version

```bash
apt-cache madison nginx          # list available versions
sudo apt install nginx=1.24.0-2ubuntu7
```

Useful when you need to match a version across a fleet, or avoid a known-bad
release.

### Adding a third-party repository

Many vendors (Docker, PostgreSQL, Node.js) ship their own apt repo for
newer versions than the distro provides:

```bash
# Example pattern (illustrative — check the vendor's current instructions)
curl -fsSL https://download.example.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/example.gpg
echo "deb [signed-by=/usr/share/keyrings/example.gpg] https://download.example.com/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/example.list
sudo apt update
sudo apt install example-package
```

The GPG key step matters: apt refuses to install from a repo it can't verify
the signature of, which is what stops a compromised mirror from silently
swapping in malicious packages.

## yum / dnf (RHEL, CentOS, Fedora, Amazon Linux)

`dnf` is the modern replacement for `yum` (RHEL 8+/Fedora); the command
syntax is nearly identical and `yum` is often just aliased to `dnf` now.

```bash
sudo dnf check-update            # like apt update, but doesn't modify state
sudo dnf upgrade                 # upgrade all installed packages
sudo dnf install nginx
sudo dnf remove nginx
sudo dnf search "web server"
dnf info nginx
dnf list installed | grep nginx
sudo dnf autoremove
```

Adding a repo (RPM-based) typically means dropping a `.repo` file into
`/etc/yum.repos.d/`:

```ini
# /etc/yum.repos.d/example.repo
[example]
name=Example Repo
baseurl=https://download.example.com/rpm/el9/
enabled=1
gpgcheck=1
gpgkey=https://download.example.com/gpg
```

## Distro detection in scripts

If you're writing scripts that need to run on either family, detect the
package manager rather than hardcoding one:

```bash
#!/usr/bin/env bash
set -euo pipefail

if command -v apt-get >/dev/null 2>&1; then
    PKG_INSTALL="sudo apt-get install -y"
    sudo apt-get update
elif command -v dnf >/dev/null 2>&1; then
    PKG_INSTALL="sudo dnf install -y"
elif command -v yum >/dev/null 2>&1; then
    PKG_INSTALL="sudo yum install -y"
else
    echo "Unsupported distro: no apt-get, dnf, or yum found" >&2
    exit 1
fi

$PKG_INSTALL nginx git curl
```

`command -v <tool>` is the portable way to check whether a binary exists on
`PATH` — prefer it over `which`, which isn't guaranteed to exist on minimal
images.

## Worked example: installing and verifying a web server

```bash
# Debian/Ubuntu
sudo apt update
sudo apt install -y nginx
systemctl is-active nginx        # -> active
systemctl is-enabled nginx       # -> enabled (starts on boot)
curl -I http://localhost         # -> HTTP/1.1 200 OK
nginx -v                         # -> nginx version: nginx/1.24.0
dpkg -L nginx | head -5          # -> list files installed by the package
```

`dpkg -L <package>` (Debian/Ubuntu) or `rpm -ql <package>` (RHEL-family) is
often the fastest way to answer "where did this config file come from?" when
you're debugging an unfamiliar server.

## Exercise

On the hardened VM from module 2:

1. Install `nginx` via your distro's package manager and confirm it's both
   `active` and `enabled` with `systemctl`.
2. Use `apt-cache madison nginx` (or `dnf list --showduplicates nginx`) to
   see what versions are available in your configured repos.
3. Remove `nginx` with `purge`/equivalent and confirm `/etc/nginx` is gone
   afterward (`ls /etc/nginx` should fail).
4. Reinstall it, and this time list every file the package placed on disk.
