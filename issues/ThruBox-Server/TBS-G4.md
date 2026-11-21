<!-- Team note, not issue text: do not file. Merged into TBS-05.md; this file is kept only as evidence. -->

# [BUG]: Config accepts `ttl_days` above 106751, which overflows the expiry to a wrong date, and IPv6 `server.host` values such as `::1` that cannot be bound

## Bug Description

`Config.Validate` (`internal/config/config.go:204`) accepts two kinds of values that the relay cannot use. The relay starts with them anyway and only goes wrong later.

**1. `messages.ttl_days` has no upper bound.** `Validate` only rejects `TTLDays < 0` (`config.go:208`). `HandleCreate` turns it into a `time.Duration` (`internal/handler/message.go:70`):

```go
exp := now.Add(time.Duration(h.TTLDays) * 24 * time.Hour)
```

`time.Duration` is int64 nanoseconds, so it tops out at about 106751.99 days (~292 years). From `ttl_days: 106752` on, the multiplication wraps around silently and the stored `expires_at` is effectively random:

- For 106752 to 213503, and in about half of all larger values (e.g. 999999), the expiry is in the past. The POST still returns `201 Created` with that past `expiresAt`. `GET /api/messages/{address}` filters the message out right away (`internal/store/sqlite.go:249`), and the next purge deletes it. The sender gets a success response, and the recipient never sees the message.
- In other cases the expiry is in the future but has nothing to do with the setting. `ttl_days: 213504` makes messages expire about 25 minutes after they are sent, and `300000` gives the year 2263.

An operator who wants messages kept "practically forever" and writes a large number such as `999999` (instead of `0`) loses every message. Values up to 106751 work as expected.

**2. `server.host` is joined to the port with `fmt.Sprintf("%s:%d")`.** `Config.Addr()` (`config.go:268-270`) does not bracket IPv6 literals. `server.host: "::1"`, `"::"` or a global IPv6 address (also via `RELAY_SERVER_HOST`) passes validation. The resulting address (`::1:3000`, `:::3000`) is then rejected by `ListenAndServe` (`cmd/relay/main.go:126-128`) with `too many colons in address`, and the process exits. The failure is loud, but it happens at runtime rather than in config validation. By then the database has already been opened (the ordering and the `os.Exit` that skips deferred cleanup are a separate issue). Writing the host with brackets (`"[::1]"`) works today, but nothing documents that. Note that `"::"` gains nothing over the default `0.0.0.0` on Linux, because Go already listens dual-stack there. The realistic cases are `::1` (IPv6 loopback only) and a specific IPv6 address.

## Steps to Reproduce

On `main` (`f026f22`), with the real binary:

```bash
go build -o relay ./cmd/relay
export RELAY_CONFIG_PATH=/nonexistent RELAY_STORAGE_PATH=$PWD/data/relay.db

# 1. ttl_days overflow
RELAY_SERVER_HOST=127.0.0.1 RELAY_SERVER_PORT=38243 RELAY_MESSAGES_TTL_DAYS=200000 ./relay & PID=$!; sleep 1
curl -s -w ' %{http_code}\n' -X POST -H 'Content-Type: application/json' \
  -d '{"to":"0xabc","from":"0xdef","payload":"hello"}' http://127.0.0.1:38243/api/messages
curl -s -w ' %{http_code}\n' http://127.0.0.1:38243/api/messages/0xabc
sqlite3 data/relay.db 'select seq, expires_at from messages'
kill $PID

# 2. IPv6 host
RELAY_SERVER_HOST=::1 RELAY_SERVER_PORT=38242 ./relay      # exits immediately
RELAY_SERVER_HOST='[::1]' RELAY_SERVER_PORT=38244 ./relay  # works
```

To cover more values, a test can call `config.Load` on a YAML file with `messages: {ttl_days: N}` and then POST and GET through `MessageHandler`.

**Expected:** values the relay cannot honour are rejected by `Validate` at startup with a clear message. Alternatively, `ttl_days` is capped, and any valid IPv6 host is bracketed correctly.

**Actual:** both values pass validation. The large TTL silently gives messages a wrong expiry, often in the past. The IPv6 host makes the server exit at bind time.

## Logs and Screenshots

```
## RELAY_MESSAGES_TTL_DAYS=200000 (real binary)
level=INFO msg="auto-purge enabled" ttl_days=200000
POST -> 201 {"id":"f3bb...30b4","seq":1,...,"createdAt":"2026-11-19T19:34:21.357379735Z","expiresAt":"1989-11-29T19:59:47.647828119Z"}
GET  -> 200 []
sqlite: 1|1989-11-29T19:59:47Z

## config.Load + MessageHandler, POST then GET then PurgeExpired (now = 2026-11-19T19:33:58Z)
ttl_days=7       POST=201 expiresAt=2026-11-26T19:33:58Z GET=[{...}] purged=0
ttl_days=106751  POST=201 expiresAt=2319-02-28T19:33:58Z GET=[{...}] purged=0
ttl_days=106752  POST=201 expiresAt=1734-08-10T19:59:24Z GET=[]      purged=1
ttl_days=200000  POST=201 expiresAt=1989-11-29T19:59:24Z GET=[]      purged=1
ttl_days=213504  POST=201 expiresAt=2026-11-19T19:59:24Z GET=[{...}] purged=0   (expires ~25 min later)
ttl_days=300000  POST=201 expiresAt=2263-09-14T19:59:24Z GET=[{...}] purged=0
ttl_days=999999  POST=201 expiresAt=1842-01-07T21:41:09Z GET=[]      purged=1

## RELAY_SERVER_HOST=:: (real binary)
level=INFO msg="database initialized" path=.../relay.db
level=INFO msg="relay server starting" addr=:::38242
level=ERROR msg="server failed" error="listen tcp: address :::38242: too many colons in address"
exit=1

## config.Load + net.Listen("tcp", cfg.Addr())
host="::"     Addr()=":::38241"   listen err=listen tcp: address :::38241: too many colons in address
host="::1"    Addr()="::1:38241"  listen err=listen tcp: address ::1:38241: too many colons in address
host="[::1]"  Addr()="[::1]:38241" listen err=<nil>
```

## Environment Details

- ThruBox-Server `main` @ `f026f22` (VERSION 1.0.0)
- Linux x86_64, built locally with Go 1.24.6 (CGo, `mattn/go-sqlite3`)
- Configured through `RELAY_*` env vars; the same happens with `config.yaml`

## Impact

Low - Minor inconvenience

## Proposed fix

- `Validate`: reject `ttl_days` above `math.MaxInt64 / int64(24*time.Hour)` (106751), or above a saner documented cap such as 36500. Make the error say that `0` means "keep forever".
- `Addr()`: build the address with `net.JoinHostPort(host, strconv.Itoa(port))`. Strip one pair of surrounding brackets from `host` first. Otherwise configs that already use `"[::1]"` would become `[[::1]]:3000` and break.
- Optionally have `Validate` check that the host is empty, an IP, or a hostname, so a bad value is reported as a config error.
- Add table tests to `internal/config/config_test.go` for `ttl_days` 106751/106752 and for `Addr()` with `0.0.0.0`, `::`, `::1`, `[::1]` and `localhost`. Mention in the README config table that IPv6 hosts are accepted.
