# ThruBox-Server audit: contribution opportunities

- Repo: `AOSSIE-Org/ThruBox-Server`, branch `main`, commit `f026f22` (2026-11-07), version file `1.0.0`
- Companion client read for protocol checks: `AOSSIE-Org/ThruBox-Client` `9afc707`. The client audit is in `ThruBox-Client.md` in this folder; overlapping protocol items are cross-referenced (TBC-xx / P-x).
- Audit date: 2026-11-19. Read-only on GitHub. Both working trees were left clean. All repro code ran against a scratch copy of the repo, never the checkout.

---

## 1. Summary

| Check | Result |
|---|---|
| `go build ./...` | OK |
| `go vet ./...` | OK, 0 findings |
| `go test -race -count=1 -cover ./...` | All 5 test packages pass (54 test funcs), no data races. Coverage per package: config 80.0%, handler 64.6%, middleware 88.5%, model 50.0%, store 72.7%, `cmd/relay` 0% (no tests). **Total 64.8%** of statements. 0% functions: `main`, `purgeLoop`, `runPurge`, `HandleHealth`, `CreateMessageRequest.Validate`, `RateLimiter.cleanup`. `APIKeyAuth` is at 61.5% and has no test file of its own |
| `staticcheck` (default checks) | 0 findings. `-checks all` gives only ST1000 (missing package comments, 9 files) |
| `golangci-lint` v2.15 (default linters) | **14 errcheck** (5 in `store/sqlite.go`, 1 each in `main.go`, `health.go`, `model/message.go`, 6 in tests). Extra linters (`gocritic`, `noctx`, `gosec`) add `exitAfterDefer` at `cmd/relay/main.go:144`, plus 7 non-context DB calls in `sqlite.go` |
| `gosec` | 11 issues, all LOW/MEDIUM: G304 (config path from env var, expected), G301 (DB dir `0755`), 5× G706 (log injection, false positives because `%q` is used), 4× G104 (unchecked errors) |
| `govulncheck` with the **shipped** toolchain (`GOTOOLCHAIN=go1.23.12`, same as `go.mod` and the Dockerfile) | **"Your code is affected by 32 vulnerabilities from the Go standard library"**, all reachable. They include GO-2026-6859 (*HTTP/1 server crash due to MaxBytesReader header race*, and `HandleCreate` uses `MaxBytesReader`), GO-2026-6617, 6613, 6612, 6611 and 6603 (net/http crash, desync and DoS), plus crypto/tls and x509 issues. The fixes landed in go1.24.8 through go1.26.11. No vulnerable third-party module code is called → **TBS-01** |
| `hadolint` 2.12 on the `Dockerfile` | 2× DL3018 warnings (unpinned `apk add`) |
| `docker build .` | **Not run locally**, on purpose: the machine was out of disk during the audit and the coordinator asked to skip it. The CI "Docker Publish" workflow builds and smoke-tests the same Dockerfile on every PR, and the last 4 runs (2026-11-09 to 11-16) were green |
| CI | **No workflow runs `go build`, `go vet`, `go test` or a linter** (see TBS-13). The 54 tests never run in CI |
| End-to-end runs | The real binary was run on loopback and checked with curl for the TTL purge, CORS, rate limiting, a 50 MB mailbox read and a slow client. Every server I started was stopped |
| GitHub dedup | 8 closed issues (#11, #12, #25, #26, #27, #32, #68, #69), 0 open. 5 open PRs, all Dependabot (#77 go-sqlite3, #78–#81 Actions). I also read `audit/sec_review_2026-09-23*.md` (the maintainers' own security review and remediation notes) |

---

## 2. Findings table (ranked by value)

| ID | Title (ready to file) | Type | Sev | Status | Effort | Dedup |
|---|---|---|---|---|---|---|
| TBS-01 | Upgrade Go toolchain from EOL 1.23.12 (32 reachable stdlib CVEs, incl. a `MaxBytesReader` server crash) and Alpine 3.20 | security | high | VERIFIED | S | PARTIALLY COVERED: Dependabot PRs #71 (Go 1.27.3) and #16 (Alpine 3.24) were auto-closed by the stale bot, and there is no issue |
| TBS-02 | `GET /api/messages/{address}` is unbounded: one unauthenticated request can allocate hundreds of MB, and big mailboxes become unreadable | security | high | VERIFIED | M | NEW (builds on #69) |
| TBS-03 | Validate `to`/`from`: unbounded addresses create `mailboxes` rows that TTL never purges, odd addresses cannot be read back, and `max_payload_size` limits the body, not the payload | bug | medium | VERIFIED | S | NEW |
| TBS-04 | Rate limiter can be bypassed by rotating IPv6 addresses inside one /64, and its visitor map grows without bound | security | medium | VERIFIED | S–M | NEW (related to closed #68) |
| TBS-05 | Config accepts values that break or silently weaken the relay (`max_payload_size: 0`, `rate_limit: 0`, typo'd keys such as `api-key`, invalid env vars, any `storage.driver`) | bug | medium | VERIFIED | S | NEW |
| TBS-06 | `/health` reports OK while the database is dead, is rate limited, and the image has no `HEALTHCHECK` | bug | medium | VERIFIED | S | NEW |
| TBS-07 | Wallet addresses are matched byte-for-byte: EIP-55 checksummed and lowercase forms are different mailboxes | bug (protocol) | medium | VERIFIED | M | NEW (= client P-6) |
| TBS-08 | Cross-origin "simple" POSTs from non-allowlisted origins are accepted and stored | security | low | VERIFIED | S | NEW (related to closed #32) |
| TBS-09 | Failed API-key attempts are never rate limited (auth runs before the limiter) | security | low | VERIFIED | S | PARTIALLY COVERED: listed as a non-remediated follow-up in `audit/sec_review_2026-09-23T10-33-16Z_remediations.md`, no issue |
| TBS-10 | `ttl_days: 0` stops the purge loop, so rows that already have an expiry are never deleted | bug | low | VERIFIED | S | NEW |
| TBS-11 | `createdAt`/`expiresAt` in the POST response differ from the stored values (nanoseconds vs seconds) | bug | low | VERIFIED | S | NEW (= client P-1) |
| TBS-12 | Rate limiter: `Retry-After` is always the full window, the "sliding window" is really a fixed window (2× burst), and every 429 writes a WARN log line | bug | low | VERIFIED | S | NEW (partly = client P-4) |
| TBS-13 | Add a Go CI workflow (build, vet, `test -race`, golangci-lint, govulncheck) | ci | high | VERIFIED | S | NEW (gap acknowledged in `BestPracticesChecklist.md`) |
| TBS-14 | `.gitignore` entry `relay` silently ignores every new file under `cmd/relay/` | ci | medium | VERIFIED | S | NEW |
| TBS-15 | `stale.yml` auto-closes Dependabot PRs, including the Go and Alpine security bumps | ci | medium | VERIFIED | S | NEW |
| TBS-16 | Add per-mailbox quotas (max messages/bytes per recipient) | feature | medium | VERIFIED (gap) | M | NEW |
| TBS-17 | Expose server version and limits (`GET /api/info` or `/health` fields, `--version`) | feature | low | VERIFIED (gap) | S–M | NEW |
| TBS-18 | Raise test coverage: `cmd/relay` (purge loop, middleware order), `/health`, `APIKeyAuth`, `Validate`, `HandleCreate` error paths | test | medium | VERIFIED | M | NEW |
| TBS-19 | Docs refresh: stale "no tests" claims, wrong Go version, payload-size wording, wrong code comments, `SECURITY.md` / private reporting | docs | low | VERIFIED | S | NEW |
| TBS-20 | Cleanup bundle: lint errors, `os.Exit` skipping defers, dead code, DB file permissions, redundant index, template leftovers | cleanup | low | VERIFIED | S each | NEW |

---

## 3. Findings

All repro tests were Go tests in a scratch copy of the repo (package `repro`, plus one internal test in `internal/middleware`). They used this harness, which copies the wiring in `cmd/relay/main.go:89-108` exactly:

```go
type opts struct{ rateLimit, trustedProxies, maxPayload, ttlDays int; apiKey string; origins []string }

func buildChain(t *testing.T, o opts) (http.Handler, *store.SQLiteStore, string) {
	dbPath := filepath.Join(t.TempDir(), "relay.db")
	db, _ := store.NewSQLiteStore(dbPath); t.Cleanup(func() { db.Close() })
	if o.maxPayload == 0 { o.maxPayload = 524288 }
	if o.rateLimit == 0 { o.rateLimit = 1 << 30 }
	mh := &handler.MessageHandler{Store: db, TTLDays: o.ttlDays, MaxPayloadSize: o.maxPayload}
	mux := http.NewServeMux()
	mux.HandleFunc("GET /health", handler.HandleHealth)
	mux.HandleFunc("POST /api/messages", mh.HandleCreate)
	mux.HandleFunc("GET /api/messages/{address}", mh.HandleGetByAddress)
	rl := middleware.NewRateLimiter(o.rateLimit, o.trustedProxies); t.Cleanup(rl.Stop)
	var h http.Handler = mux
	h = rl.Middleware(h)
	h = middleware.APIKeyAuth(o.apiKey)(h)
	h = middleware.CORS(o.origins, o.apiKey != "")(h)
	return h, db, dbPath
}
func do(h http.Handler, method, target, body string, hdr map[string]string) *httptest.ResponseRecorder { /* httptest request + headers */ }
```

---

### TBS-01: Upgrade Go toolchain from EOL 1.23.12 (32 reachable stdlib CVEs, incl. a `MaxBytesReader` server crash) and Alpine 3.20

- **Type:** security · **Severity:** high · **Status:** VERIFIED · **Effort:** S
- **Evidence**
  - `go.mod:3` sets `go 1.23.12`. `Dockerfile:2` uses `FROM golang:1.23.12-alpine` and `Dockerfile:18` uses `FROM alpine:3.20`. Go 1.23 has been out of support since Aug 2025. Alpine 3.20's support window ended in April 2026.
  - `GOTOOLCHAIN=go1.23.12 govulncheck ./...` reports *"Your code is affected by 32 vulnerabilities from the Go standard library"*, all at symbol level. Notable ones:
    - GO-2026-6859 *HTTP/1 server crash due to MaxBytesReader header race in net/http* (fixed in 1.26.11). `internal/handler/message.go:35` wraps every POST body in `http.MaxBytesReader`.
    - GO-2026-6617, 6613, 6612, 6611 and 6603: HTTP/2 crash, HTTP/1 desync, flow-control and CPU/memory exhaustion in net/http.
    - GO-2026-6608: MIME header memory-limit bypass in net/textproto.
    - Several crypto/tls and crypto/x509 issues.
  - Root cause of the drift: Dependabot opened Go bumps #15, #33, #40, #47, #65 and #71 (1.27.3) and an Alpine bump #16 (3.24). None was merged. #71 and #16 were closed by `github-actions[bot]` after the `Stale` label (see TBS-15).
- **Repro:** `GOTOOLCHAIN=go1.23.12 govulncheck ./...` (exit code 3, 32 vulns).
- **Fix:** set `go 1.27` / `toolchain go1.27.3` in `go.mod`, use `golang:1.27.3-alpine` and `alpine:3.24` in the `Dockerfile` (ideally pinned by digest), and update the prerequisites in README and CONTRIBUTING. Run the test suite. Also merge the go-sqlite3 bump in open PR #77 (it ships SQLite 3.54.1). Add `govulncheck` to CI (TBS-13) so this cannot drift silently again.
- **Dedup:** PARTIALLY COVERED. Dependabot PRs #71/#16 existed but were closed unmerged, and no issue tracks it.

---

### TBS-02: `GET /api/messages/{address}` is unbounded: one unauthenticated request can allocate hundreds of MB, and big mailboxes become unreadable

- **Type:** security (DoS) · **Severity:** high · **Status:** VERIFIED · **Effort:** M
- **Evidence**
  - Without `?limit` the store runs the query with no `LIMIT` (`internal/store/sqlite.go:252`). It loads every row into a slice (`:263`) and encodes the whole slice in one go (`internal/handler/message.go:145`).
  - The maximum `limit` is 1000 (`message.go:17`) and the default `max_payload_size` is 500 KB, so even one "legal" page can be about 500 MB.
  - Anyone can fill any mailbox: sending needs no auth beyond the optional shared key, and there is no per-mailbox quota (TBS-16). One IP gets 30 req/min × 500 KB = 15 MB/min, which is about 150 GB over the default 7-day TTL.
  - `http.Server{WriteTimeout: 15s}` (`cmd/relay/main.go:119`) cuts a large response off midway, after a `200` has already been sent. The client then gets truncated JSON. `response.json()` throws, the SDK wraps it as `RelayNetworkError` and retries the GET up to 3 times (`ThruBox-Client/src/client.ts:250,271,330`). Each retry costs the server the full price again. The client's 10 s timeout (`client.ts:18`) is shorter than the server's 15 s, so on slow links the client aborts and retries even earlier.
- **Repro** (all three VERIFIED):
  ```go
  // 200 messages of ~500 KB to one address, then one GET
  for i := 0; i < 200; i++ { do(h, "POST", "/api/messages", fmt.Sprintf(`{"to":"0xvictim","from":"0xattacker%d","payload":"%s"}`, i, payload), nil) }
  rec := do(h, "GET", "/api/messages/0xvictim", "", nil)
  ```
  - Output: `one unauthenticated GET without ?limit: status=200 body=102.4 MB, allocated=474.4 MB, took 446ms`. With `?limit=1000` it is the same: `body=102.4 MB, allocated=205.9 MB`.
  - Real binary, 100 × 500 KB (50 MB mailbox): server RSS went from 23 MB to **201 MB** during one GET.
  - Real binary, slow client: `curl --limit-rate 1M .../api/messages/0xvictim` gives `curl: (18) transfer closed with outstanding read data remaining — http=200 got 17961921 of 51199686 bytes in 17.1s`. The server logs `failed to encode response ... i/o timeout`. A legitimate user on a slow link can never read their own mailbox.
- **Fix:**
  1. Apply a default page size when `limit` is omitted (e.g. 100) and a per-page **byte budget** (e.g. 4–8 MB, always at least one message).
  2. Signal more data. Either a `Link`/`X-Next-After` header (backward compatible with the bare-array body), or a new `?envelope=1` that returns `{messages, nextAfter, hasMore}`.
  3. Stream rows to the encoder instead of collecting them into a slice.
  4. Coordinate with the client so `poll()` drains pages (client finding TBC-09). Mention the behaviour change in CHANGELOG: clients that relied on "no limit = everything" must page.
- **Dedup:** NEW. #69 added the optional `limit`, but nothing bounds the default.

---

### TBS-03: Validate `to`/`from`: unbounded addresses create `mailboxes` rows that TTL never purges, odd addresses cannot be read back, and `max_payload_size` limits the body, not the payload

- **Type:** bug · **Severity:** medium · **Status:** VERIFIED · **Effort:** S
- **Evidence**
  - `CreateMessageRequest.Validate` (`internal/model/message.go:47-58`) only checks for empty strings. There is no length limit, no charset check and no trimming.
  - `SaveMessage` upserts a row into `mailboxes(to_addr, last_seq)` (`sqlite.go:206`). `PurgeExpired` (`sqlite.go:284`) only deletes from `messages`, never from `mailboxes`. So every distinct `to` stays on disk forever, and an "address" can be about 500 KB long.
  - `"."` and `".."` are accepted on POST, but Go's mux cleans the path on GET, so those mailboxes cannot be read. Whitespace-only and newline-containing fields are accepted too.
  - `max_payload_size` is applied to the whole body (`message.go:35`), but README and config describe it as "Max payload". The client report reached the same conclusion (P-2).
- **Repro (VERIFIED):**
  ```go
  bigTo := strings.Repeat("Z", 400*1024)
  for i := 0; i < 5; i++ { /* SaveMessage(to=bigTo+i, ExpiresAt=1h ago) */ }
  db.PurgeExpired(ctx)
  // SELECT COUNT(*) FROM messages; SELECT COUNT(*), SUM(LENGTH(to_addr)) FROM mailboxes
  ```
  - Output: `after purge: purged=5, messages left=0, mailboxes rows left=5 holding 2000 KB of to_addr`.
  - Address edge cases through the real mux: `to=".."` gives `POST 201, GET /api/messages/.. -> 301 Moved Permanently` (unreadable). `to=" 0xabc "` and `to="0xabc\n"` give `POST 201`. `{"to":"   ","from":" ","payload":" "}` gives `201`.
  - Payload limit with `max_payload_size=1000`: a payload of exactly 1000 bytes returns `413 payload too large`, and a payload of 970 bytes also returns `413`.
- **Fix:**
  - Add `MaxAddressLen` (e.g. 256 bytes) and reject control characters, whitespace, and `.`/`..`. Return `400` with a clear message.
  - Either enforce `len(req.Payload) <= MaxPayloadSize` after decoding, with the body cap set to `MaxPayloadSize + 2*MaxAddressLen + 1 KiB`, or rename the setting and docs to "max request body".
  - Optionally prune `mailboxes` rows whose mailbox has had no messages for N×TTL. This needs a maintainer decision, because deleting a counter restarts `seq` at 1 and breaks clients that saved a cursor. Bounding the address length is the safe part.
  - Add tests.
- **Dedup:** NEW.

---

### TBS-04: Rate limiter can be bypassed by rotating IPv6 addresses inside one /64, and its visitor map grows without bound

- **Type:** security · **Severity:** medium · **Status:** VERIFIED · **Effort:** S–M
- **Evidence**
  - `clientIP` (`internal/middleware/ratelimit.go:135-158`) keys buckets by the full address.
  - Every IPv6 subscriber normally gets at least a /64 (2^64 addresses), so per-/128 limiting does not limit anyone.
  - `rl.visitors` has no size cap (`ratelimit.go:75`). Cleanup only removes entries older than one window, so a flood of distinct sources inflates memory for 1–2 minutes at a time.
- **Repro (VERIFIED):**
  ```go
  rl := middleware.NewRateLimiter(3, 0)
  for i := 0; i < 1000; i++ { r.RemoteAddr = fmt.Sprintf("[2001:db8:1:2::%x]:5555", i+1); h.ServeHTTP(rec, r) } // one /64
  ```
  - Output: `rate_limit=3, 1000 requests from one IPv6 /64 (rotating interface id) -> map[200:1000]`.
  - A second test (1,000,000 distinct sources within one window) gave `limiter retains ~120 MB live heap`.
- **Fix:** normalise IPv6 to a configurable prefix before using it as the key (`security.ipv6_prefix`, default 64, using `net.IPNet` masking). Cap the map, for example with a max-entries LRU or by refusing new buckets under pressure. Optionally switch to a token bucket (`golang.org/x/time/rate` would be a new dependency, which needs discussion per AGENTS.md, so a tiny in-house token bucket is the safer choice). Add tests.
- **Dedup:** NEW. #68 (X-Forwarded-For spoofing, closed) fixed a different bypass.

---

### TBS-05: Config accepts values that break or silently weaken the relay

- **Type:** bug · **Severity:** medium · **Status:** VERIFIED · **Effort:** S
- **Evidence:** `internal/config/config.go`:
  - `Validate` allows `max_payload_size: 0` (`:211`) and `rate_limit: 0` (`:214`).
  - `yaml.Unmarshal` is not strict (`:106`), so unknown keys are ignored.
  - Invalid env values only `log.Printf("... using default")` and keep going (`:132-176`). The message is also wrong: the value kept is the YAML value, not the default. These are the only `log` (not `slog`) lines in the program.
  - `storage.driver` / `RELAY_STORAGE_DRIVER` (`:140`) accepts anything and is logged as if it were in effect (`main.go:54`), but SQLite is always used.
- **Repro (VERIFIED):**
  - `RELAY_MESSAGES_MAX_PAYLOAD_SIZE=0` loads fine, and then *every* POST fails: `config accepted max_payload_size=0; tiny POST -> 413 "payload too large\n"`.
  - `RELAY_SECURITY_RATE_LIMIT=0` loads fine and means "1 request per minute", not "off" and not "block": `three requests -> [200 429 429]`. The cause is `allow()` returning `true` for the first request in a window before checking the limit (`ratelimit.go:79`).
  - A YAML file with `security: {ratelimit: 5, api-key: topsecret}`, `storage: {driver: postgres}` and env `RELAY_SECURITY_TRUSTED_PROXIES=one` gave: `rate_limit=30 (wanted 5), api_key="" (wanted topsecret -> relay is OPEN), driver="postgres" (sqlite is used anyway), trusted_proxies=0`. A one-character typo turns a private relay into a public one, and nothing at startup says so.
- **Fix:**
  - Require `max_payload_size >= 1` and `rate_limit >= 1`, or define 0 explicitly (e.g. `rate_limit: 0` = disabled) and document it.
  - Use `yaml.NewDecoder(...).KnownFields(true)`.
  - Make invalid env values a startup error, or at least an `slog.Warn` that names the value actually used.
  - Reject `storage.driver` values other than `sqlite`.
  - Add table tests.
- **Dedup:** NEW.

---

### TBS-06: `/health` reports OK while the database is dead, is rate limited, and the image has no `HEALTHCHECK`

- **Type:** bug · **Severity:** medium · **Status:** VERIFIED · **Effort:** S
- **Evidence**
  - `HandleHealth` (`internal/handler/health.go:17-25`) has no access to the store and always returns `ok`.
  - It sits behind the rate limiter (`main.go:106`). Only `APIKeyAuth` exempts it (`apikey.go:22`).
  - `Dockerfile` and `docker-compose.yml` have no `HEALTHCHECK`/`healthcheck`.
  - Behind a proxy with the default `trusted_proxies: 0`, users and load-balancer probes share one bucket. Under load the probe gets 429, the orchestrator marks the container unhealthy and restarts it.
- **Repro (VERIFIED):**
  - After closing the DB: `/health -> 200 {"status":"ok",...}; POST -> 500; GET -> 500`.
  - Real binary with `RELAY_SECURITY_RATE_LIMIT=8` (after 5 other requests): `/health` ×5 gives `200 200 200 429 429`. In the harness, `rate_limit=2` gives `/health x4 -> [200 200 429 429]`.
- **Fix:** keep `/health` as liveness, but exempt it (and a new `/readyz`) from the rate limiter. Make readiness run `db.PingContext` with a ~1 s timeout and return `503` on failure, which needs a `Ping(ctx)` method on `store.Store`. Add `HEALTHCHECK CMD wget -qO- http://127.0.0.1:${PORT:-3000}/health || exit 1` (BusyBox wget ships with Alpine) and a compose `healthcheck`. Add tests.
- **Dedup:** NEW.

---

### TBS-07: Wallet addresses are matched byte-for-byte: EIP-55 checksummed and lowercase forms are different mailboxes

- **Type:** bug (protocol) · **Severity:** medium · **Status:** VERIFIED · **Effort:** M
- **Evidence**
  - Store queries use `WHERE to_addr = ?` (`sqlite.go:249`), and the client passes addresses through unchanged (`client.ts:115`, `JSON.stringify(params)` at `:90`).
  - Wallet libraries disagree on casing (ethers returns checksummed, many backends lowercase). A sender writing to `0x5aAe…` and a recipient polling `0x5aae…` never meet, and neither side sees an error.
- **Repro (VERIFIED):** POST to `0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed`, then GET with the lowercase form, gives `"[]"`. The checksummed GET returns the message. The client audit confirmed the same (P-6).
- **Fix (needs a maintainer decision, so file it as a proposal):**
  - **Option A:** the server canonicalises addresses matching `^0x[0-9a-fA-F]{40}$` to lowercase on write and read. Other chains such as base58 Solana are case-sensitive, so the rule applies only to that pattern. This needs a one-off migration that merges `mailboxes` counters and renumbers `seq`, which is delicate, so ship it behind `messages.normalize_evm_addresses`.
  - **Option B:** declare addresses opaque, exact-match keys in both READMEs, and have the SDK offer an `addressNormalizer`.
  - Either way, add a test.
- **Dedup:** NEW (client report P-6 recommends docs at least).

---

### TBS-08: Cross-origin "simple" POSTs from non-allowlisted origins are accepted and stored

- **Type:** security · **Severity:** low · **Status:** VERIFIED · **Effort:** S
- **Evidence**
  - The CORS middleware deliberately serves non-preflight requests from unlisted origins (`internal/middleware/cors.go:80-83`).
  - `HandleCreate` never checks `Content-Type`.
  - A browser therefore sends `POST` with `Content-Type: text/plain` cross-origin **without a preflight**. Any web page can make its visitors' browsers write into an open relay: spam a victim's mailbox using thousands of visitor IPs to dodge the per-IP limiter (TBS-04), or reach relays on `localhost`/LAN that were never meant to be public.
  - With `api_key` set this is blocked, because the browser cannot add `X-API-Key` without a preflight.
- **Repro (VERIFIED, real binary):**
  ```
  curl -i -X POST :3918/api/messages -H 'Origin: https://evil.example' -H 'Content-Type: text/plain' \
       -d '{"to":"0xVictim","from":"0xForged","payload":"x"}'
  HTTP/1.1 201 Created            (no Access-Control-Allow-Origin, but the message is stored)
  ```
  The harness gives the same result: `-> 201, ACAO="", stored=1`. A `application/x-www-form-urlencoded` POST is also accepted (`201`).
- **Fix:** require `Content-Type: application/json` (media type parsed with `mime.ParseMediaType`) on `POST /api/messages` and return `415` otherwise. This forces a preflight for every cross-origin write. Optionally, when an allowlist is configured, answer `403` to *any* request whose `Origin` is present and not allowed. The official SDK already sends `application/json` (`client.ts:243`), so it keeps working. Add tests.
- **Dedup:** NEW (related to closed #32, which added CORS).

---

### TBS-09: Failed API-key attempts are never rate limited (auth runs before the limiter)

- **Type:** security · **Severity:** low · **Status:** VERIFIED · **Effort:** S
- **Evidence**
  - Chain order: CORS, then `APIKeyAuth`, then `RateLimiter` (`cmd/relay/main.go:105-108`). A wrong key gets its 401 before `allow()` runs.
  - There is no minimum key length.
  - The maintainers' remediation notes accept this for now ("Rate-limiting failed key checks and warning on short keys are left as a follow-up"), on the grounds that the key is often shipped in browser bundles.
- **Repro (VERIFIED):** `rate_limit=5`: 1000 wrong-key requests from one IP give `map[401:1000]`, and the correct key afterwards gives `200`.
- **Fix:** order the chain as CORS → RateLimiter → APIKeyAuth, which keeps preflights answered first. Alternatively count only failures, e.g. 10/min/IP with a 401 → 429 escalation. Warn at startup if `len(api_key) < 32`. Update the test `TestCORS_PreflightSurvivesAPIKeyAuth` / add a chain test (see TBS-18).
- **Dedup:** PARTIALLY COVERED. It is acknowledged in `audit/sec_review_2026-09-23T10-33-16Z_remediations.md` ("Non-remediated findings"), but no GitHub issue exists. It is a good small follow-up to file.

---

### TBS-10: `ttl_days: 0` stops the purge loop, so rows that already have an expiry are never deleted

- **Type:** bug · **Severity:** low · **Status:** VERIFIED · **Effort:** S
- **Evidence**
  - `cmd/relay/main.go:75` starts `purgeLoop` only when `TTLDays > 0`. Messages written while TTL was >0 keep their `expires_at`.
  - After the operator switches to `0` ("keep forever"), those rows are hidden from reads (`sqlite.go:249` filters them), but they stay on disk forever.
- **Repro (VERIFIED, real binary):** I seeded a DB with one message that expired yesterday and ran the server for 1.5 s with each setting:
  - `RELAY_MESSAGES_TTL_DAYS=0`: rows before 1, `GET -> []`, rows after **1**.
  - `RELAY_MESSAGES_TTL_DAYS=7`: rows before 1, rows after **0**.
- **Fix:** always run `purgeLoop`. It only deletes rows with a non-NULL past `expires_at`, so it is a no-op in pure forever mode. Keep the warning log. Extract it so it can be tested (needs TBS-14 first, because new files in `cmd/relay/` are git-ignored).
- **Dedup:** NEW.

---

### TBS-11: `createdAt`/`expiresAt` in the POST response differ from the stored values

- **Type:** bug · **Severity:** low · **Status:** VERIFIED · **Effort:** S
- **Evidence**
  - `HandleCreate` builds `now := time.Now().UTC()` (`message.go:59`) and encodes that in-memory value (`:92`).
  - The store persists `Format(time.RFC3339)`, which has second precision (`sqlite.go:218,231`).
  - Reads and idempotent retries therefore return a different timestamp for the same message.
- **Repro (VERIFIED):**
  ```
  POST 201 createdAt=2026-11-19T17:45:59.315655718Z expiresAt=2026-11-26T17:45:59.315655718Z
  GET      createdAt=2026-11-19T17:45:59Z           expiresAt=2026-11-26T17:45:59Z
  retry 200 createdAt=2026-11-19T17:45:59Z
  ```
- **Fix:** `now := time.Now().UTC().Truncate(time.Second)`. Do **not** switch the store to `RFC3339Nano`: it trims trailing zeros, so the fixed-width string comparisons used for expiry (`expires_at >= ?`) would stop being correct. If sub-second precision is wanted, use a fixed-width layout or integer Unix milliseconds. Add a test.
- **Dedup:** NEW (= client P-1).

---

### TBS-12: Rate limiter: `Retry-After` is always the full window, the "sliding window" is a fixed window, and every 429 logs

- **Type:** bug · **Severity:** low · **Status:** VERIFIED · **Effort:** S
- **Evidence:** `ratelimit.go`:
  - `:53` always sends `int(rl.window.Seconds())`, which is 60, no matter how much of the window remains.
  - `:19` claims a "sliding window counter", but `allow()` is a fixed window that resets per IP.
  - `:56` writes one `slog.Warn` per rejected request, so an attacker controls the log volume.
  - The CORS middleware never sets `Access-Control-Expose-Headers: Retry-After`, so browser JS cannot read the header at all (client P-4).
- **Repro (VERIFIED):** an internal test built the limiter with a 4 s window: a request 3 s into the window gives `429 Retry-After=4 (real reset in ~1s)`, and 1.1 s later the next request gives `200`. With limit 30, a fixed-window burst let `59 requests through within 25 ms straddling the window boundary`. The harness shows the 429 cross-origin as `Retry-After="60" ACAO="https://app.example" Access-Control-Expose-Headers=""`.
- **Fix:** compute `ceil(windowStart + window - now)`. Either fix the comment or implement a real sliding window / token bucket (could be combined with TBS-04). Log only the first rejection per IP per window. Add `Access-Control-Expose-Headers: Retry-After` to allowed CORS responses.
- **Dedup:** NEW.

---

### TBS-13: Add a Go CI workflow (build, vet, `test -race`, golangci-lint, govulncheck)

- **Type:** ci · **Severity:** high · **Status:** VERIFIED · **Effort:** S
- **Evidence:**
  - `grep -rn "go test\|go vet\|golangci" .github/` finds nothing; the only `setup-go` is in the manual GoReleaser workflow.
  - The 54 existing tests, `go vet` and any lint never run on PRs. `docker-publish.yml` builds the image and checks `/health`, nothing else.
  - `BestPracticesChecklist.md` lists `warnings`/`test_policy` as unmet for this reason.
  - Running the tools locally turned up 14 errcheck findings, `exitAfterDefer`, and 32 stdlib vulns (TBS-01).
- **Fix:**
  - Add `.github/workflows/go-ci.yml` on `pull_request` and `push` to main, with these steps: `actions/setup-go` using `go-version-file: go.mod`; `go build ./...`; `go vet ./...`; `go test -race -coverprofile=cover.out ./...`, uploading the profile; `golangci/golangci-lint-action` with a committed `.golangci.yml` (errcheck, govet, staticcheck, gosec, gocritic, noctx); and `golang/govulncheck-action`.
  - Pin actions by SHA like the other workflows, and add `step-security/harden-runner`.
  - Fix or justify the 14 errcheck hits in the same PR, or a follow-up.
  - Then tick the checklist items.
- **Dedup:** NEW. The gap is stated in the checklist, but no issue exists.

---

### TBS-14: `.gitignore` entry `relay` silently ignores every new file under `cmd/relay/`

- **Type:** ci · **Severity:** medium · **Status:** VERIFIED · **Effort:** S
- **Evidence:** `.gitignore:34` contains `relay`. That pattern has no slash, so it matches the `cmd/relay/` directory, and only the already-tracked `main.go` escapes it.
- **Repro (VERIFIED, read-only):**
  ```
  $ git check-ignore -v cmd/relay/main_test.go cmd/relay/purge.go
  .gitignore:34:relay	cmd/relay/main_test.go
  .gitignore:34:relay	cmd/relay/purge.go
  ```
  `git add .` silently skips such files, and `git add cmd/relay/main_test.go` refuses. This is a plausible reason `cmd/relay` has 0% coverage, and it blocks TBS-10 and TBS-18.
- **Fix:** anchor the patterns: `/relay`, `/relay-server`, `/relay-server.exe`.
- **Dedup:** NEW. A good first issue.

---

### TBS-15: `stale.yml` auto-closes Dependabot PRs, including the Go and Alpine security bumps

- **Type:** ci · **Severity:** medium · **Status:** VERIFIED · **Effort:** S
- **Evidence**
  - `.github/workflows/stale.yml` marks PRs stale after 14 days and closes them 7 days later. `exempt-pr-labels: 'Keep Open,Work In Progress,WIP'` does not cover bots.
  - Issue events: #71 (golang 1.27.3) was *"closed by github-actions[bot] at 2026-11-17"* with labels `docker,…,Stale`. #16 (alpine 3.24) was closed the same way on 2026-08-30.
  - `sync-pr-labels` also strips `dependencies` from some Dependabot PRs and labels them `first-time-contributor`. #71 and #16 carry `docker` but not `dependencies`, so exempting `dependencies` alone is not enough.
  - Open PR #77 (go-sqlite3) is next in line to be auto-closed.
  - The workflow also uses `actions/stale@v10` by tag, while every other action is SHA-pinned (Scorecard "Pinned-Dependencies").
- **Fix:** add `dependencies,docker,github-actions,go` to `exempt-pr-labels`, or run stale only on issues (`days-before-pr-stale: -1`). Stop `sync-pr-labels` from removing `dependencies` and from tagging bots as first-time contributors. Pin `actions/stale` by SHA.
- **Dedup:** NEW.

---

### TBS-16: Add per-mailbox quotas (max messages/bytes per recipient)

- **Type:** feature · **Severity:** medium · **Status:** VERIFIED (gap) · **Effort:** M
- **Evidence**
  - The only abuse control is the per-IP limiter. Nothing caps how much one mailbox can hold: 100 × 500 KB POSTs from one IP to one address were all `201` in the e2e run.
  - Together with TBS-02 this lets anyone make a victim's mailbox expensive or impossible to read, and fill the disk.
- **Proposal:** add `messages.max_per_mailbox` and `messages.max_bytes_per_mailbox` (0 = unlimited for backward compatibility, with sensible defaults in `config.yaml`). Check them inside the existing `BEGIN IMMEDIATE` transaction in `SaveMessage`, so concurrent inserts cannot overshoot. Reject with `507 Insufficient Storage` or `429` and a clear body. Add a matching client error type in a follow-up. Optionally add a global `storage.max_db_bytes`. Add tests, including a concurrency test like `TestSaveMessage_ConcurrentSeqsAreUnique`.
- **Dedup:** NEW.

---

### TBS-17: Expose server version and limits (`GET /api/info` or `/health` fields, `--version`)

- **Type:** feature · **Severity:** low · **Status:** VERIFIED (gap) · **Effort:** S–M
- **Evidence**
  - The binary has no version string: no ldflags, no `--version`, and the `VERSION` file is never read.
  - Clients cannot discover `max_payload_size` (they learn it from a 413 after uploading), the max page size, TTL, the rate limit, or whether an API key is required.
  - The client README has to say "Older servers send no `seq` and ignore `after` and `limit`" because there is no feature detection.
- **Proposal:** add `GET /api/info` → `{"version":"1.0.0","maxPayloadSize":524288,"maxPageSize":1000,"ttlDays":7,"rateLimitPerMinute":30,"apiKeyRequired":false}`. Make it exempt from the API key (the same public fields), or add `version` to `/health`. Inject the version with `-ldflags "-X main.version=$(cat VERSION)"` in the Dockerfile and log it at startup. Add the method in the SDK later.
- **Dedup:** NEW.

---

### TBS-18: Raise test coverage

- **Type:** test · **Severity:** medium · **Status:** VERIFIED · **Effort:** M
- **Evidence:** `go tool cover -func` gives these figures:
  - `cmd/relay` 0%: `main`, `purgeLoop`, `runPurge`. The middleware **order**, which carries the security properties behind TBS-09 and the CORS/API-key interplay, is only tested piecewise.
  - `HandleHealth` 0%.
  - `model.Validate` 0%.
  - `APIKeyAuth` 61.5%, with no `apikey_test.go` (missing key, wrong key, `/health` exemption, empty-key passthrough).
  - `RateLimiter.cleanup` 0%; window reset and `Retry-After` are untested.
  - `HandleCreate` 60%: no tests for 413, invalid JSON, missing fields, or a store error → 500.
  - `NewSQLiteStore` 53.8% (directory-creation and ping failures).
- **Fix:**
  - Fix TBS-14 first.
  - Extract `func newRouter(cfg *config.Config, s store.Store) (http.Handler, func())` from `main` and test it end to end: preflight with no key, 401, 429, `/health` exemption.
  - Inject the purge ticker interval so the loop can be tested quickly.
  - Add an `apikey_test.go` and handler error-path tables.
  - Gate coverage in CI (TBS-13). The repro tests in this report can be adapted directly.
- **Dedup:** NEW.

---

### TBS-19: Docs refresh

- **Type:** docs · **Severity:** low · **Status:** VERIFIED · **Effort:** S
- **Evidence:**
  - `AGENTS.md:32` says "there are no `_test.go` files". `CONTRIBUTING.md` (Test Your Changes) says "The repository doesn't have test files yet". `BestPracticesChecklist.md` reports "0% — no tests exist yet" and leaves `test_invocation`/`tests_are_added` unmet. In fact there are 54 tests and 64.8% coverage, so the self-reported score (53%) is understated.
  - README "Tech Stack/Prerequisites" says Go 1.22+, but `go.mod` requires 1.23.12, and 1.27 after TBS-01.
  - README and `config.yaml` say "Max payload", but it is the max request body (TBS-03).
  - `ratelimit.go:19` says "sliding window" (it is a fixed window, TBS-12).
  - `cors.go:29` says "OPTIONS keeps 404ing from the router", but the real binary answers `405 Method Not Allowed`.
  - `storage.driver` / `RELAY_STORAGE_DRIVER` exist but are undocumented and have no effect.
  - The `AGENTS.md` layout omits `internal/`.
  - There is no repo `SECURITY.md`. The org default exists (`AOSSIE-Org/.github/SECURITY.md`), and it prefers GitHub Security Advisories "if enabled", but private vulnerability reporting is **disabled** on this repo (`gh api .../private-vulnerability-reporting` returns `{"enabled":false}`).
  - README does not mention that `from` is unauthenticated and freely spoofable (noted as a limitation in the maintainers' audit).
- **Fix:** one docs PR covering all of the above. Ask maintainers to enable private vulnerability reporting, and add a short repo `SECURITY.md` that points to it.
- **Dedup:** NEW.

---

### TBS-20: Cleanup bundle (lint-level, one PR or several tiny ones)

- **Type:** cleanup · **Severity:** low · **Status:** VERIFIED · **Effort:** S each
- 14 golangci-lint `errcheck` hits:
  - `health.go:24` drops the `Encode` error.
  - `model/message.go:34`.
  - `sqlite.go:43,49,116,183,261`.
  - `main.go:68`.
  - Tests.
- `os.Exit(1)` inside the server goroutine (`main.go:128`) and after a failed `Shutdown` (`:144`) skips `db.Close`, `rateLimiter.Stop` and `purgeCancel` (gocritic `exitAfterDefer`).
  - Verified: on `bind: address already in use` the process has already opened the DB and run a purge, then exits without closing it.
  - Fix: return errors from a `run()` function and `os.Exit` only in `main`. Bind the listener (`net.Listen`) before opening the DB.
- The purge goroutine is cancelled but never joined. The real binary logs `server stopped gracefully` **before** `purge loop stopped`. Use a `sync.WaitGroup`.
- Dead code: `if address == ""` in `HandleGetByAddress` (`message.go:110`) is unreachable. `GET /api/messages/` returns the mux's `404 page not found`, verified.
- DB directory is created `0755` (`sqlite.go:32`, gosec G301) and SQLite creates `relay.db` as `-rw-r--r--` (verified). On shared hosts every local user can read all ciphertext and the who-talks-to-whom metadata. Use `0700` and `0600`, or set the umask.
- `idx_messages_to_addr` is redundant with the unique `idx_messages_to_addr_seq` and costs a write per insert.
- Mailbox responses carry no `Cache-Control: no-store`, and JSON responses no `X-Content-Type-Options: nosniff`.
- hadolint DL3018 (pin `apk` versions or accept with an inline ignore).
- Template leftovers:
  - `.goreleaser.yaml` has `project_name: template-repo` and no `builds:`, so no binaries are released, although README advertises a "single binary" (acknowledged in a `release-goreleaser.yml` comment).
  - `dependabot.yml` lists 12 ecosystems that do not exist here (npm, cargo, maven, pip…).
  - `.pre-commit-config.yaml` has no Go hooks (`gofmt`, `go vet`).
- Use `PingContext`/`ExecContext` in schema setup (`noctx`).
- **Dedup:** NEW.

---

## 4. Client/Server protocol mismatches

Compared `ThruBox-Client/src/{client,types,errors}.ts` (commit `9afc707`) with the server. The client audit found several of these independently, and its IDs are cross-referenced.

| # | Mismatch | Evidence | Status | Owner / action |
|---|---|---|---|---|
| S-P1 | **Unbounded reads vs client polling.** `poll()` never sends `limit` and has no page-draining loop (`client.ts:189-192`). Without `after` (the default) it downloads the **whole** mailbox every 10 s. The client timeout is 10 s (`client.ts:18`) and the server write timeout 15 s, so large mailboxes get aborted or truncated (`200` + partial JSON → `RelayNetworkError` → up to 3 GET retries). | Server side VERIFIED (TBS-02: 102 MB / 474 MB alloc, truncation at 17.9 MB of 51 MB). Client retry path `client.ts:250,271,319-335` | VERIFIED | **Server:** default page size + byte budget + next-cursor signal (TBS-02). **Client:** page-draining `poll()` (TBC-09) |
| S-P2 | **Address canonicalisation.** Both sides treat addresses as exact strings. EIP-55 checksummed and lowercase forms do not match | TBS-07 repro; client P-6 | VERIFIED | Decision needed: server normalisation for `0x`+40-hex, or documented opaque keys plus a client normaliser |
| S-P3 | **Timestamp precision** differs between POST and GET or retry | TBS-11; client P-1 | VERIFIED | Server: truncate to seconds |
| S-P4 | **`max_payload_size` is a body limit**, but both docs and `RelayPayloadTooLargeError` say "payload". The client cannot learn the limit before uploading | TBS-03; client P-2; no discovery endpoint (TBS-17) | VERIFIED | Server: enforce on the payload field or rename; expose limits (TBS-17) |
| S-P5 | **`Retry-After` hidden from browsers and inaccurate.** No `Access-Control-Expose-Headers`, so `RelayRateLimitError.retryAfter` always falls back to 60 cross-origin. The server always sends the full window anyway | TBS-12; client P-4 | VERIFIED | Server: expose the header and compute the remaining time |
| S-P6 | **`X-API-Key` is only allowed in CORS when the server has a key.** A client configured with `apiKey` against an open relay fails every browser request with an opaque CORS error. The client always sends the header when `apiKey` is set (`client.ts:246-248`) | Preflight with `Access-Control-Request-Headers: content-type,x-api-key` returns `204 Allow-Headers="Content-Type"` (`cors.go:43-46`); client P-5 | VERIFIED | Server: always list `X-API-Key` in `Access-Control-Allow-Headers` (one line, harmless) |
| S-P7 | **Default rate limit vs default polling.** 30 req/min/IP (`config.go`) vs `poll()` every 10 s per address (6/min), plus 3 GET retries on 5xx/network errors. Five tabs or addresses, or several users behind one NAT, or every user behind a proxy with `trusted_proxies: 0`, exhaust the bucket. `poll()` then ignores `Retry-After` (client TBC-04) | Arithmetic from the defaults. 429 behaviour VERIFIED in tests | SUSPECTED (impact depends on deployment) | Docs: state the polling budget in both READMEs. Server: IPv6 /64 grouping (TBS-04) avoids over- or under-limiting. Client: back off on 429 (TBC-04) |
| S-P8 | **Error contract.** All errors are `text/plain` from `http.Error`, with a trailing `\n` and no machine-readable code (`"field 'from' is required\n"`, `"404 page not found\n"`). The client copies the text into `RelayError.message` (`client.ts:307-308`) | e2e `od -c` shows the trailing `\n`; client P-3 | VERIFIED | Server (cross-repo, needs a design issue): JSON envelope `{"error":{"code":"missing_field","message":"..."}}` with `Content-Type: application/json`, versioned via `/api/info` (TBS-17). Client: trim now (TBC-08) |
| S-P9 | **No capability/version discovery.** The client guesses features ("older servers send no `seq`") | TBS-17 | VERIFIED (gap) | Server: `/api/info` |
| S-P10 | **POST is idempotent server-side, but the client never retries it** (`maxRetries = 0` for POST, `client.ts:250`) | Server returns `200` with the stored message on a duplicate (`message.go:82-87`) | VERIFIED | Client only (tracked by client #82 / PR #83) |
| S-P11 | **`from` is unauthenticated.** The server stores it verbatim, and the client types call it "Sender wallet address". Consumers must authenticate the sender inside the encrypted payload | `model/message.go:40-44`; maintainers' audit "Review-specific limitations" | VERIFIED (by design) | Docs in both repos (TBS-19, TBC-16) |

No mismatch found in:
- The JSON field names and types (`id`, `seq`, `to`, `from`, `payload`, `createdAt`, `expiresAt: string|null`, `HealthResponse`).
- The `after`/`limit` names and validation (`limit` 1–1000; `limit=0` returns `400`).
- 401, 413 and 429 status mapping.
- `/health` being exempt from the API key.
- `DELETE` returning `405 Allow: GET, HEAD`. The client's `delete()` was removed too.
- Reserved characters in addresses round-tripping via `encodeURIComponent` and `PathValue` (`a/b` → `%2F` works; `..` does not, see TBS-03).

---

## 5. Things checked that turned out fine (no issue needed)

- **Concurrency:** 400 concurrent 64 KB POSTs to 4 mailboxes on a file DB gave `map[201:400]` in 1.1 s, with no `database is locked` errors. `seq` stays unique (the existing test also covers this). `go test -race` is clean.
- **SQL:** all queries are parameterised. The content-ID uses length prefixes (no field-shifting collisions).
- **Schema migration (`migrateAddSeq`)** scales fine: 32,000 legacy rows took 152 ms.
- **Graceful shutdown** on SIGTERM works ("server stopped gracefully"). `ReadTimeout`/`WriteTimeout`/`IdleTimeout` are set, so the classic slowloris on headers is bounded.
- **`X-Forwarded-For` handling** with `trusted_proxies` matches the README. API-key comparison is constant-time. The API key is never logged.
- **CORS:** a preflight is answered before auth. Credentials are never allowed. `Vary: Origin` is set.

---

## 6. Suggested split for a 3-person team

| Person | Track | Findings (in order) |
|---|---|---|
| A | Security/runtime | TBS-01 (S, do first), TBS-02 (M), TBS-16 (M) |
| B | Validation/config/rate limiting | TBS-03, TBS-05, TBS-04, TBS-12, TBS-09, TBS-08 |
| C | CI/tests/ops | TBS-14 (first, it unblocks tests), TBS-13, TBS-15, TBS-18, TBS-06, TBS-10, then TBS-19/TBS-20 |

Needs a maintainer decision before coding: TBS-07 (address normalisation), S-P8 (error envelope), TBS-02's response-shape change, TBS-16 defaults. Per `CONTRIBUTING.md`, open an issue and post in the Discord `#thrubox` channel before starting.

---

## Appendix: tool commands used

```
go build ./... ; go vet ./... ; go test -race -count=1 -cover ./...
go test -coverprofile=cover.out ./... && go tool cover -func=cover.out
staticcheck ./... ; staticcheck -checks all ./...
golangci-lint run --max-issues-per-linter=0 --max-same-issues=0 ./...
golangci-lint run --enable gosec,noctx,gocritic,errorlint,unparam ./...
gosec ./...
GOTOOLCHAIN=go1.23.12 govulncheck ./...
hadolint Dockerfile                      # v2.12.0 binary
git check-ignore -v cmd/relay/main_test.go
gh issue list / gh pr list / gh api repos/AOSSIE-Org/ThruBox-Server/issues/71/events
```

The scratch copy, repro tests, binaries and tool caches lived under the session scratchpad and were deleted after the audit. The repro snippets above are enough to recreate them on top of the `buildChain` harness.
