# SSE + Basic Auth — EventSource Limitation & Token Workaround

**Skill:** `ec2-service-deploy` | **Pattern type:** Troubleshooting / integration

---

## The Problem

Browser's `EventSource` API **cannot send HTTP Basic Auth credentials** — no `Authorization` header mechanism exists in the SSE spec. When a live-updating endpoint is protected by nginx `auth_basic`, the HTML page loads fine but the EventSource connection gets a `401` and silently fails.

**Symptoms:**
- Page loads correctly (auth works for HTML)
- Status shows "Reconnecting..." indefinitely
- No SSE events arrive
- nginx access log: `GET /api/events HTTP/1.1" 499 0` (client closed connection before response)

## The Solution — Token Auth for SSE

Three-step pattern:

1. `GET /api/token` — returns a bearer token. Basic-auth gated, so regular `fetch()` succeeds (browsers send stored basic auth automatically on same-origin requests).
2. UI calls `fetch('/api/token')` first to get the token.
3. UI opens `EventSource('/api/events?token=<token>')` — nginx forwards query param to backend; backend validates.

```
Browser                 nginx                   API
  │                        │                      │
  ├─ GET /api/token ──────▶│ (basic auth OK)     │
  │◀─────────── {token} ───│                      │
  ├─ GET /api/events? ────▶│ (token in URL)       │
  │   token=xxx             │────────▶ [valid] ──▶ 200, SSE stream
```

## API Server (Node.js)

```javascript
const SSE_TOKEN = 'hermes-kanban-live-' + Math.random().toString(36).slice(2, 10);

// Token endpoint — basic auth gated
if (pathname === '/api/token') {
  return sendJSON(res, 200, { token: SSE_TOKEN });
}

// SSE endpoint — requires token query param
if (pathname === '/api/events') {
  const token = url.parse(req.url, true).query.token;
  if (token !== SSE_TOKEN) {
    res.writeHead(401, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: 'Invalid or missing SSE token' }));
    return;
  }
  // SSE setup...
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'X-Accel-Buffering': 'no',  // disable nginx buffering
  });
  const keepalive = setInterval(() => { try { res.write(': ping\n\n'); } catch { clearInterval(keepalive); } }, 30000);
  clients.add(res);
  req.on('close', () => { clearInterval(keepalive); clients.delete(res); });
}
```

## UI (Vanilla JS)

```javascript
let sseToken = null;
let es = null;

async function init() {
  await fetchTasks(); // render initial state

  // Get SSE token (browser sends basic auth automatically on same-origin fetch)
  try {
    const res = await fetch('/api/token');
    const { token } = await res.json();
    sseToken = token;
  } catch (e) {
    console.error('Failed to get SSE token:', e);
    setStatus('offline');
    return;
  }
  connectSSE();
}

function connectSSE() {
  if (!sseToken) return; // guard: don't open until token is ready
  if (es) es.close();
  setStatus('reconnecting');
  es = new EventSource(`/api/events?token=${encodeURIComponent(sseToken)}`);
  es.onopen = () => { setStatus('live'); reconnectDelay = 1000; };
  es.onmessage = (e) => { /* handle task events */ };
  es.onerror = () => {
    es.close(); es = null;
    setStatus('reconnecting');
    reconnectTimer = setTimeout(() => {
      reconnectDelay = Math.min(reconnectDelay * 2, 30000);
      connectSSE();
    }, reconnectDelay);
  };
}

init();
```

## Why Not WebSocket?

WebSocket can send auth headers via the HTTP upgrade request. But it requires a different protocol, bidirectional communication (overkill for one-way push), and a separate server implementation. Token-in-URL is a minimal change to an existing SSE setup.

## nginx

No changes needed — query params forward automatically. Keep standard SSE proxy settings:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:4444;
    proxy_http_version 1.1;
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 86400;
    proxy_send_timeout 86400;
}
```
