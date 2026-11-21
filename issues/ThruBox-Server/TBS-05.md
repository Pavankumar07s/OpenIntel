# [BUG]: Config silently ignores typo'd keys and accepts values that break the relay (`max_payload_size: 0`, `rate_limit: 0`, `ttl_days` > 106751, IPv6 `server.host`, any `storage.driver`)

## Bug Description

`config.Load` starts the server with several configurations that do not do what the operator wrote, and nothing at startup says so:

1. **Unknown YAML keys are ignored.** `yaml.Unmarshal` (`internal/config/config.go:106`) is not strict. A typo such as `api-key:` / `apiKey:` (instead of `api_key:`) or a misspelled section (`secuirty:`) is dropped, and the default is used instead. With a typo'd API key the relay runs with no authentication. The startup log does not show whether API-key auth is enabled, so the operator cannot spot the problem there.
2. **`max_payload_size: 0` passes `Validate`** (`config.go:211` only rejects `< 0`). The handler then wraps every body in `http.MaxBytesReader(w, r.Body, 0)`, so every `POST /api/messages` returns `413 payload too large`. `max_payload_size` is also missing from the "configuration loaded" log line.
3. **`rate_limit: 0` passes `Validate`** (`config.go:214`). It means neither "off" nor "block everything". It means 1 request per minute per IP, because `allow()` returns `true` for the first request of each window before it checks the limit (`internal/middleware/ratelimit.go:73-80`). Many tools use 0 to mean "disabled", and the README does not say what 0 does.
4. **`storage.driver` / `RELAY_STORAGE_DRIVER` accepts any value.** The value is logged as `storage_driver=postgres` (`cmd/relay/main.go:54`), but SQLite is always used. The key is in `config.yaml` but not in the README configuration table.
5. **The warning for an unparseable numeric env var is misleading.** For example, `RELAY_SECURITY_RATE_LIMIT=10/min` prints `warning: invalid RELAY_SECURITY_RATE_LIMIT="10/min", using default` (`config.go:132-176`). When the YAML file sets that field, the value kept is the YAML value, not the default. These are the only `log.Printf` calls in the program. Because `main` installs a `slog` default handler first, they come out as `level=INFO msg="warning: ..."`, so filtering on `level=WARN` misses them.
6. **`messages.ttl_days` has no upper bound** (`config.go:208` only rejects `< 0`). `HandleCreate` computes `now.Add(time.Duration(h.TTLDays) * 24 * time.Hour)` (`internal/handler/message.go:70`). `time.Duration` is int64 nanoseconds and tops out at about 106751.99 days, so from `ttl_days: 106752` on the multiplication wraps silently. For 106752 to 213503, and for about half of all larger values (e.g. `999999`), the stored expiry is in the past: the POST still returns `201` with that past `expiresAt`, `GET` filters the message out at once (`internal/store/sqlite.go:249`), and the next purge deletes it. Other values give an unrelated future expiry (`213504` expires about 25 minutes after sending). An operator who writes a large number to mean "keep practically forever" (instead of `0`) loses every message.
7. **IPv6 `server.host` values pass validation but cannot be bound.** `Config.Addr()` (`config.go:268-270`) joins host and port with `fmt.Sprintf("%s:%d")` and does not bracket IPv6 literals. `server.host: "::1"` or a specific IPv6 address (also via `RELAY_SERVER_HOST`) becomes `::1:3000`, which `ListenAndServe` rejects with `too many colons in address`, and the process exits after the database has already been opened. Writing `"[::1]"` works today, but nothing documents it.

(Not part of this report: an unparseable env value being a warning instead of a startup error is intentional, as `TestLoad_PortOutOfRangeIsRejected` documents. Only the wording and level of that warning are wrong.)

## Steps to Reproduce

```bash
go build -o relay-server ./cmd/relay
BODY='{"to":"0xabc","from":"0xdef","payload":"aGVsbG8="}'

# 1. typo'd api key -> relay is open
printf 'security:\n  api-key: "topsecret"\n' > typo.yaml
RELAY_CONFIG_PATH=typo.yaml ./relay-server &
curl -s -o /dev/null -w '%{http_code}\n' -X POST -H 'Content-Type: application/json' -d "$BODY" localhost:3000/api/messages
# -> 201 (with the correct key name `api_key` the same request gets 401)
kill %1

# 2. max_payload_size: 0 -> every POST is rejected
printf 'messages:\n  max_payload_size: 0\n' > p0.yaml
RELAY_CONFIG_PATH=p0.yaml ./relay-server &
curl -s -w ' %{http_code}\n' -X POST -H 'Content-Type: application/json' -d "$BODY" localhost:3000/api/messages
# -> payload too large 413
kill %1

# 3. rate_limit: 0 -> 1 request per minute
printf 'security:\n  rate_limit: 0\n' > r0.yaml
RELAY_CONFIG_PATH=r0.yaml ./relay-server &
for i in 1 2 3; do curl -s -o /dev/null -w '%{http_code} ' localhost:3000/api/messages/0xabc; done; echo
# -> 200 429 429
kill %1

# 6. ttl_days overflow -> 201 with an expiry in 1989, then GET returns []
RELAY_MESSAGES_TTL_DAYS=200000 ./relay-server &
curl -s -X POST -H 'Content-Type: application/json' -d "$BODY" localhost:3000/api/messages
curl -s localhost:3000/api/messages/0xabc
kill %1

# 7. IPv6 host -> exits at bind time; the bracketed form works
RELAY_SERVER_HOST=::1 ./relay-server         # listen tcp: address ::1:3000: too many colons in address
RELAY_SERVER_HOST='[::1]' ./relay-server     # starts

# 4 + 5. any driver is accepted; bad env value keeps the YAML value but says "using default"
printf 'storage:\n  driver: "postgres"\nsecurity:\n  rate_limit: 5\n  trusted_proxies: 2\n' > misc.yaml
RELAY_CONFIG_PATH=misc.yaml RELAY_SECURITY_RATE_LIMIT=10/min RELAY_SECURITY_TRUSTED_PROXIES=one ./relay-server
```

