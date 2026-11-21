# [BUG]: Rate limiter ignores X-Forwarded-For entries that include a port (IP:port) and limits by the proxy's address instead

## Bug Description

With `security.trusted_proxies: 1`, `clientIP` in `internal/middleware/ratelimit.go` picks the right X-Forwarded-For entry and passes it straight to `net.ParseIP` (lines 145-149):

```go
if len(hops) >= trustedProxies {
    if ip := net.ParseIP(hops[len(hops)-trustedProxies]); ip != nil {
        return ip.String()
    }
}
```

`net.ParseIP` returns `nil` for an address that has a port, such as `203.0.113.7:51234` or `[2001:db8::1]:443`. The code then falls back to `RemoteAddr` (lines 152-157), which is the proxy. So every client behind a proxy that writes ports into X-Forwarded-For shares the proxy's rate-limit bucket, even with `trusted_proxies` set the way the README says.

Some proxies do write ports. Azure Application Gateway documents its X-Forwarded-For as "a comma-separated list of IP:port". The setups the README documents (the Nginx config with `$remote_addr`, Cloudflare Tunnel, Render, Railway, Fly.io) write plain IPs and are not affected.

The README's check (set `rate_limit` low and read the `ip` in the `rate limit exceeded` log line) shows the proxy's address here. Raising `trusted_proxies` does not help: with a single X-Forwarded-For entry the list is shorter than the setting, so the code still falls back to `RemoteAddr`. It would also let clients choose their own IP by sending their own X-Forwarded-For.

The fallback is the safe direction, because it never trusts a client value. This is a correctness problem, not a bypass.

## Steps to Reproduce

**Unit level.** Save as `internal/middleware/xff_port_test.go` and run `go test -run TestXFFWithPort -v ./internal/middleware/`:

```go
package middleware

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestXFFWithPort(t *testing.T) {
	forms := []struct {
		name string
		xff  func(i int) string
	}{
		{"plain", func(i int) string { return fmt.Sprintf("203.0.113.%d", i+1) }},
		{"ip:port", func(i int) string { return fmt.Sprintf("203.0.113.%d:5123%d", i+1, i) }},
		{"[v6]:port", func(i int) string { return fmt.Sprintf("[2001:db8::%d]:443", i+1) }},
	}
	for _, f := range forms {
		rl := NewRateLimiter(3, 1) // rate_limit=3, trusted_proxies=1
		h := rl.Middleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {}))
		var codes []int
		for i := 0; i < 5; i++ { // five different clients, one request each
			r := httptest.NewRequest("GET", "/api/messages/x", nil)
			r.RemoteAddr = "10.0.7.1:443" // the proxy
			r.Header.Set("X-Forwarded-For", f.xff(i))
			w := httptest.NewRecorder()
			h.ServeHTTP(w, r)
			codes = append(codes, w.Code)
		}
		rl.Stop()
		fmt.Printf("XFF form %-10s -> %v\n", f.name, codes)
	}
}
```

**Real binary.** Here curl from loopback plays the proxy:

```sh
go build -o relay ./cmd/relay
RELAY_SERVER_HOST=127.0.0.1 RELAY_SERVER_PORT=3000 RELAY_SECURITY_RATE_LIMIT=3 \
RELAY_SECURITY_TRUSTED_PROXIES=1 ./relay &
for i in 1 2 3 4 5; do
  curl -s -o /dev/null -w '%{http_code} ' -H "X-Forwarded-For: 203.0.113.$i:5123$i" \
    http://127.0.0.1:3000/api/messages/0x1111111111111111111111111111111111111111
done; echo
```

**Expected:** five different clients each get their own bucket, so every request returns 200, the same as with plain IPs.

**Actual:** the clients share the proxy's bucket. Requests 4 and 5 get 429.

## Logs and Screenshots

Unit test:

```
XFF form plain      -> [200 200 200 200 200]
XFF form ip:port    -> [200 200 200 429 429]
XFF form [v6]:port  -> [200 200 200 429 429]
```

Log lines printed during the test (the logged `ip` is the proxy, not a client):

```
WARN rate limit exceeded ip=10.0.7.1
```

Real binary: `200 200 200 429 429`, and the log shows `msg="rate limit exceeded" ip=127.0.0.1`.

## Environment Details

- ThruBox-Server `main` at `f026f22`
- Go 1.24.6, Linux
- `security.rate_limit: 3`, `security.trusted_proxies: 1`
- Affects deployments behind a proxy that writes `IP:port` into X-Forwarded-For (for example Azure Application Gateway)

## Impact

Low - Minor inconvenience

## Proposed fix

In `clientIP`, when `net.ParseIP` fails on the selected entry, try `net.SplitHostPort` and parse the host part. This also covers `[v6]:port`. Optionally, strip surrounding brackets for a bare `[v6]`. Only the entry at the trusted position is parsed, as today, so clients still cannot choose their own IP. Add `ip:port` and `[v6]:port` cases to `TestClientIP` in `internal/middleware/ratelimit_test.go`. Update the comment above `clientIP` ("or the entry is not an IP") to match.
