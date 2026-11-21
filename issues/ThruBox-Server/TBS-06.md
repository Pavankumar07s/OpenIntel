# [BUG]: `/health` counts against the per-IP rate limit (returns 429), and the image has no `HEALTHCHECK`

## Bug Description

`GET /health` is meant to be what uptime monitors, platform health checks and the SDK's `relay.health()` call. Right now it has three problems:

1. **It is rate limited like any other route.** In `cmd/relay/main.go:105-108` the rate limiter wraps the whole mux, `/health` included. `APIKeyAuth` skips `/health` (`internal/middleware/apikey.go:22`), but `RateLimiter.Middleware` (`internal/middleware/ratelimit.go:48-62`) does not. So every health check uses up one of the caller's `rate_limit` requests per minute. Once the caller's IP has used them all, `/health` itself returns `429 Too Many Requests`. A monitor or load balancer that shares an IP bucket with real traffic (for example an external uptime check through a proxy, or an app that calls `health()` and also sends and receives) will then report the relay as down.
2. **It always reports `ok`.** `HandleHealth` (`internal/handler/health.go:17-25`) has no access to the store. When storage writes fail (for example the disk or volume is full), `POST /api/messages` returns 500 but `/health` still returns `{"status":"ok"}`.
3. **The image has no health check.** Neither `Dockerfile` nor `docker-compose.yml` defines a `HEALTHCHECK`/`healthcheck`. `docker ps` never shows the container as healthy or unhealthy, and compose users cannot use `depends_on: condition: service_healthy` (for example for the `cloudflared` service in the README's tunnel setup).

## Steps to Reproduce

Build from `main` (`f026f22`):

```bash
go build -o relay-server ./cmd/relay
```

**A. `/health` is rate limited** (default `rate_limit: 30`):

```bash
RELAY_SERVER_PORT=3000 ./relay-server &
for i in $(seq 1 31); do printf '%s ' "$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:3000/health)"; done; echo
curl -si http://127.0.0.1:3000/health | head -3
```

The same happens if the 30 requests are API calls: after 29 `/health` calls and one `GET /api/messages/0xabc` (200), the next `/health` returns 429.

**B. `/health` says `ok` while writes fail.** This uses a file-size limit to simulate a full disk:

```bash
(ulimit -f 300; RELAY_SERVER_PORT=3001 RELAY_STORAGE_PATH=$PWD/t/relay.db ./relay-server) &
for c in A B; do
  python3 -c "import json,sys; print(json.dumps({'to':'0xabc','from':'0xdef','payload':sys.argv[1]*150000}))" $c > m$c.json
  curl -s -o /dev/null -w 'POST %{http_code}\n' -H 'Content-Type: application/json' --data-binary @m$c.json http://127.0.0.1:3001/api/messages
done
curl -s -o /dev/null -w 'GET %{http_code}\n' http://127.0.0.1:3001/api/messages/0xabc
curl -s -w ' %{http_code}\n' http://127.0.0.1:3001/health
```

**C.** Run `grep -in healthcheck Dockerfile docker-compose.yml`. It prints nothing. The `docker-publish.yml` smoke test curls `/health` from the runner, but the image itself has no health check.

**Expected:** `/health` is not affected by the rate limit, the image has a health check, and the docs say what `/health` actually checks.

**Actual:** see the logs below.

## Logs and Screenshots

A:
```
200 200 200 ... 200 (30 times) 429
HTTP/1.1 429 Too Many Requests
Content-Type: text/plain; charset=utf-8
Retry-After: 60
```

B:
```
POST 201
POST 500
GET 200
{"status":"ok","timestamp":"2026-11-19T18:49:12Z"} 200
```
Server log: `level=ERROR msg="failed to save message" error="saving message: disk I/O error: file too large"`

## Environment Details

- ThruBox-Server `main` @ `f026f22` (VERSION 1.0.0), default config (`rate_limit: 30`, `trusted_proxies: 0`)
- Linux 6.17, Go 1.24.6 (go.mod: 1.23.12), binary run directly on loopback
- `Dockerfile` / `docker-compose.yml` read from the same commit

## Impact

Low - Minor inconvenience

Health probes only share a bucket with real traffic in some setups, and a full disk is an operator problem that restarting will not fix. But in those setups the monitor reports a relay that is up as down, or reports one that cannot store messages as up.

## Proposed fix

- **Exempt `GET /health` from the rate limiter.** It does no database work, so leaving it unlimited is cheap. One way is the same path check `APIKeyAuth` already uses, inside `RateLimiter.Middleware`. Another is to route `/health` around the limiter in `main.go`. Add a test showing that `/health` still returns 200 after the limit is reached.
- **Add a `HEALTHCHECK` to the `Dockerfile`.** BusyBox `wget` is already in `alpine`. It should follow the same port order the server uses (`RELAY_SERVER_PORT`, then `PORT`, then 3000):
  ```dockerfile
  HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -q -O /dev/null "http://127.0.0.1:${RELAY_SERVER_PORT:-${PORT:-3000}}/health" || exit 1
  ```
  Note in the README that users who change the port only in a mounted `config.yaml` need to override the health check. Optionally add a matching `healthcheck:` to `docker-compose.yml`.
- **Storage check (open for discussion).** Keep `/health` as a liveness check and document that it does not check storage. A plain `db.PingContext` does not help here: with `mattn/go-sqlite3`, `Ping` only checks that the connection is open, and it returned `nil` (as did `SELECT 1`) right after a write failed with `disk I/O error: file too large`. A useful storage signal would need something else, for example recording the last write error and reporting it, or a separate readiness endpoint that tries a small write. Such an endpoint should not be both unauthenticated and exempt from the rate limit.