## Logs and Screenshots

Output from step 4 + 5 (timestamps, file paths and the port field trimmed):

```
level=INFO msg="warning: invalid RELAY_SECURITY_RATE_LIMIT=\"10/min\", using default"
level=INFO msg="warning: invalid RELAY_SECURITY_TRUSTED_PROXIES=\"one\", using default"
level=INFO msg="configuration loaded" storage_driver=postgres ttl_days=7 rate_limit=5 trusted_proxies=2 allowed_origins=[]
level=INFO msg="database initialized" path=.../relay.db
```

The defaults are `rate_limit=30` and `trusted_proxies=0`, but the values in use are the YAML ones (5 and 2). `storage_driver=postgres` is logged, but a SQLite file is created.

For step 1, the startup log looks exactly like a normal start. There is no warning and no sign that authentication is off.

Item 6 (`config.Load` + `MessageHandler`, POST then GET then `PurgeExpired`, now = 2026-11-19T19:33:58Z):

```
ttl_days=106751  POST=201 expiresAt=2319-02-28T19:33:58Z GET=[{...}] purged=0
ttl_days=106752  POST=201 expiresAt=1734-08-10T19:59:24Z GET=[]      purged=1
ttl_days=200000  POST=201 expiresAt=1989-11-29T19:59:24Z GET=[]      purged=1
ttl_days=213504  POST=201 expiresAt=2026-11-19T19:59:24Z GET=[{...}] purged=0   (expires ~25 min later)
ttl_days=999999  POST=201 expiresAt=1842-01-07T21:41:09Z GET=[]      purged=1
```

Item 7 (`config.Load` + `net.Listen("tcp", cfg.Addr())`):

```
host="::"     Addr()=":::38241"   listen err=listen tcp: address :::38241: too many colons in address
host="::1"    Addr()="::1:38241"  listen err=listen tcp: address ::1:38241: too many colons in address
host="[::1]"  Addr()="[::1]:38241" listen err=<nil>
```

## Environment Details

- ThruBox-Server `main` @ `f026f22` (VERSION 1.0.0)
- Built from source with Go 1.24.6 on Linux. The code paths do not depend on the OS.

## Impact

Low - Minor inconvenience

These are operator misconfigurations, not attacker-reachable bugs. Items 2, 3 and 7 fail loudly (on the first requests, or at startup). Items 1, 4, 5 and 6 are silent. Item 1 matters most, because a relay the operator meant to put behind an API key runs open.

## Proposed fix

Almost all of this fits in `internal/config/config.go` and needs no new dependency:

- Decode with `dec := yaml.NewDecoder(bytes.NewReader(data)); dec.KnownFields(true)`, and treat `io.EOF` (an empty or comment-only file) as "no keys". With this, `api-key:` fails with `field api-key not found in type config.SecurityConfig`. The shipped `config.yaml` still decodes cleanly.
- In `Validate`, require `max_payload_size >= 1` and `rate_limit >= 1`. Alternatively, give `rate_limit: 0` an explicit meaning ("disabled") and document it in the README table.
- In `Validate`, reject `ttl_days` above `math.MaxInt64 / int64(24*time.Hour)` (106751), or above a saner documented cap such as 36500, and say in the error that `0` means "keep forever".
- Build `Addr()` with `net.JoinHostPort(host, strconv.Itoa(port))`, after stripping one pair of surrounding brackets from `host` so configs that already use `"[::1]"` keep working (otherwise they would become `[[::1]]:3000`). Mention in the README config table that IPv6 hosts are accepted.
- Reject `storage.driver` values other than `"sqlite"`, or drop the field.
- Replace the `log.Printf` warnings with `slog.Warn("ignoring invalid env value", "var", ..., "value", ..., "using", <value actually kept>)`.
- In `main.go`, add `max_payload_size` and `api_key_set=true/false` to the "configuration loaded" line.
- Add table tests in `internal/config` for each case, including `ttl_days` 106751/106752 and `Addr()` with `0.0.0.0`, `::`, `::1`, `[::1]` and `localhost`.
