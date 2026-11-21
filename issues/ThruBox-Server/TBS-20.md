# [BUG]: Small robustness cleanups: database is opened before the port is bound, `os.Exit` skips `db.Close`, `relay.db` is world-readable, one index is never used

## Bug Description

Four small, independent problems in `cmd/relay/main.go` and `internal/store/sqlite.go` (`main` @ `f026f22`):

1. **The database is set up before the listen address is checked, and a bind failure skips cleanup.** `main` opens the database and creates the schema (running the migration if one is needed). It also starts the purge goroutine. Only after that does `ListenAndServe` run, inside a goroutine. If the port is taken, that goroutine calls `os.Exit(1)` (`main.go:128`), so the deferred `db.Close()`, `rateLimiter.Stop()` and `purgeCancel()` never run. `main.go:144` does the same when `Shutdown` times out. The `-wal`/`-shm` files are left behind, while a clean shutdown checkpoints and removes them. There is no data loss, because SQLite recovers on the next open, but a misconfigured port should fail before the database is touched.
2. **The database is readable by every local user.** `NewSQLiteStore` creates the directory with `0755` (`sqlite.go:32`), and SQLite creates `relay.db`, `-wal` and `-shm` as `0644`. On a shared host, any local account can copy all stored ciphertext and the sender/recipient metadata. That bypasses `security.api_key` on a private relay. The Docker image is mostly unaffected, because `/data` is inside the container.
3. **`idx_messages_to_addr` is redundant.** The only query that filters on `to_addr` (`GetMessagesByAddress`) uses the unique `idx_messages_to_addr_seq (to_addr, seq)`, even when the single-column index exists. A plain `to_addr = ?` lookup also uses the composite index's prefix. The extra index still costs a write on every insert, plus disk space.
4. **`HandleHealth` ignores the `Encode` error** (`health.go:24`). `HandleCreate` and `HandleGetByAddress` log it. This is the only unchecked error in non-test code that is not an idiomatic deferred `Close`/`Rollback`.

## Steps to Reproduce

```sh
go build -o relay-server ./cmd/relay

# 1. Port already in use
RELAY_CONFIG_PATH=/nonexistent RELAY_SERVER_PORT=3000 RELAY_STORAGE_PATH=/tmp/a/relay.db ./relay-server &
sleep 1
RELAY_CONFIG_PATH=/nonexistent RELAY_SERVER_PORT=3000 RELAY_STORAGE_PATH=/tmp/b/relay.db ./relay-server; echo "exit=$?"
ls -la /tmp/b          # relay.db, relay.db-shm, relay.db-wal were created; -rw-r--r--, directory drwxr-xr-x

# 3. Query plan
sqlite3 /tmp/a/relay.db "EXPLAIN QUERY PLAN SELECT id FROM messages WHERE to_addr='x' AND seq > 0 ORDER BY seq;"
sqlite3 /tmp/a/relay.db "DROP INDEX idx_messages_to_addr; EXPLAIN QUERY PLAN SELECT id FROM messages WHERE to_addr='x';"
```

## Logs and Screenshots

```
level=INFO msg="database initialized" path=/tmp/b/relay.db
level=INFO msg="auto-purge enabled" ttl_days=7
level=INFO msg="relay server starting" addr=0.0.0.0:3000
level=ERROR msg="server failed" error="listen tcp 0.0.0.0:3000: bind: address already in use"
exit=1

-rw-r--r-- relay.db   -rw-r--r-- relay.db-shm   -rw-r--r-- relay.db-wal

`--SEARCH messages USING INDEX idx_messages_to_addr_seq (to_addr=? AND seq>?)
`--SEARCH messages USING INDEX idx_messages_to_addr_seq (to_addr=?)
```

## Environment Details

- ThruBox-Server `main` @ `f026f22` (`VERSION` 1.0.0), Linux, Go 1.24.6, SQLite 3 CLI for the query plan.

## Impact

Low - Minor inconvenience

**Proposed fix**

- Move the body of `main` into `run() error`, and call `os.Exit` only in `main` after the deferred cleanup has run. Call `net.Listen` before `NewSQLiteStore`, then `srv.Serve(ln)`. Send serve errors back over a channel instead of exiting inside the goroutine.
- Create the directory with `0o700`, and create `relay.db` with `0o600` before `sql.Open` (or set `syscall.Umask(0o077)` at startup). Document that operators who want group access can loosen it.
- Add `DROP INDEX IF EXISTS idx_messages_to_addr;` to `initSchema` and remove its `CREATE`.
- Log the `Encode` error in `HandleHealth`, as the other handlers do.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
