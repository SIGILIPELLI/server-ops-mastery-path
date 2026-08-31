# 01 · Reverse Proxies (nginx basics)

A reverse proxy sits in front of one or more backend applications and
handles the stuff you don't want every app to reimplement: TLS termination,
routing by hostname/path, buffering slow clients, compression, and a stable
public port (80/443) in front of app processes that bind to internal ports.
nginx is the default choice for this on most Linux fleets.

## Installing and the config layout

```bash
sudo apt update && sudo apt install -y nginx
sudo systemctl enable --now nginx
```

Debian/Ubuntu nginx layout:

```
/etc/nginx/nginx.conf              # main config, includes the rest
/etc/nginx/sites-available/        # one file per virtual host, written here
/etc/nginx/sites-enabled/          # symlinks into sites-available — nginx only reads these
/etc/nginx/conf.d/                 # global snippets, auto-included by nginx.conf
```

A site is "off" if its symlink isn't in `sites-enabled`, even if the file
exists in `sites-available`. That two-directory split exists so you can keep
a config around without it being live.

## A minimal reverse proxy vhost

Say your app is a Node/Python/Go process listening on `127.0.0.1:3000`.

```nginx
# /etc/nginx/sites-available/myapp.conf
server {
    listen 80;
    server_name myapp.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Why each header matters:

- `Host $host` — without this, the backend sees `Host: 127.0.0.1` instead of
  `myapp.example.com`, which breaks apps that generate absolute URLs or do
  host-based routing/vhost logic themselves.
- `X-Real-IP` / `X-Forwarded-For` — the backend's TCP connection is always
  from nginx (127.0.0.1); these headers are how the real client IP survives
  the hop. Never trust `X-Forwarded-For` from the public internet directly —
  only trust it because *you* set it here, at the only entry point.
- `X-Forwarded-Proto` — tells the backend whether the original request was
  HTTP or HTTPS, since nginx may terminate TLS before it ever reaches the
  app. Frameworks use this to decide whether to set `Secure` cookies or
  redirect to HTTPS.

Enable it and reload:

```bash
sudo ln -s /etc/nginx/sites-available/myapp.conf /etc/nginx/sites-enabled/
sudo nginx -t          # ALWAYS test before reloading
sudo systemctl reload nginx
```

`nginx -t` validates syntax and referenced files without touching the
running process — get in the habit of running it before every reload. A
broken config caught by `-t` is an annoyance; one that gets reloaded live is
an outage.

## Path-based routing to multiple backends

```nginx
server {
    listen 80;
    server_name example.com;

    location /api/ {
        proxy_pass http://127.0.0.1:4000/;   # trailing slash rewrites the prefix away
        proxy_set_header Host $host;
    }

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
    }
}
```

The trailing slash on `proxy_pass http://127.0.0.1:4000/;` matters: a
request to `/api/users` is forwarded to the backend as `/users` (the
`/api/` prefix is stripped). Omit the trailing slash and it forwards as
`/api/users` unchanged. This is one of the most common nginx gotchas —
always check which behavior you actually want.

## Timeouts and buffering

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_connect_timeout 5s;
    proxy_read_timeout 60s;
    proxy_send_timeout 60s;
    proxy_buffering on;
    client_max_body_size 10m;   # reject large uploads before they hit the app
}
```

`proxy_read_timeout` is how long nginx waits for the backend to send data
once connected — bump it for slow endpoints (report generation, large
exports) rather than globally, using a dedicated `location` block for those
paths.

## Serving static files directly (skip the app for these)

```nginx
location /static/ {
    alias /opt/myapp/static/;
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

Letting nginx serve static assets directly (instead of proxying them to the
app) is both faster and takes load off the application process — nginx is
built for exactly this.

## Worked example: two apps behind one nginx

```bash
# app A on :3000, app B on :3001, both already running via systemd
sudo tee /etc/nginx/sites-available/multiapp.conf <<'EOF'
server {
    listen 80;
    server_name a.example.com;
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
server {
    listen 80;
    server_name b.example.com;
    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
sudo ln -sf /etc/nginx/sites-available/multiapp.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
curl -H "Host: a.example.com" http://localhost/
curl -H "Host: b.example.com" http://localhost/
```

The `curl -H "Host: ..."` trick lets you test name-based virtual hosting
from the server itself without needing real DNS yet — nginx picks the
`server_name` block purely from the `Host` header.

## Exercise

1. Run two toy backends on `:3000` and `:3001` (e.g. `python3 -m http.server`
   twice with different ports, or two tiny Flask/Express apps).
2. Write an nginx vhost that reverse-proxies `/a/` to the first and `/b/` to
   the second on a single `server_name`, with correct trailing-slash prefix
   stripping.
3. Confirm with `curl -v` that `X-Forwarded-For` and `Host` arrive correctly
   at each backend (log `request.headers` / `req.headers` in your toy app to
   check), and that `nginx -t` catches a deliberately broken brace before
   you reload.
