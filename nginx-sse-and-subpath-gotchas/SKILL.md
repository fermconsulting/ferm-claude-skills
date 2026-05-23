---
name: nginx-sse-and-subpath-gotchas
description: Use when serving a web app behind nginx / Nginx Proxy Manager (NPM) under a SUBPATH (e.g. example.com/myapp) or when the app uses Server-Sent Events / EventSource / streaming responses through a proxy. Fixes "connection lost" on streaming, 404 on api calls that drop the subpath prefix, blank pages behind a reverse proxy, and "my fix isn't showing" browser-cache traps. Apply when the user mentions SSE, EventSource, streaming, subpath, base href, proxy_buffering, or assets/api 404ing only behind the proxy.
---

# Nginx SSE + Subpath Gotchas

Two adjacent problems that bite every reverse-proxied app: **streaming through nginx**
and **running under a subpath**. Both are solved, both are non-obvious.

## Decision first: subpath vs subdomain

**Prefer a subdomain** (`myapp.example.com`). It needs zero code changes — the app
serves from `/` as it always did. Subpaths (`example.com/myapp`) require base-path
support in the app AND nginx rewrites, and they silently break absolute URLs. Only
choose a subpath if a subdomain is genuinely unavailable.

## Problem 1 — SSE / EventSource "connection lost" through nginx

nginx buffers responses by default. Buffering holds the streamed events instead of
flushing them, so the browser's `EventSource` sees nothing and the connection dies.

**Fix — in the NPM custom location "Advanced" box (or nginx `location` block):**

```nginx
proxy_pass http://myapp:8080;          # internal container port
proxy_http_version 1.1;
proxy_set_header Connection "";        # keep upstream alive for streaming
proxy_buffering off;                    # THE critical line for SSE
proxy_cache off;
chunked_transfer_encoding off;
proxy_read_timeout 3600s;              # long-running agents/streams
proxy_send_timeout 3600s;
add_header X-Accel-Buffering no;        # belt-and-braces: tell nginx not to buffer
```

- `proxy_buffering off` is the one that actually fixes SSE. Without it, nothing else matters.
- `X-Accel-Buffering` goes via `add_header` (response to client), NOT `proxy_set_header` (that sends it to the upstream — wrong direction).
- Long `proxy_read_timeout` prevents the proxy killing a slow agent mid-stream.

**Verify from the server** (bypasses browser cache, isolates proxy vs app):

```bash
# Direct to the app — does the backend stream at all?
curl -N "http://localhost:9090/api/ask?q=hello" | head
# Through the proxy — does streaming survive nginx?
curl -N "https://example.com/myapp/api/ask?q=hello" | head
```

If direct works but proxied hangs → it's the buffering config. If both hang → it's the backend.

## Problem 2 — Subpath breaks absolute URLs

An app written for `/` emits absolute URLs: `<script src="/static/app.js">`,
`new EventSource('/api/ask')`, `fetch('/api/...')`. Behind `/myapp`, the browser
requests `example.com/static/app.js` (no `/myapp`) → **404 from nginx**, not the app.

### Fix A — make the app base-path aware (the right fix)

**Static HTML/JS:** add a `<base>` tag and switch absolute → relative URLs.

```html
<head>
  <base href="/myapp/">           <!-- everything relative resolves under /myapp/ -->
  <meta charset="UTF-8">
  ...
```

```js
// BEFORE (absolute — ignores <base>, breaks under subpath):
const es = new EventSource('/api/ask?q=' + q);
// AFTER (relative — respects <base>):
const es = new EventSource('api/ask?q=' + q);
```

Key rule: **a leading `/` makes a URL absolute and bypasses `<base>`.** Drop the leading slash for `<base>` to take effect.

**Next.js:** set `basePath` + `assetPrefix` and use `output: "standalone"`:

```ts
// next.config.ts
const nextConfig = { basePath: "/myapp", assetPrefix: "/myapp", output: "standalone" };
```

**FastAPI:** set `root_path` so generated URLs/docs include the prefix:

```python
app = FastAPI(root_path="/myapp")
```

### Fix B — nginx rewrite (when you can't touch the app cleanly)

Strip the prefix before forwarding so the app receives `/` paths:

```nginx
location /myapp {
    rewrite ^/myapp/?(.*)$ /$1 break;   # /myapp/api/x -> /api/x
    proxy_pass http://myapp:8080;
    proxy_buffering off;                 # keep SSE rules from Problem 1
    proxy_http_version 1.1;
    proxy_set_header Host $host;
}
```

Combine with a `<base href="/myapp/">` so the browser re-prefixes outbound requests
and the rewrite strips it on the way in. Get exactly one of (base prefixes, rewrite
strips) on each leg — double-prefixing gives `/myapp/myapp/...`.

### Subpath redirect convenience

Make `/myapp` (no trailing slash) redirect to `/myapp/` so relative `<base>`
resolution is consistent:

```nginx
if ($request_uri = "/myapp") { return 301 /myapp/; }
```

## Problem 3 — "I deployed the fix but the browser still shows the old version"

After confirming the **server** serves the new file, a stale page is browser cache.

```bash
# Prove the server is correct (hashes must match):
docker exec myapp md5sum static/index.html
curl -s https://example.com/myapp/ | md5sum
```

If they match, it's the client. To rule out cache definitively: open an
**Incognito/Private window** (ignores all cache). Then for the normal window:
hard-reload (Cmd/Ctrl+Shift+R) or DevTools → Network → "Disable cache".

## Anti-patterns to refuse

1. **`proxy_set_header X-Accel-Buffering no`** — wrong direction; it must be `add_header` to the client.
2. **Stripping query params or clauses when "normalising" rewrites** — `/api/ask?q=x` must keep `?q=x`. Only strip the path prefix.
3. **Fighting a subpath with sub_filter to rewrite HTML on the fly** — fragile; fix the app's URLs or use a subdomain.
4. **Blaming the backend for an SSE hang without the two curl tests** — always isolate proxy vs app with direct-port vs through-proxy curls.
5. **Tuning down a timeout to "fix" a dropped stream** — raise `proxy_read_timeout`, don't lower it; the stream is long by design.

## Checklist

- [ ] Chose subdomain unless a subpath is unavoidable
- [ ] SSE location has `proxy_buffering off` + `proxy_http_version 1.1` + `Connection ""` + long read timeout
- [ ] `X-Accel-Buffering no` set via `add_header` (not `proxy_set_header`)
- [ ] Subpath: app emits relative URLs (no leading `/`) AND has `<base href="/myapp/">`, or framework `basePath`/`root_path` set
- [ ] `/myapp` → `/myapp/` redirect in place
- [ ] Two-curl test passes: direct streams AND proxied streams
- [ ] Stale-UI ruled out via md5sum match + incognito check
