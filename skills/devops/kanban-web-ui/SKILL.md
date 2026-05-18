---
name: kanban-web-ui
category: devops
description: >
  Build, deploy, fix, or extend a live Kanban board UI for tracking agent tasks;
  includes real-time polling UIs, incremental DOM, nginx plus Node API stacks.
trigger: >
  Build, deploy, fix, or extend a live Kanban board UI for tracking agent tasks.
  Also real-time polling UIs, incremental DOM updates, nginx plus Node API stacks,
  session-cookie auth for web dashboards.
---

# Kanban Web UI

Deployed Node.js + nginx + PostgreSQL Kanban board at `https://kanban.pistorius.live` (EC2, user: ubuntu).

## Stack
- **API**: Node.js HTTP server (`~/kanban/api/server.js`) — vanilla `http` module, no Express
- **DB**: PostgreSQL in Docker (`kanban-db` container, port 5433→5432 on host)
- **Proxy**: Nginx on `kanban.pistorius.live` (HTTPS, Let's Encrypt)
- **Auth**: Session cookies (7-day expiry, `httpOnly`, `sameSite`), creds at `~/.hermes/vault/kanban/credentials.json`
- **UI**: Static HTML in `~/kanban/ui/index.html` — 3-second HTTP polling (`GET /api/tasks`)
- **CLI**: `~/bin/kanban` — auto-login, cookie-jar, `create`/`move`/`list` commands

## Architecture
```
nginx (:443)
  ├── /api/*       → proxy_pass 127.0.0.1:4444 (Node API)
  ├── /login       → proxy_pass 127.0.0.1:4444
  ├── /logout      → proxy_pass 127.0.0.1:4444
  ├── /auth/*      → proxy_pass 127.0.0.1:4444
  └── /*           → static files /var/www/kanban/static/
```

## Nginx gotcha
`try_files $uri $uri/ /index.html` serves `index.html` **before** Node can intercept `/login` and `/logout`. Fix: proxy auth routes explicitly before static fallback, don't let `try_files` catch them.

## DB schema
- **Tables**: `tasks`, `users`, `sessions`
- **Reserved word**: `column` is reserved in Postgres → use `status` instead (API uses `column`, DB uses `status`, server maps between them)
- **Container**: `docker run -d --name kanban-db -p 5433:5432 -v ~/kanban/db/data:/var/lib/postgresql/data -e POSTGRES_USER=kanban -e POSTGRES_PASSWORD=<pass> -e POSTGRES_DB=kanban postgres:16`
- **Auth**: vault json has `{ "username": "warren", "password": "..." }` for app login. DB user is separate (`kanban`/`kanban`).
- **DB password**: stored at `~/.hermes/vault/kanban/db_password.txt` (read by server.js at startup)

## API server (PostgreSQL)
All endpoints use `pg.Pool`. The server reads DB password from `~/.hermes/vault/kanban/db_password.txt` at startup.

Key patterns:
- **Connection pool**: `new Pool({ host: '127.0.0.1', port: 5433, user: 'kanban', password: DB_PASS, database: 'kanban', max: 5 })`
- **Startup flow**: verify `pool.connect()` → `seedFromJson()` (one-time, migrates any remaining JSON data) → `server.listen()`
- **Column mapping**: DB column `status` ↔ API field `column` — `server.js` translates between them in all queries and responses
- **Sessions**: stored in `sessions` table, validated via `SELECT username, expires_at FROM sessions WHERE token = $1`
- **Users**: stored in `users` table with `password_hash` (bcrypt), `is_admin` flag
- **Seed function**: reads `tasks.json`/`users.json` on first boot → checks table empty → batch INSERT → idempotent via `ON CONFLICT DO NOTHING`

## Real-time polling (no flash)

**Pitfall**: `body.innerHTML = ...` destroys and recreates every DOM element on every poll cycle → full reflow → visible flash.

**Three-part fix:**
1. **Diff before render** — compare task IDs + timestamps in `fetchTasks()`. If nothing changed, update data but skip calling `render()` entirely.
2. **Incremental DOM** — in `render(previousTasks)`, track existing cards by their `id` attribute. Only create/remove/move cards that actually changed. Update existing cards in-place via `el.innerHTML = newContent.innerHTML` (not replacing the element).
3. **Animation on modifier class** — don't put `@keyframes` animation on the base `.card` class. Put it on `.card.new` applied only to newly inserted elements.

**Polling interval**: 3s (`setInterval(fetchTasks, 3000)`). SSE/EventSource was unreliable on mobile/carrier — dropped connections, backoff conflicts with nginx idle timeouts. HTTP polling is deterministic.

## Process management
No pm2 installed on this EC2. Run the Node API as a background process via `terminal(background=true)` or use a systemd unit.

## Task breakdown preference

Warren corrects broad, phase-level task breakdowns. Use **granular, atomic tasks** — each should be a single operation (one endpoint, one migration, one test). Show a structured list with clear detail, not "Phase 1: migrate everything".

## Agent integration
When working on tasks, log them to the board via `~/bin/kanban create ...` and move them through columns as work progresses. The board's purpose is visibility into agent activity.

## Key files
- `~/kanban/api/server.js` — main API + session middleware (PostgreSQL)
- `~/kanban/ui/index.html` — board UI + polling logic (incremental DOM)
- `~/kanban/api/package.json` — deps (bcryptjs, pg)
- `~/bin/kanban` — shell CLI wrapper
- `~/.hermes/vault/kanban/credentials.json` — app login creds
- `~/.hermes/vault/kanban/db_password.txt` — PostgreSQL password

## Quick commands
```bash
# Check DB
docker exec kanban-db psql -U kanban -d kanban -c "SELECT count(*) FROM tasks;"

# Check API health
curl -s http://127.0.0.1:4444/api/health

# Restart API (Hermes background process or systemd)
# See ec2-service-deploy skill for systemd pattern
```
