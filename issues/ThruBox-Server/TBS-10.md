# [BUG]: Switching to `ttl_days: 0` leaves already-expiring messages on disk after they expire

## Bug Description

`cmd/relay/main.go:75` only starts the hourly purge goroutine when `messages.ttl_days > 0`:

```go
if cfg.Messages.TTLDays > 0 {
    go purgeLoop(purgeCtx, db)
    ...
} else {
    slog.Warn("auto-purge disabled (ttl_days=0): messages are kept forever and the database only grows")
}
```

`ttl_days` only decides the `expires_at` given to **new** messages (`internal/handler/message.go:69`). Messages stored while `ttl_days` was > 0 (the default is 7) keep their own `expires_at`. If an operator later switches to `ttl_days: 0`, those messages still expire: `GetMessagesByAddress` stops returning them once `expires_at` has passed (`internal/store/sqlite.go:249`). But nothing deletes them, because `PurgeExpired` never runs.

These rows end up in a dead state. Nobody can read them, and the API cannot delete them (`DELETE /api/messages/{id}` was removed, and expiry is now the only way messages leave the relay). They stay on disk past the `expiresAt` the server gave the sender in the POST response, and are only removed if `ttl_days` is set back above 0 or an identical message is sent again (`SaveMessage` deletes an expired copy with the same ID). At most, the leftover rows are the messages created during the last `ttl_days` window before the switch.

## Steps to Reproduce

On `main` (`f026f22`), using the real binary and the `sqlite3` CLI:

```bash
go build -o relay ./cmd/relay
export RELAY_CONFIG_PATH=/nonexistent RELAY_SERVER_HOST=127.0.0.1 RELAY_SERVER_PORT=31281 RELAY_STORAGE_PATH=$PWD/data/relay.db

# 1. Run with the default TTL and store two messages
RELAY_MESSAGES_TTL_DAYS=7 ./relay & PID=$!; sleep 1
for p in msg-A msg-B; do
  curl -s -o /dev/null -w "%{http_code}\n" -XPOST 127.0.0.1:31281/api/messages \
    -H 'Content-Type: application/json' -d "{\"to\":\"0xBob\",\"from\":\"0xAlice\",\"payload\":\"$p\"}"
done
kill $PID

# 2. Simulate the passage of time for msg-A (its expiry is now in the past)
sqlite3 data/relay.db "UPDATE messages SET expires_at = strftime('%Y-%m-%dT%H:%M:%SZ','now','-1 day') WHERE payload='msg-A';"

# 3. Operator switches to "keep forever"
RELAY_MESSAGES_TTL_DAYS=0 ./relay & PID=$!; sleep 1.5
curl -s 127.0.0.1:31281/api/messages/0xBob      # only msg-B is returned
kill $PID
sqlite3 data/relay.db "SELECT count(*) FROM messages;"

# 4. Control: same DB with ttl_days=7
RELAY_MESSAGES_TTL_DAYS=7 ./relay & PID=$!; sleep 1.5; kill $PID
sqlite3 data/relay.db "SELECT count(*) FROM messages;"
```

**Expected:** a message whose `expires_at` has passed is deleted by the purge, whatever `ttl_days` is set to for new messages.

**Actual:** with `ttl_days: 0` the expired message is hidden from reads but stays in the database. It is only removed after `ttl_days` is set back above 0.

## Logs and Screenshots

```
## ttl_days=0 run
GET /api/messages/0xBob -> ['msg-B']
msg="auto-purge disabled (ttl_days=0): messages are kept forever and the database only grows"
rows on disk after ttl_days=0 run: 2
b66ccf8d...901b|2026-11-18T18:52:19Z     <- expired, unreadable, not deleted
ca02277f...d644|2026-11-26T18:52:19Z

## control: ttl_days=7 run on the same DB
msg="purged expired messages" count=1
rows on disk after ttl_days=7 run: 1
```

## Environment Details

- ThruBox-Server `main` @ `f026f22` (VERSION 1.0.0)
- Linux x86_64, built locally with Go 1.24.6 (CGo, `mattn/go-sqlite3`)
- No config file, configured through `RELAY_*` env vars

## Impact

Low - Minor inconvenience

## Proposed fix

Always start `purgeLoop`, whatever `ttl_days` is set to. `PurgeExpired` only deletes rows with a non-NULL `expires_at` in the past (`DELETE ... WHERE expires_at IS NOT NULL AND expires_at < ?`, which uses `idx_messages_expires_at`). In pure "forever" mode, where every row has `expires_at = NULL`, it is a cheap hourly no-op. Keep the existing warning for `ttl_days: 0`, perhaps reworded to say that only new messages are kept forever. Update the comment at `main.go:72`.

If a test is added under `cmd/relay/` (for example `main_test.go`), note that the `.gitignore` entry `relay` currently ignores new files in that directory, so the entry has to be anchored (`/relay`) or the file force-added.
