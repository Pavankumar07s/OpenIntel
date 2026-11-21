# Rate limiter can be bypassed by rotating IPv6 addresses within one /64

## Bug Description

The rate limiter keys its buckets on the full client address. `clientIP` (`internal/middleware/ratelimit.go:135-158`) returns the whole IP string, and `allow()` (`ratelimit.go:66-85`) uses it as the map key. For IPv6 that means one bucket per /128.

One IPv6 host usually controls far more than one address. ISPs and VPS providers commonly route a /64 or larger to each customer, and the host can open connections from any address in that range. If every request comes from a new address, every request gets a new bucket, so `security.rate_limit` never applies. The effect is the same as the `X-Forwarded-For` bypass fixed in #68, but here the client changes its real source address instead of a header.

The relay keys on the client's full IPv6 address in both supported setups:

- **Direct (`trusted_proxies: 0`).** The default `host: "0.0.0.0"` gives a dual-stack listener in Go, so IPv6 clients connect directly and `RemoteAddr` holds their full address. I checked this with the real binary (see below).
- **Behind a proxy (`trusted_proxies: 1`).** The proxy appends the client's full IPv6 address to `X-Forwarded-For`, and `clientIP` uses that address as the key.

A related problem in the same file: the `visitors` map has no size limit. Entries are only removed by `cleanup()`, which runs once a minute and drops entries whose window has ended. So the map holds every distinct source seen in the last 1 to 2 minutes. In a test with 1,000,000 distinct sources (Go 1.24.6):

- The heap grew by about 114 MB.
- After the window passed and `cleanup()` had removed every entry, about 53 MB was still allocated. Go maps do not shrink when entries are deleted.
- That `cleanup()` call held `rl.mu` for about 127 ms. Every request waits on `rl.mu` during that time.

Filling the map this far takes a million distinct source addresses within about 2 minutes. It mostly matters in combination with the IPv6 bypass above.

## Steps to Reproduce

1. Add this test to `internal/middleware/` and run `go test -run TestIPv6Slash64Bypass -v ./internal/middleware/`:

   ```go
   package middleware_test

   import (
   	"fmt"
   	"net/http"
   	"net/http/httptest"
   	"testing"

   	"github.com/AOSSIE-Org/ThruBox-Server/internal/middleware"
   )

   func TestIPv6Slash64Bypass(t *testing.T) {
   	ok := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {})
   	for _, proxies := range []int{0, 1} {
   		rl := middleware.NewRateLimiter(3, proxies)
   		h := rl.Middleware(ok)
   		got := map[int]int{}
   		for i := 1; i <= 500; i++ {
   			r := httptest.NewRequest("GET", "/health", nil)
   			addr := fmt.Sprintf("2001:db8:aa:cc::%x", i) // same /64, new interface ID each time
   			if proxies == 0 {
   				r.RemoteAddr = "[" + addr + "]:40000"
   			} else {
   				r.RemoteAddr = "10.0.0.1:443" // the trusted proxy
   				r.Header.Set("X-Forwarded-For", addr)
   			}
   			rec := httptest.NewRecorder()
   			h.ServeHTTP(rec, r)
   			got[rec.Code]++
   		}
   		rl.Stop()
   		t.Logf("trusted_proxies=%d rate_limit=3, 500 requests from one /64: %v", proxies, got)
   	}
   }
   ```

   Expected: at most 3 requests get `200` and the rest get `429`. Actual: all 500 get `200` in both modes.

2. To check with the real binary over real TCP, use a Linux host that has two SLAAC addresses in the same /64 (for example a stable address and a privacy/temporary address). Here `$A` and `$B` are those two addresses:

   ```bash
   go build -o relay ./cmd/relay
   RELAY_SERVER_PORT=3000 RELAY_SECURITY_RATE_LIMIT=3 ./relay &
   for src in "$A" "$B"; do
     for i in 1 2 3 4; do curl -s -o /dev/null -w "%{http_code} " --interface "$src" "http://[$B]:3000/health"; done; echo
   done
   ```

   Each address gets its own 3 requests:

   ```
   200 200 200 429
   200 200 200 429
   ```

   The log shows two separate buckets, one per address:

   ```
   level=WARN msg="rate limit exceeded" ip=<address A>
   level=WARN msg="rate limit exceeded" ip=<address B>
   ```

   With a whole /64 routed to the host, the same works across 2^64 addresses.

## Logs and Screenshots

```
trusted_proxies=0 rate_limit=3, 500 requests from one /64: map[200:500]
trusted_proxies=1 rate_limit=3, 500 requests from one /64: map[200:500]
```

Control: 10 requests from a single IPv6 address give `map[200:3 429:7]`, as expected.

Visitor map with 1,000,000 distinct sources: heap +113.8 MB while the entries are held. After the window passed and `cleanup()` ran, the map had 0 entries and the heap was still +53.2 MB. That `cleanup()` call held the mutex for 126.8 ms.

## Environment Details

- `main` at `f026f22` (version 1.0.0)
- Linux. Tests were run with Go 1.24.6. `go.mod` pins 1.23.12. The bypass does not depend on the Go version. The memory numbers were measured on 1.24.6 only.
- Default config apart from `rate_limit: 3`, and `trusted_proxies: 1` for the proxy case

## Impact

Medium - Feature works but has issues

## Proposed fix

- **Key IPv6 clients by prefix.** Before calling `allow()`, mask IPv6 addresses to their /64, for example with `ip.Mask(net.CIDRMask(64, 128))` or `netip.Prefix`, and use the prefix as the key. Leave IPv4 addresses, including IPv4-mapped IPv6 addresses, unchanged. Optionally make the prefix length configurable (`security.ipv6_prefix`, default 64), with validation in `config.Validate`. This uses only the standard library, so no new dependency is needed.
- **Release memory after a burst.** In `cleanup()`, copy the live entries into a new map instead of deleting from the old one, so the memory is freed. A hard cap on entries is a separate design choice: refusing new buckets when the map is full would also lock out legitimate new clients.
- **Tests.** Two addresses in the same /64 share a bucket. Different /64s do not. IPv4 keys are unchanged. The `X-Forwarded-For` path is masked too.
- **Docs.** Update the README and `config.yaml` comments from "per IP" to "per IPv4 address or IPv6 /64".

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
