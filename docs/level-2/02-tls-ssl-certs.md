# 02 · TLS/SSL Certificates (Let's Encrypt / certbot)

TLS is what turns `http://` into `https://`: it encrypts traffic between
client and server and proves (via a certificate signed by a trusted CA)
that the server is who it claims to be. Let's Encrypt provides free,
short-lived (90-day) certificates issued automatically via the ACME
protocol, and `certbot` is the standard client for getting and renewing
them.

## Prerequisites

- A domain name with an A/AAAA record pointing at the server's public IP.
- Port 80 (and ideally 443) reachable from the internet — Let's Encrypt's
  HTTP-01 challenge fetches a token over plain HTTP to prove you control the
  domain.
- nginx already serving that `server_name` (see module 1).

## Installing certbot and getting a cert (nginx plugin)

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx

sudo certbot --nginx -d example.com -d www.example.com
```

The nginx plugin does three things for you: proves domain ownership by
temporarily serving the ACME challenge through your existing nginx vhost,
downloads the certificate to `/etc/letsencrypt/live/example.com/`, and
**edits your nginx config in place** to add a `listen 443 ssl` block plus an
HTTP→HTTPS redirect. Always run `sudo nginx -t` and read the diff of what it
changed if you want to know exactly what happened:

```bash
sudo cat /etc/nginx/sites-available/example.com.conf
```

Resulting shape (roughly what certbot writes):

```nginx
server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;   # certbot adds this redirect
}
```

`fullchain.pem` contains your certificate plus the intermediate CA
certificate (clients need the whole chain to verify trust); `privkey.pem` is
the private key — never copy it off the box or commit it anywhere.

## Auto-renewal

Let's Encrypt certs expire every 90 days by design (to force automation and
limit blast radius of a leaked key). The certbot package installs a
systemd timer that handles this without you doing anything:

```bash
systemctl list-timers | grep certbot
sudo certbot renew --dry-run     # simulate a renewal without changing anything
```

`renew` only actually replaces certs that are within 30 days of expiry, so
running it manually anytime is safe — it's a no-op for certs that aren't
close to expiring. If nginx's config changed after the cert was issued
(e.g. you added a new `server` block), certbot's renewal hook reloads nginx
automatically so the new cert is picked up without manual intervention.

## Checking a cert from the outside

```bash
curl -vI https://example.com 2>&1 | grep -A2 "expire date\|subject:"
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -noout -dates -subject -issuer
```

`-servername` matters for SNI (Server Name Indication) — without it, a
server hosting multiple TLS vhosts on one IP may return the wrong
certificate (usually a default/self-signed one), and you'll be debugging
the wrong problem.

## Strong TLS defaults

certbot's `options-ssl-nginx.conf` already sets sane modern defaults, but
know what's in there:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;   # never allow TLSv1.0/1.1 or SSLv3
ssl_prefer_server_ciphers off;    # let the (modern) client pick, per current best practice
ssl_session_cache shared:le_nginx_SSL:10m;
```

Add HSTS once you're confident every subdomain is served over HTTPS
(it tells browsers to *never* attempt plain HTTP for this domain again, so
adding it before HTTPS is fully working everywhere can lock you out):

```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
```

## Worked example: cert issuance and renewal dry run

```bash
sudo certbot --nginx -d demo.example.com --non-interactive --agree-tos -m ops@example.com
sudo certbot certificates                 # lists all managed certs + expiry
sudo certbot renew --dry-run --cert-name demo.example.com
sudo nginx -t && sudo systemctl reload nginx
```

`--non-interactive --agree-tos -m <email>` is the scripted/unattended form
you'd use from a provisioning script rather than answering prompts by hand.

## Exercise

1. Point a real (or test) domain's A record at a server you control, issue a
   Let's Encrypt cert with `certbot --nginx`, and verify HTTPS works with
   `curl -vI https://yourdomain`.
2. Confirm the HTTP→HTTPS redirect certbot added actually returns `301`
   (`curl -I http://yourdomain`).
3. Run `certbot renew --dry-run` and read its output to confirm it identifies
   your cert and simulates a successful renewal without changing the
   on-disk cert files (`ls -la /etc/letsencrypt/live/yourdomain/` timestamps
   unchanged).
