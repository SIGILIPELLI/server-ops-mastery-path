# 05 · Basic Networking Concepts

You don't need to be a network engineer to run servers, but you do need a
working model of IP addresses, subnets, DNS, and ports — because nearly
every "it doesn't work" question in server ops reduces to one of these four.

## IP addresses and subnets

An IPv4 address like `203.0.113.10` identifies a host on a network. A
**subnet mask** (or CIDR prefix) says how much of that address is the
"network" portion vs. the "host" portion.

`203.0.113.0/24` means: the first 24 bits (3 octets — `203.0.113`) are fixed
as the network, leaving the last 8 bits (0-255) for individual hosts. That's
a network of 256 addresses (`203.0.113.0` through `203.0.113.255`), with
`.0` as the network address and `.255` as the broadcast address, leaving 254
usable host addresses.

Common prefixes you'll see:

| CIDR | Usable hosts | Typical use |
|------|---------------|-------------|
| `/32` | 1 | a single host route |
| `/30` | 2 | point-to-point link |
| /24 | 254 | a typical small LAN or VPC subnet |
| /16 | 65,534 | a large private network |

Private (non-internet-routable) ranges you'll see constantly inside cloud
VPCs and internal networks:

```text
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

Inspect a machine's own addressing:

```bash
ip addr show          # modern tool: interfaces + addresses
ip route show          # routing table — where traffic to each destination goes
```

## DNS: names to addresses

DNS translates human-readable names (`example.com`) into IP addresses.
Key record types you'll actually touch as a server admin:

- **A** record — hostname → IPv4 address (`api.example.com → 203.0.113.10`)
- **AAAA** record — hostname → IPv6 address
- **CNAME** record — hostname → another hostname (an alias)
- **MX** record — mail routing for a domain
- **TXT** record — arbitrary text, commonly used for domain verification
  and SPF/DKIM email authentication

Lookup tools:

```bash
dig example.com                 # detailed query, shows the full response
dig +short example.com          # just the resolved IP(s)
dig example.com MX               # query a specific record type
nslookup example.com             # older, simpler alternative to dig
host example.com                 # another simple alternative
```

DNS changes take time to propagate because of **TTL** (time to live) —
resolvers cache a record for however many seconds the TTL says before
re-checking. If you're about to make a DNS change that matters for a
migration or cutover, lower the TTL (e.g., to 60-300 seconds) a day ahead of
time so the eventual real change propagates fast.

## Ports and the client/server model

A **port** is a number (0-65535) that lets multiple network services share
one IP address — the OS uses the combination of (IP, port, protocol) to
route incoming traffic to the right listening process.

Well-known ports you'll see constantly:

| Port | Protocol | Service |
|------|----------|---------|
| 22 | TCP | SSH |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 53 | TCP/UDP | DNS |
| 5432 | TCP | PostgreSQL |
| 3306 | TCP | MySQL/MariaDB |
| 6379 | TCP | Redis |

Check what's actually listening on a server:

```bash
sudo ss -tulpn
```

Reading `ss -tulpn` output:

- `t`/`u` — TCP / UDP sockets
- `l` — only listening sockets (not established connections)
- `p` — show the owning process
- `n` — numeric (don't resolve port numbers to service names)

```text
Netid  State   Local Address:Port   Peer Address:Port  Process
tcp    LISTEN  0.0.0.0:22           0.0.0.0:*           users:(("sshd",pid=812,fd=3))
tcp    LISTEN  127.0.0.1:5432       0.0.0.0:*           users:(("postgres",pid=1120,fd=6))
tcp    LISTEN  0.0.0.0:80           0.0.0.0:*           users:(("nginx",pid=1560,fd=6))
```

Note the difference between `0.0.0.0:80` (listening on all interfaces —
reachable from outside) and `127.0.0.1:5432` (listening only on loopback —
reachable only from processes on the same machine, which is exactly what you
want for a database that the app on the same box connects to over
localhost).

`ss` is the modern replacement for the older `netstat -tulpn`, which still
works on most systems but is deprecated upstream.

## Connectivity troubleshooting toolkit

```bash
ping -c 4 example.com          # basic reachability + latency (ICMP)
traceroute example.com          # see the network path, hop by hop
curl -v https://example.com     # full HTTP request/response, verbose
curl -I https://example.com     # headers only (HEAD-like)
telnet example.com 443          # raw TCP connect test to a specific port
nc -zv example.com 443          # same idea, via netcat, scriptable
```

`nc -zv` is often the quickest way to answer "is anything even listening on
that port from here?" without worrying about HTTP semantics at all.

## Worked example: diagnosing "the site is down"

A realistic troubleshooting sequence when a report comes in that a site is
unreachable:

```bash
# 1. Does the name resolve at all?
dig +short example.com

# 2. Is the host reachable at the network layer?
ping -c 4 example.com

# 3. Is anything listening on port 443 there?
nc -zv example.com 443

# 4. If step 3 succeeds, get the actual HTTP response
curl -I https://example.com

# 5. On the server itself, confirm nginx is listening and running
ssh webserver 'sudo ss -tulpn | grep :443; systemctl is-active nginx'
```

Each step isolates a layer: DNS, network reachability, the listening socket,
the application response, and the service's own state — so you know exactly
where the chain breaks instead of guessing.

## Exercise

On your VM:

1. Run `ip addr show` and identify the VM's private and/or public IP and its
   CIDR prefix.
2. Run `dig +short` against a domain you control (or any public domain) and
   note the TTL with plain `dig <domain>` (look at the number column before
   record type).
3. Run `sudo ss -tulpn` and identify every listening port, which process
   owns it, and whether it's bound to `0.0.0.0` or `127.0.0.1`.
4. Use `nc -zv` to test whether port 443 is reachable on a domain that
   should be serving HTTPS, and explain in one sentence what a "connection
   refused" vs. a "timeout" would each tell you about the failure.
