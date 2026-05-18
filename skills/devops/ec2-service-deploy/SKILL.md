---
name: ec2-service-deploy
description: Deploy a Node.js web service to EC2 behind nginx with systemd. Covers API server setup, nginx reverse proxy with session-based auth, polling-based live updates, SSL via certbot, and systemd hardening. For Warren's EC2 (3.104.68.19) running Ubuntu.
version: 1.0.0
platforms: [linux]
metadata:
  hermes:
    tags: [ec2, nginx, systemd, deploy, nodejs, ssl, lets-encrypt]
    related_skills: [firebase-deploy, shopping-ai-deploy, shoppingAIv2-logs]
---

# EC2 Service Deploy — Node.js + nginx + systemd

Deploy any Node.js HTTP service to Warren's EC2 with nginx reverse proxy, basic auth, SSE support, and Let's Encrypt SSL.

## When to Use This Skill

- Building a web service (API, dashboard, UI) that runs on EC2
- Adding a subdomain vhost with nginx proxy
- Setting up a systemd service for a long-running process
- Configuring session auth for a web property behind nginx
- Getting SSL certs via certbot for a new subdomain
- Building a live-updating UI behind auth — prefer **polling (3s interval)** over SSE; polling is more reliable across mobile carriers and corporate proxies

---

## Architecture

```
Browser ──▶ nginx (443/80) ──▶ Node.js API (:4444 internal)
                     │
                     └── nginx proxies ALL requests to Node.js
                         (Node handles auth + serves UI)
```

Key pattern: **Route everything through Node.js, not just /api.** Nginx should be a dumb proxy — Node.js decides auth and what to serve. This avoids the common `try_files /index.html` serving the wrong page for routes like `/login`.

For live updates, prefer polling (fetch every 3s) over SSE. SSE is fragile on mobile networks and corporate proxies.

---

## Prerequisites

- DNS A record pointing subdomain to `3.104.68.19` (EC2 public IP)
- Service code lives in `~/kanban/` or `~/<service>/`
- Credentials stored in `~/.hermes/vault/<service>/`
- Node.js installed at `/home/ubuntu/.local/bin/node` (in PATH for user, but use absolute path in systemd)

---

## Step 1 — Prepare the Service

Place service code in `~/<service>/`:
```
~/kanban/
├── api/server.js      # Node.js API (listen on 127.0.0.1:4444)
├── ui/                # Static files (served by nginx, NOT Node)
├── tasks.json         # Data file (read/written by Node)
└── <service>.service  # systemd unit
```

**Critical:** Serve static files from nginx, not from Node.js `static/` handler. This avoids permission headaches — nginx runs as `www-data` and can't read home directories.

If you must serve static files from Node, put them in `/var/www/<service>/` and `chown www-data:www-data`.

---

## Step 2 — Session Auth (No htpasswd needed)

For session-based auth, credentials are validated by the Node.js API — no htpasswd required. Credentials are stored in `~/.hermes/vault/<service>/` as JSON env or a `.env` file.

Example server-side session validation:
```js
// In your Node.js API — validate credentials, set HttpOnly session cookie
app.post('/api/auth/login', (req, res) => {
  const { username, password } = req.body;
  if (validUser(username, password)) {
    res.setHeader('Set-Cookie', `session=${token}; Path=/; HttpOnly; SameSite=Lax; Expires=${exp}`);
    res.json({ ok: true, user: username });
  } else {
    res.status(401).json({ error: 'Invalid credentials' });
  }
});
```

---

## Step 3 — Process Management (Hermes background or systemd)

The EC2 doesn't have pm2 installed. Two options:

**Option A — Hermes background process** (recommended for dev):
Use `terminal(background=true)` to run the Node process. The process is tracked and can be polled/killed via session management.

**Option B — systemd** (for production):
Place unit at `/etc/systemd/system/<service>.service`:

```ini
[Unit]
Description=<Service Description>
After=network.target

[Service]
Type=simple
User=ubuntu
Environment="PATH=/home/ubuntu/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
WorkingDirectory=/home/ubuntu/<service>
ExecStart=/home/ubuntu/.local/bin/node /home/ubuntu/<service>/api/server.js
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

# Hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/ubuntu/<service>
```

Key points:
- Use **absolute path** to node (`/home/ubuntu/.local/bin/node`) — systemd's default PATH doesn't include `~/.local/bin`
- Set `Environment="PATH=..."` to ensure all subprocesses find node
- `ProtectHome=read-only` (not `=true`) — still restricts access but allows read from home dir
- `WorkingDirectory` set explicitly; if it fails, systemd drops to `/` which is fine but logs a warning
- `ReadWritePaths=/home/ubuntu/<service>` — allows writes to service directory

Install:
```bash
sudo cp ~/<service>/<service>.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable <service>
sudo systemctl start <service>
sudo systemctl status <service> --no-pager -l
```

**Port conflict (EADDRINUSE):** If the port is already in use, find and kill the old process:
```bash
sudo lsof -i :<PORT> -t | xargs -r sudo kill -9
sudo systemctl restart <service>
```

