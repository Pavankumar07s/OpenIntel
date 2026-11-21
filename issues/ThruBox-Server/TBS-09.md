# [BUG]: Wrong `X-API-Key` guesses are never rate limited because `APIKeyAuth` runs before the rate limiter

## Bug Description

When `security.api_key` is set, requests with a wrong or missing `X-API-Key` are never counted by the rate limiter. The middleware chain in `cmd/relay/main.go:105-108` is built as:

```go
h = rateLimiter.Middleware(h)
h = middleware.APIKeyAuth(cfg.Security.APIKey)(h)
h = middleware.CORS(cfg.Security.AllowedOrigins, cfg.Security.APIKey != "")(h)
```

So the request order is CORS → `APIKeyAuth` → `RateLimiter` → router. `APIKeyAuth` returns its 401 before `RateLimiter.allow()` runs. One client IP can therefore try keys as fast as the server answers, no matter what `rate_limit` is. `Config.Validate` also accepts a key of any length.

There is one more effect. Once an IP has used up its budget with valid requests, a wrong key still gets `401` and the right key gets `429`. The two statuses differ, so the limiter does not hide whether a guess was right.

The README tells operators to use `security.api_key` to "Protect Private Relays" with a secret key, so this matters for that setup. The risk is only real when the operator picks a short or guessable key. A long random key cannot be guessed online even without throttling, so the severity is low. The repository's security remediation notes (`audit/sec_review_2026-09-23T10-33-16Z_remediations.md`, "Non-remediated findings") already list "Rate-limiting failed key checks and warning on short keys" as a follow-up. This issue tracks that follow-up.

## Steps to Reproduce

1. Build the relay: `go build -o relay ./cmd/relay`
2. Save this as `cfg.yaml`:
   ```yaml
   server: { port: 31270, host: "127.0.0.1" }
   storage: { driver: "sqlite", path: "./relay.db" }
   messages: { ttl_days: 7, max_payload_size: 524288 }
   security: { rate_limit: 5, trusted_proxies: 0, api_key: "relay2026", allowed_origins: [] }
   ```
3. Start it: `RELAY_CONFIG_PATH=./cfg.yaml ./relay`
4. Send 500 requests from one IP, each with a different wrong key:
   ```bash
   A=0x1111111111111111111111111111111111111111
   for i in $(seq 1 500); do
     curl -s -o /dev/null -w '%{http_code}\n' -H "X-API-Key: guess$i" http://127.0.0.1:31270/api/messages/$A
   done | sort | uniq -c
   ```
5. Then send 7 requests with the right key, then 3 more with wrong keys:
   ```bash
   for i in $(seq 1 7); do curl -s -o /dev/null -w '%{http_code} ' -H "X-API-Key: relay2026" http://127.0.0.1:31270/api/messages/$A; done; echo
   for i in 1 2 3; do curl -s -o /dev/null -w '%{http_code} ' -H "X-API-Key: nope$i" http://127.0.0.1:31270/api/messages/$A; done; echo
   ```

**Expected:** with `rate_limit: 5`, failed key checks count toward the per-IP limit (or toward a separate failed-auth limit), and requests after the limit get `429` whether the key is right or wrong.

**Actual:** all 500 wrong-key requests get `401` and none get `429`. Only requests with the right key are limited. After the limit is hit, wrong keys still get `401` while the right key gets `429`.

## Logs and Screenshots

Output on `main` (`f026f22`). The `/health` readiness probe used one of the 5 slots, so 4 valid requests got through:

```
== 500 wrong-key guesses (distinct keys) ==
    500 401
elapsed: 2.18 s              # single sequential curl loop
== 50 requests with no key ==
     50 401
== correct key, 7 requests in same window ==
200 200 200 200 429 429 429
== wrong key again after limiter tripped ==
401 401 401
```

With only the two `h = ...` lines for `RateLimiter` and `APIKeyAuth` swapped (CORS still outermost), the same 20 wrong-key requests give `4 × 401, 16 × 429`, and `go test ./...` still passes.

## Environment Details

- ThruBox-Server `main` at `f026f22` (`VERSION` 1.0.0)
- Built with Go 1.24.6 (`go.mod` says 1.23.12)
- Linux, server on loopback, `trusted_proxies: 0`

## Impact

Low - Minor inconvenience

## Proposed fix

- In `cmd/relay/main.go`, put the rate limiter before the API-key check: CORS → `RateLimiter` → `APIKeyAuth` → router. CORS stays outermost, so preflights are still answered without a key and are not counted. Update the order comment and the README architecture diagram.
- Trade-off: unauthenticated requests would then use the per-IP budget. Clients that share a bucket (same NAT, or a proxy deployment with `trusted_proxies` left at 0) could be limited by someone who does not have the key. If that is not acceptable, keep the current order and add a separate failed-auth counter per client IP inside `APIKeyAuth`, for example 10 failures per minute and then `429`, using the same `clientIP` logic.
- In `config.Validate` or at startup, log a warning when `security.api_key` is set but shorter than about 32 characters.
- Add a test for the chain order: wrong keys get `429` after the limit, and a preflight still gets `204`. The chain is built inline in `main()` and `cmd/relay` has no tests. One option is a small exported helper in `internal/middleware` that builds the chain. Note that the `.gitignore` entry `relay` also ignores new files under `cmd/relay/`.
