# Nginx Commands & Config I Actually Use

Started this after spending too much time digging through nginx docs every time I needed to tweak a server block or debug a 502. This covers the CLI commands, config patterns, and troubleshooting steps I reach for regularly as a backend developer.

Not a full nginx manual — just the practical stuff.

---

## Table of Contents

- [CLI Commands](#cli-commands)
- [Service Management](#service-management)
- [Configuration Structure](#configuration-structure)
- [Server Block Patterns](#server-block-patterns)
- [Reverse Proxy](#reverse-proxy)
- [SSL / HTTPS](#ssl--https)
- [Logs](#logs)
- [Performance Tuning](#performance-tuning)
- [Security Headers](#security-headers)
- [Troubleshooting](#troubleshooting)

---

## CLI Commands

| Command | What it does |
|---|---|
| `nginx -t` | **Test your config before applying it.** Always run this before reloading. Catches syntax errors |
| `nginx -T` | Same as `-t` but also prints the entire resolved config — useful for seeing what nginx actually sees |
| `nginx -s reload` | Reload config without dropping existing connections. The safe way to apply changes |
| `nginx -s stop` | Fast shutdown — drops all connections immediately |
| `nginx -s quit` | Graceful shutdown — waits for current requests to finish before stopping |
| `nginx -v` | Print nginx version |
| `nginx -V` | Print version AND all compile-time options and modules — useful for checking if a module is available |
| `nginx -c /path/nginx.conf` | Start nginx with a specific config file instead of the default |

> **Rule of thumb:** Always run `nginx -t` before `nginx -s reload`. A bad config can take down your server if you skip this.

---

## Service Management

On most modern Linux systems you'll use `systemctl` to manage nginx rather than the `nginx -s` commands directly.

| Command | What it does |
|---|---|
| `sudo systemctl start nginx` | Start nginx |
| `sudo systemctl stop nginx` | Stop nginx |
| `sudo systemctl restart nginx` | Full restart — briefly drops connections. Use `reload` instead when possible |
| `sudo systemctl reload nginx` | Reloads config gracefully — no dropped connections |
| `sudo systemctl status nginx` | Shows if nginx is running, recent logs, and the active PID |
| `sudo systemctl enable nginx` | Auto-start nginx on system boot |
| `sudo systemctl disable nginx` | Remove auto-start on boot |
| `sudo nginx -t && sudo systemctl reload nginx` | The combo I always use — test first, then reload only if config is valid |

---

## Configuration Structure

Understanding where things live saves a lot of time.

```
/etc/nginx/
├── nginx.conf              ← main config file
├── conf.d/                 ← drop-in config files (always loaded)
│   └── default.conf
└── sites-available/        ← config files (not active until symlinked)
    └── myapp.conf
/etc/nginx/sites-enabled/   ← symlinks to sites-available (what's actually active)
/var/log/nginx/
├── access.log
└── error.log
/var/www/html/              ← default web root
```

**The sites-available / sites-enabled pattern:**
```bash
# enable a site
sudo ln -s /etc/nginx/sites-available/myapp.conf /etc/nginx/sites-enabled/

# disable a site (just remove the symlink, file stays safe in sites-available)
sudo rm /etc/nginx/sites-enabled/myapp.conf
```

---

## Server Block Patterns

The configs I reuse most often.

### Static site

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Single Page App (React / Vue / Angular)

The key is the `try_files` fallback to `index.html` so client-side routing works.

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/myapp/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # cache static assets aggressively
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### Serve on a specific path

```nginx
location /api/ {
    proxy_pass http://localhost:8000/;  # note trailing slash — strips /api prefix
}
```

---

## Reverse Proxy

The pattern I use most — nginx in front of a FastAPI / Node / any backend app.

### Basic reverse proxy

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

The `proxy_set_header` lines pass the real client IP and protocol to your backend — without these, your app sees nginx's IP, not the user's.

### With timeouts (important for slow APIs)

```nginx
location / {
    proxy_pass http://localhost:8000;
    proxy_read_timeout 60s;    # wait up to 60s for backend response
    proxy_connect_timeout 10s; # wait up to 10s to connect to backend
    proxy_send_timeout 60s;
}
```

### Load balancing across multiple instances

```nginx
upstream myapp {
    server localhost:8001;
    server localhost:8002;
    server localhost:8003;
    # least_conn;  # uncomment to use least-connections instead of round-robin
}

server {
    listen 80;
    location / {
        proxy_pass http://myapp;
    }
}
```

### WebSocket support

```nginx
location /ws/ {
    proxy_pass http://localhost:8000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
}
```

---

## SSL / HTTPS

### Let's Encrypt with Certbot (the standard way)

```bash
# install certbot
sudo apt install certbot python3-certbot-nginx

# get a certificate and auto-configure nginx
sudo certbot --nginx -d example.com -d www.example.com

# test auto-renewal
sudo certbot renew --dry-run
```

Certbot edits your nginx config automatically and sets up a cron job for renewal. Easy.

### What the SSL server block looks like after certbot

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # good SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;

    location / {
        proxy_pass http://localhost:8000;
    }
}

# redirect HTTP to HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

---

## Logs

| Command | What it does |
|---|---|
| `tail -f /var/log/nginx/access.log` | Stream live access logs — every request in real time |
| `tail -f /var/log/nginx/error.log` | Stream live error logs — first place to look when something breaks |
| `cat /var/log/nginx/error.log \| grep "crit"` | Filter for critical errors only |
| `sudo journalctl -u nginx -f` | Same as error log but through systemd's journal |
| `sudo journalctl -u nginx --since "1 hour ago"` | Errors from the last hour |
| `sudo nginx -t 2>&1` | Capture config test output — useful in deploy scripts |

### Custom log format

```nginx
http {
    log_format detailed '$remote_addr - $remote_user [$time_local] '
                        '"$request" $status $body_bytes_sent '
                        '"$http_referer" "$http_user_agent" '
                        'rt=$request_time uct=$upstream_connect_time urt=$upstream_response_time';

    access_log /var/log/nginx/access.log detailed;
}
```

`$request_time` and `$upstream_response_time` are especially useful — they tell you how long nginx took vs how long your backend took.

---

## Performance Tuning

Things worth setting once you're past the default config.

```nginx
# in nginx.conf, inside http {}

# number of worker processes — set to number of CPU cores
worker_processes auto;

# max connections per worker
events {
    worker_connections 1024;
}

http {
    # enable gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    gzip_min_length 1000;
    gzip_comp_level 6;

    # client body size limit (default is 1MB — increase for file uploads)
    client_max_body_size 20M;

    # keep connections alive longer to reduce TCP overhead
    keepalive_timeout 65;
    keepalive_requests 100;

    # sendfile — lets the OS handle file transfers directly (faster for static files)
    sendfile on;
    tcp_nopush on;

    # hide nginx version from response headers
    server_tokens off;
}
```

---

## Security Headers

Add these to your server block to improve security posture without much effort.

```nginx
server {
    # hide nginx version
    server_tokens off;

    # prevent clickjacking
    add_header X-Frame-Options "SAMEORIGIN" always;

    # prevent MIME type sniffing
    add_header X-Content-Type-Options "nosniff" always;

    # enable XSS protection in older browsers
    add_header X-XSS-Protection "1; mode=block" always;

    # force HTTPS for future visits (set a short max-age first, increase after testing)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # referrer policy
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
}
```

---

## Troubleshooting

The scenarios I've hit most often.

| Problem | What to check |
|---|---|
| `nginx -t` fails | Read the error carefully — it tells you the file and line number |
| 502 Bad Gateway | Your backend isn't running, or nginx can't reach it. Check `systemctl status yourapp` and the backend port |
| 504 Gateway Timeout | Backend is running but too slow. Increase `proxy_read_timeout` |
| 413 Request Entity Too Large | File upload exceeds `client_max_body_size`. Increase it in your server block |
| 403 Forbidden | File permission issue. Check `ls -la` on your web root — nginx user needs read access |
| Changes not applying | Did you run `nginx -t` and `systemctl reload nginx`? Also check you're editing the right file |
| Port 80 already in use | Something else is on port 80. Run `ss -tulnp \| grep :80` to find what |
| SSL cert expired | Run `sudo certbot renew` — or check if the cron job is actually running |

### Quick debug sequence

```bash
# 1. check nginx is running
sudo systemctl status nginx

# 2. test config
sudo nginx -t

# 3. check error logs
sudo tail -50 /var/log/nginx/error.log

# 4. check if backend is reachable
curl -I http://localhost:8000

# 5. reload after fixing
sudo nginx -t && sudo systemctl reload nginx
```

---

## Contributing

If something's wrong or missing — PRs welcome.

---

*Last updated: 2025*
