# 01 · Server Basics & SSH Access

Before you can administer a server you need to know what kind of thing you're
actually logging into, and how to get a secure shell on it. This module covers
the mental model and the first login.

## What "a server" actually is

In practice you'll meet three flavors of "server," and the day-to-day admin
work is almost identical on all three:

- **Physical server** — a real machine in a rack (or under someone's desk).
  You reach it via an out-of-band management interface (iDRAC/iLO/IPMI) or
  physically, then SSH once the OS is up.
- **Virtual machine (VM)** — a slice of a physical host's CPU/RAM/disk,
  managed by a hypervisor (KVM, VMware, Hyper-V). Feels like a real machine
  to the OS running on it.
- **Cloud instance** — a VM (or occasionally bare metal) provisioned through
  a cloud provider's API (AWS EC2, GCP Compute Engine, Azure VM, DigitalOcean
  Droplet, Hetzner Cloud, etc.), billed by the hour/month, with the provider
  handling the hypervisor and physical hardware entirely.

For this course, assume a fresh Ubuntu/Debian-family cloud instance or VM
with only SSH access — that's the most common starting point in the real
world and everything here transfers directly to bare metal.

## SSH: your primary interface to the server

SSH (Secure Shell) gives you an encrypted remote shell. Almost everything
else in server administration — file transfer (`scp`/`rsync`), port
forwarding, running remote commands — is built on top of it.

### First connection

```bash
ssh root@203.0.113.10
```

The first time you connect to a given host, SSH shows you the server's host
key fingerprint and asks you to confirm it:

```text
The authenticity of host '203.0.113.10 (203.0.113.10)' can't be established.
ED25519 key fingerprint is SHA256:abcd1234...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Accepting this stores the key in `~/.ssh/known_hosts` on your local machine.
On a real deployment, verify the fingerprint out-of-band (from your cloud
provider's console output, for instance) before typing "yes" — this is the
step that protects you from a machine-in-the-middle silently swapping in a
different server.

### Generating and using an SSH key pair

Password auth over SSH is workable but weaker and slower than key-based auth
(we'll disable it entirely in the hardening module). Generate a key pair on
your **local** machine, not the server:

```bash
ssh-keygen -t ed25519 -C "you@example.com" -f ~/.ssh/id_ed25519
```

- `-t ed25519` — a modern, fast, secure key type. Use `-t rsa -b 4096` only if
  you must support very old systems that lack ed25519 support.
- `-C` — a comment, purely for your own bookkeeping (shows up next to the key
  wherever it's listed).
- `-f` — output path. Omit it and `ssh-keygen` will prompt interactively.

This produces two files:

- `~/.ssh/id_ed25519` — your **private** key. Never copy this to a server,
  paste it anywhere, or commit it to a repo. Treat it like a password.
- `~/.ssh/id_ed25519.pub` — your **public** key. Safe to share; this is what
  gets installed on servers you want to access.

Copy the public key to the server (if you still have password access, or the
provider lets you paste a key at instance-creation time):

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@203.0.113.10
```

`ssh-copy-id` appends your public key to `~/.ssh/authorized_keys` on the
remote user's home directory. If `ssh-copy-id` isn't available, the manual
equivalent is:

```bash
cat ~/.ssh/id_ed25519.pub | ssh root@203.0.113.10 \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
   cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

The permission bits matter: SSH refuses to trust `authorized_keys` if the
`.ssh` directory or the file itself is group- or world-writable.

### A minimal local SSH config

`~/.ssh/config` on your local machine lets you avoid retyping hostnames,
users, and key paths:

```text
Host webserver
    HostName 203.0.113.10
    User deploy
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

Now `ssh webserver` connects with the right user and key automatically —
useful once you're managing more than one host.

## Worked example: first login and a sanity check

```bash
# Connect using the alias defined above
ssh webserver

# Once connected, confirm who and where you are
whoami
hostname
uname -a
cat /etc/os-release | grep PRETTY_NAME
df -h /
free -h
```

Typical output on a fresh 1 vCPU / 1 GB cloud instance:

```text
deploy
web-01
Linux web-01 6.8.0-31-generic #31-Ubuntu SMP x86_64 GNU/Linux
PRETTY_NAME="Ubuntu 24.04.1 LTS"
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        25G  1.8G   22G   8% /
              total        used        free      shared  buff/cache   available
Mem:           957Mi       112Mi       650Mi       0.0Ki       194Mi       688Mi
```

That's your baseline: OS version, disk headroom, and memory. You'll come back
to `df -h` and `free -h` constantly once the server is doing real work.

!!! note "Not literally executed against a live server"
    The commands and output above are correct and runnable as written, but
    this course text is generated rather than run against a real cloud
    instance — treat sample output as representative, not as a captured
    transcript, and verify against your own server.

## Exercise

1. Spin up a free-tier or lowest-cost VM from any provider you have access to
   (or a local VM via VirtualBox/UTM/multipass if you'd rather not spend
   money), running Ubuntu 22.04 or 24.04.
2. Generate a new ed25519 key pair dedicated to this course (don't reuse a
   key you use elsewhere).
3. Get the public key onto the server and confirm you can `ssh` in **without**
   being prompted for a password.
4. Add a `Host` entry for it in your local `~/.ssh/config` and confirm
   `ssh <alias>` works.
5. Run `uname -a`, `cat /etc/os-release`, `df -h`, and `free -h` and note the
   values — you'll compare against them again in later modules.