---

## Step 4 — nginx Vhost

HTTP-only config first (for certbot verification):

```nginx
server {
    listen 80;
    server_name <subdomain>.<domain>;

    # Redirect everything to HTTPS (certbot handles this automatically)
    location / {
        return 301 https://$host$request_uri;
    }
}
```

HTTPS config (after certbot) — route everything through Node.js for session auth:

```nginx
server {
    listen 443 ssl;
    server_name <subdomain>.<domain>;

    ssl_certificate /etc/letsencrypt/live/<subdomain>.<domain>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<subdomain>.<domain>/privkey.pem;

    # Route everything to Node.js — Node handles auth, serves UI/login
    location ~ ^/(api|health) {
        proxy_pass http://127.0.0.1:<INTERNAL_PORT>;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_cache off;
    }

    location / {
        proxy_pass http://127.0.0.1:<INTERNAL_PORT>;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**No basic auth in nginx.** Session auth is handled by Node.js. The regex `~ ^/(api|health)` ensures API routes are proxied; all other routes (including `/`, `/login`, `/logout`) also go to Node.js which enforces auth.

Install:
```bash
sudo cp <nginx.conf> /etc/nginx/sites-available/<subdomain>.<domain>
sudo ln -sf /etc/nginx/sites-available/<subdomain>.<domain> /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## Step 5 — SSL with Certbot

DNS must already point `<subdomain>.<domain>` to EC2 IP before running certbot.

```bash
sudo certbot --nginx -d <subdomain>.<domain>
```

Certbot auto-reads the nginx vhost config and adds SSL directives. If it fails with "could not validate DNS propagation", wait for DNS to propagate (5-30 min) and retry.

After certbot succeeds, nginx config is updated with SSL cert paths. No manual config needed.

---

## File Locations (Warren's EC2)

| Purpose | Path |
|---|---|
| Service code | `~/kanban/` |
| Static UI | `/var/www/kanban/` (chown www-data) |
| Systemd unit | `/etc/systemd/system/kanban-api.service` |
| nginx vhost | `/etc/nginx/sites-available/kanban.pistorius.live` |
| htpasswd | `/etc/nginx/.kanban.htpasswd` | (deprecated — session auth now used, credentials in bcrypt via users.json) |
| SSL certs | `/etc/letsencrypt/live/kanban.pistorius.live/` |
| API log | `journalctl -u kanban-api --no-pager -f` |

---

## Key Permissions + Patterns

| Resource | Problem | Solution |
|---|---|---|
| `__dirname` in Node.js | When Node.js is invoked from systemd with a relative `WorkingDirectory`, `__dirname` resolves to wherever systemd started, not the intended directory | Always use an **absolute path** for data dirs: `const BASE_DIR = '/home/ubuntu/<service>'` instead of `__dirname` |
| Session files | Sessions stored in `sessions.json` | Point to the right BASE_DIR; sessions persist across restarts |
| Node in `.local/bin` | systemd PATH doesn't include `~/.local/bin` | Use absolute path in ExecStart + PATH env var |
| `ProtectHome=true` | Blocks all home access even with `ReadWritePaths` | Use `ProtectHome=read-only` instead |
| Nginx proxying `/login` | `try_files $uri /index.html` serves board as login page | Route ALL requests to Node.js — let Node serve the login page and enforce auth |

**When using session-based auth + Node.js behind nginx:** do NOT use `try_files`. Route everything through Node.js via `proxy_pass`. Node handles `/login` rendering and auth redirects. Using `try_files` with a static root causes nginx to serve `index.html` for `/login` — the browser sees the board instead of the login page.

---

## Verification Checklist

After deploy:
- [ ] `curl -s http://127.0.0.1:<PORT>/health` → `{"status":"ok"}`
- [ ] `curl -s -X POST -H "Content-Type: application/json" -d '{"username":"<user>","password":"<pass>"}' http://127.0.0.1:<PORT>/api/auth/login` → sets session cookie
- [ ] `curl -sb <cookie-jar> http://127.0.0.1:<PORT>/api/tasks` → `{"tasks":[...]}`
- [ ] `curl -s http://127.0.0.1:<PORT>/` (no cookie) → 302 redirect to `/login`
- [ ] `sudo systemctl status <service>` → `active (running)`
- [ ] DNS resolves `<subdomain>.<domain>` to `3.104.68.19`
- [ ] `sudo certbot --nginx -d <subdomain>.<domain>` succeeds
- [ ] Browser loads `https://<subdomain>.<domain>` with valid SSL cert, gets redirected to login

---

## Related Skills

- `firebase-deploy` — Firebase-specific CI/CD deploys (different target, same discipline of non-interactive auth + env scoping)
- `shoppingAIv2-deploy` — shoppingAIv2 full-stack deploy (Firebase Functions + Hosting)
- `shoppingAIv2-logs` — log retrieval for Firebase services
