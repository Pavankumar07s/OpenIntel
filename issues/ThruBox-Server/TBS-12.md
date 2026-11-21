# [BUG]: Rate limiter always sends `Retry-After: 60`, browsers cannot read it, and the "sliding window" is a fixed window

### Bug Description

Four small problems in the rate limiter (`internal/middleware/ratelimit.go`) and the CORS middleware (`internal/middleware/cors.go`):

1. **`Retry-After` is always the full window.** `ratelimit.go:53` sends `int(rl.window.Seconds())`, which is always `60`, however much of the client's window is left. A client that is rejected 50 s into its window is told to wait 60 s, but it could retry 10 s later. Clients that honour the header, such as `RelayRateLimitError.retryAfter` in `@aossie-org/thrubox-client`, wait longer than they need to. The value is a safe upper bound, so this costs clients time but does not break anything.

2. **Browser JS cannot read `Retry-After` cross-origin.** `Retry-After` is not a CORS-safelisted response header, and the CORS middleware never sends `Access-Control-Expose-Headers`. The 429 does carry `Access-Control-Allow-Origin`, so the browser exposes the response, but `response.headers.get('Retry-After')` returns `null`. The client SDK then falls back to its hard-coded 60 (`src/client.ts:289-301`). This only affects deployments that set `security.allowed_origins` and serve browser apps from another origin.

3. **The comment says "sliding window counter", but `allow()` is a fixed window.** `ratelimit.go:19` describes a sliding window. `allow()` (`:66-85`) keeps one counter per IP and resets it once `window` has passed since the first request. A client can therefore use up its limit at the end of one window and the full limit again at the start of the next. That is up to `2 × rate_limit − 1` requests in a few seconds. A fixed window is a reasonable choice, but the comment (and the original PR description) should not call it a sliding window.

4. **Every rejected request writes a WARN line.** `ratelimit.go:56` logs `rate limit exceeded` once per 429. A client that keeps sending requests while limited adds one log line per request, and nothing caps that. The README ("Behind a Reverse Proxy") tells operators to use this line to check `trusted_proxies`, so it should stay, but once per IP per window is enough.

### Steps to Reproduce

1. Start the relay with a small limit and CORS enabled:
   ```bash
   RELAY_SERVER_PORT=3000 RELAY_SECURITY_RATE_LIMIT=3 \
   RELAY_SECURITY_ALLOWED_ORIGINS=https://app.example go run ./cmd/relay
   ```
2. Send requests with this timing (the first request starts the window):
   ```bash
   req() { curl -s -o /dev/null -D - -H 'Origin: https://app.example' \
     http://127.0.0.1:3000/api/messages/0xabc | tr -d '\r' \
     | grep -E '^HTTP|^Retry-After|^Access-Control-' | tr '\n' ' '; echo; }
   req            # t=0
   sleep 50
   req; req; req  # t=50: 2 allowed, the third is rejected
   sleep 10.5
   req; req; req; req  # t=60.5: new window
   for i in $(seq 20); do curl -s -o /dev/null http://127.0.0.1:3000/api/messages/0xabc; done
   ```
3. Count the `rate limit exceeded` lines in the server log.

**Expected:**
- The 429 at t=50 says `Retry-After: 10` (the time left in the window).
- Responses to an allowed origin include `Access-Control-Expose-Headers: Retry-After`.
- The code comment matches the algorithm.
- Repeated 429s from one IP do not each add a log line.

**Actual:**
- The 429 at t=50 says `Retry-After: 60`, but a request 10.5 s later succeeds.
- There is no `Access-Control-Expose-Headers`.
- 5 requests succeed within 10.6 s with `rate_limit: 3`.
- Every 429 logs a WARN line (22 of the 28 log lines).

### Logs and Screenshots

Output of a timed script that runs the same sequence against `main` (`f026f22`) and labels each request (A–H) with its elapsed time:

```
t=  0.02s A -> 200 Retry-After=<none> ACAO=https://app.example Expose-Headers=<none>
t= 50.04s B -> 200 Retry-After=<none> ACAO=https://app.example Expose-Headers=<none>
t= 50.05s C -> 200 Retry-After=<none> ACAO=https://app.example Expose-Headers=<none>
t= 50.07s D -> 429 Retry-After=60 ACAO=https://app.example Expose-Headers=<none>
t= 60.59s E -> 200 Retry-After=<none> ACAO=https://app.example Expose-Headers=<none>
t= 60.61s F -> 200 Retry-After=<none> ACAO=https://app.example Expose-Headers=<none>
t= 60.63s G -> 200 Retry-After=<none> ACAO=https://app.example Expose-Headers=<none>
t= 60.64s H -> 429 Retry-After=60 ACAO=https://app.example Expose-Headers=<none>
WARN 'rate limit exceeded' lines in server log: 22
total server log lines: 28
time=2026-11-20T00:25:08.392+05:30 level=WARN msg="rate limit exceeded" ip=127.0.0.1
time=2026-11-20T00:25:18.964+05:30 level=WARN msg="rate limit exceeded" ip=127.0.0.1
```

The code involved:

```go
// ratelimit.go:19
// RateLimiter provides IP-based rate limiting using a sliding window counter.

// ratelimit.go:52-56
if !rl.allow(ip) {
    retryAfter := int(rl.window.Seconds())
    w.Header().Set("Retry-After", strconv.Itoa(retryAfter))
    http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
    slog.Warn("rate limit exceeded", "ip", ip)
```

### Environment Details

- ThruBox-Server `main` @ `f026f22` (VERSION 1.0.0), built with `go build ./cmd/relay`
- Linux, curl, real binary on loopback
- Points 1 and 2 matter to browser apps that use `@aossie-org/thrubox-client` against a relay with `security.allowed_origins` set

### Impact

Low - Minor inconvenience

### Proposed fix

All changes are in `internal/middleware`, with no new dependencies:

- Have `allow()` also return the time left in the window. Send `Retry-After: ceil(windowStart + window − now)`, with a minimum of 1.
- In `cors.go`, set `Access-Control-Expose-Headers: Retry-After` on non-preflight responses to allowed origins.
- Change the `RateLimiter` comment to "fixed window per IP". Alternatively, switch to a sliding-window or token-bucket algorithm, if maintainers want to remove the boundary burst.
- Log the first rejection per IP per window, for example with a `warned` flag on `visitor` that is cleared when the window resets. The README's `trusted_proxies` check keeps working.
- Add tests:
  - `Retry-After` is below the window size partway through a window (use a short `window` in an internal test).
  - The expose header is present on allowed-origin responses.
  - The log line is written once per window.

A quick prototype of the first two points passes the existing tests. With it, a 429 sent 20 s into the window returns `Retry-After: 40` and `Access-Control-Expose-Headers: Retry-After`.
