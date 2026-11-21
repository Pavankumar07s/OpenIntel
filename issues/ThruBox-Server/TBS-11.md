# [BUG]: POST /api/messages returns nanosecond createdAt/expiresAt, but the stored message (GET, retried POST) has second precision

## Bug Description

When a message is newly stored, `POST /api/messages` (201) returns `createdAt` and `expiresAt` with nanoseconds. Every later view of the same message has whole seconds: `GET /api/messages/{address}` and a retried identical `POST` (200). So one message has two different timestamps, depending on which response the client kept. The README example for the 201 response also shows second precision (`"createdAt": "2026-10-26T10:00:00Z"`).

Cause:

- `internal/handler/message.go:59` sets `now := time.Now().UTC()` and uses it for `CreatedAt` and `ExpiresAt`. On a new insert, `:92` encodes this in-memory value.
- `internal/store/sqlite.go:218` and `:231` save both fields with `Format(time.RFC3339)`, which drops the fraction. Reads parse that stored string, and on a duplicate `SaveMessage` copies the stored row back into the message (`sqlite.go:194`).

The store test already expects the stored value to be truncated (`internal/store/sqlite_test.go:143` compares with `original.CreatedAt.Truncate(time.Second)`). Only the first 201 response is different.

Effect: if a client compares or deduplicates messages by `createdAt`/`expiresAt`, it sees two values for one message. The `expiresAt` the sender gets can also be up to 1 s later than the expiry the server actually enforces.

## Steps to Reproduce

1. Start the relay with the default `config.yaml` (`ttl_days: 7`): `go build -o relay ./cmd/relay && ./relay`
2. Send a message, read it back, then send the identical message again:
   ```bash
   B='{"to":"0xrecipient","from":"0xsender","payload":"aGVsbG8="}'
   curl -s -H 'Content-Type: application/json' -d "$B" http://localhost:3000/api/messages
   curl -s http://localhost:3000/api/messages/0xrecipient
   curl -s -H 'Content-Type: application/json' -d "$B" http://localhost:3000/api/messages
   ```
3. Compare `createdAt` / `expiresAt` across the three responses.

**Expected:** all three responses show the same `createdAt` and `expiresAt` (second precision, as in the README example).

**Actual:** only the first response (201) has nanoseconds.

## Logs and Screenshots

```
POST (201): "createdAt":"2026-11-19T18:52:28.111980344Z","expiresAt":"2026-11-26T18:52:28.111980344Z"
GET  (200): "createdAt":"2026-11-19T18:52:28Z","expiresAt":"2026-11-26T18:52:28Z"
POST (200): "createdAt":"2026-11-19T18:52:28Z","expiresAt":"2026-11-26T18:52:28Z"
```

## Environment Details

- ThruBox-Server `main` @ `f026f22` (VERSION 1.0.0), built from source
- Linux, Go toolchain from `go.mod`
- Default `config.yaml` (any `ttl_days > 0` shows it for `expiresAt`; `createdAt` is affected with any TTL)

## Impact

Low - Minor inconvenience

## Proposed fix

Truncate the timestamp in the handler, so the response matches what is stored:

```go
now := time.Now().UTC().Truncate(time.Second)
```

`expiresAt` is computed from `now`, so it then has whole seconds too. I checked this change locally: the three responses above match and `go test ./...` still passes. Add a handler test that POSTs, GETs and re-POSTs one message and checks that the timestamps are equal.

Please don't fix it by switching the store to `time.RFC3339Nano`. That layout drops trailing zeros, so the stored strings would no longer have a fixed width. The expiry checks compare strings (`expires_at >= ?`, `expires_at < ?`), and they would break: for example, `"…:28.5Z" < "…:28Z"` as strings. If sub-second precision is ever needed, use a fixed-width layout or integer Unix milliseconds, and migrate the existing rows.
