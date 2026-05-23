---
name: deploy-docker-behind-npm
description: Use when deploying a Dockerized app (FastAPI, Next.js, Node, any web service) to a shared Linux server that already runs Nginx Proxy Manager (NPM) and other containers. Covers exposing the app at a domain/subpath with automatic Let's Encrypt SSL, the Docker-network connectivity that lets NPM reach the container by name, the internal-port vs host-port trap, cloning private repos on the server, and recovering data/env from a running container. Apply when the user mentions NPM, "fermconsulting.com", a Hetzner/Ubuntu box reached over VPN, "expose my app", "deploy to the server", or proxy 502/400/404 errors.
---

# Deploy a Docker App Behind Nginx Proxy Manager

Field-tested recipe for putting a containerised web app on a shared Ubuntu server
that already hosts NPM + other apps. Every step here corresponds to a real failure
we hit and fixed.

## Mental model — the 3 things that go wrong

1. **Network isolation** — NPM can't resolve your container by name unless they share a Docker network. Symptom: 502 Bad Gateway.
2. **Internal vs host port** — `docker ps` shows `0.0.0.0:9090->8080/tcp`. NPM (same Docker network) must target the **internal** port `8080`, NOT the host port `9090`. Symptom: connection refused / 502.
3. **Subpath asset/URL breakage** — see the companion skill `nginx-sse-and-subpath-gotchas`. Prefer a subdomain unless the app is built for a base path.

## Step 0 — Recon (read-only, never skip)

```bash
ssh root@<server-or-vpn-ip>          # often a VPN IP like 10.8.0.1

docker ps --format '{{.Names}}\t{{.Ports}}'   # what's running + port maps
docker network ls                              # find NPM's network(s)

# Which network is NPM on? (note its container name from docker ps)
docker inspect <npm-container> \
  --format '{{range $n,$c := .NetworkSettings.Networks}}{{$n}} {{end}}'

# NPM admin is usually on :81 (NOT the app ports). Confirm:
curl -s -o /dev/null -w "81=%{http_code}\n" http://localhost:81
```

**Identify before touching anything:** NPM container name, NPM network(s), the domain's existing proxy host, and a free host port (avoid ports already in `docker ps`).

## Step 1 — Get the code + data onto the server

Private repo cloning needs a **Personal Access Token** (GitHub killed password auth):

```bash
cd /opt
git clone https://<user>:<ghp_TOKEN>@github.com/<org>/<repo>.git myapp
cd myapp && git checkout <branch>
# Then scrub the token from shell history:
history -c
```

If data/secrets only live inside an already-running container, pull them out with `docker cp` rather than rebuilding blind:

```bash
docker cp <old-container>:/app/.env            ./env.docker
docker cp <old-container>:/data/analytics.duckdb ./data/analytics.duckdb
```

## Step 2 — Build (force no-cache when code "didn't update")

```bash
docker build --no-cache -t myapp:prod .
```

`--no-cache` matters: Docker layer caching silently reuses an old `COPY app.py`
step, so your new code never lands in the image. If a fix "isn't taking", this is
the first suspect.

## Step 3 — Run on the SAME network as NPM, no host port needed

```bash
docker run -d --name myapp \
  --network <npm-network> \
  --env-file ./env.docker \
  myapp:prod
```

- **No `-p` flag required** if only NPM needs to reach it — internal network traffic is enough and keeps the port off the public host.
- If the app also needs an old host port for direct access, add `-p 9090:8080` AND `docker network connect <npm-network> myapp`.

Verify NPM can resolve it by name:

```bash
docker exec <npm-container> getent hosts myapp
# → 172.x.x.x  myapp   ✅  (NPM has wget/ping rarely; getent always works)
```

## Step 4 — Configure the proxy host in NPM UI (`http://<vpn-ip>:81`)

**Subdomain (recommended, no code changes):**
1. Hosts → Proxy Hosts → Add Proxy Host
2. Domain: `myapp.fermconsulting.com`
3. Scheme `http`, Forward Hostname `myapp`, Forward Port **`8080`** (internal!)
4. Block Common Exploits ON, Websockets Support ON
5. SSL tab → Request new Let's Encrypt cert → Force SSL ON → HTTP/2 ON → accept terms

**Subpath under an existing domain** (`fermconsulting.com/myapp`): edit the existing
proxy host → Custom Locations → add `/myapp`. This needs rewrites + the app must
support a base path — see `nginx-sse-and-subpath-gotchas` before going this route.

## Step 5 — DNS for subdomains

Add an `A` record `myapp.fermconsulting.com → <server-public-ip>` at the DNS
provider. Confirm before requesting the cert (Let's Encrypt validates over HTTP):

```bash
dig myapp.fermconsulting.com +short    # must return the server's public IP
```

## Step 6 — Verify end-to-end

```bash
curl -sI https://myapp.fermconsulting.com/         # 200 (or 405 on a POST-only root = OK, route exists)
docker logs myapp --tail 30
docker exec <npm-container> tail -20 /data/logs/proxy-host-<id>_error.log
```

## Gotcha catalogue (each was a real bug)

| Symptom | Cause | Fix |
|---|---|---|
| 502 Bad Gateway | container not on NPM's network | `docker network connect <npm-network> myapp` |
| 502 / refused | NPM pointed at host port (9090) | use the **internal** port (8080) |
| 400 "Invalid HTTP request" | `include force-ssl.conf` duplicated inside a custom location | remove the duplicate; keep Force SSL only at host level |
| 404 from nginx (not the app) | custom `location myapp` missing leading slash | must be `/myapp` |
| `git clone` auth failed | GitHub password auth removed | use a Personal Access Token |
| code change "not taking" | Docker layer cache | `docker build --no-cache` |
| cert request fails | DNS not pointing at server yet, or Let's Encrypt rate limit | fix `A` record / wait out the rate-limit window |
| NPM admin not reachable | trying app port instead of `:81`, or `:81` only on VPN | use `:81` over the VPN |

## Safety rules

- **Never `docker stop`/`rm` the old container before** you've `docker cp`-ed its `.env` and data out. Losing the API key / DB mount path is the worst-case here.
- **Don't expose internal ports publicly** (`-p`) unless something outside Docker needs them. NPM-internal traffic over the shared network is enough.
- **Never paste a PAT into a shared chat/log.** If it leaks, revoke it at github.com/settings/tokens immediately and issue a new one.
- **One change at a time.** Add the network, test. Add the proxy host, test. Add SSL, test. Bundled changes make 502s impossible to bisect.

## Quick checklist

- [ ] Recon done: NPM container name, NPM network, free host port, existing domain host all identified
- [ ] Code cloned (PAT, history scrubbed) and/or data `docker cp`-ed out of old container
- [ ] Image built with `--no-cache`
- [ ] Container runs on NPM's network; `getent hosts myapp` resolves from inside NPM
- [ ] Proxy host points at the **internal** port
- [ ] SSL cert issued (DNS verified first for subdomains)
- [ ] `curl -sI https://…` returns 200/expected; logs clean
- [ ] Old container only removed AFTER the new one is confirmed working
