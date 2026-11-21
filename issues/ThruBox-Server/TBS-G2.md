# [BUG]: Saved `after=<seq>` cursors silently skip new messages after the database is recreated or restored from a backup

### Bug Description

`seq` comes from the per-mailbox counter in the `mailboxes` table (`internal/store/sqlite.go:205-212`):

```sql
INSERT INTO mailboxes (to_addr, last_seq) VALUES (?, 1)
ON CONFLICT (to_addr) DO UPDATE SET last_seq = last_seq + 1
RETURNING last_seq
```

That counter only exists in the database file. If the database is replaced, the counter starts again:

- **New, empty database:** every mailbox starts again at `seq` 1.
- **Restored backup:** every mailbox goes back to the `last_seq` it had when the backup was taken, and those numbers are handed out again to different messages.

The server never compares `after` with the mailbox's counter (`internal/handler/message.go:117-125` accepts any non-negative integer). No response includes anything that identifies the database. A client holding a cursor from before the reset therefore gets `[]` with `200 OK` and cannot tell anything is wrong. Every new message with `seq <= cursor` is skipped. The client cursor only moves forward (`ThruBox-Client` `src/client.ts:200-210`), so the client never goes back for them. A mailbox whose cursor was at 500 stays silent until it has received 500 new messages.

The docs promise that this cannot happen:

- README: "Numbers only go up and are never reused, even after a message expires."
- `internal/store/store.go:15-17`: "Sequence numbers only ever increase within a mailbox, even after expired messages are purged, so a client's cursor never skips a message."
- CHANGELOG 1.0.0: "Numbers are never reused, even after expiry."
- Client README: "It only increases and is never reused, so remembering the highest `seq` you have handled and passing it as `after` gives you exactly the new messages."

The `mailboxes` table was added so that "a client already past that cursor" never misses a message after a purge (#69). Losing or restoring the database breaks the same guarantee. Ordinary operations can do this:

- restoring a backup
- deleting a corrupt or oversized `relay.db` (deleting the file is the only way to shrink `mailboxes`)
- moving to a new host
- re-creating a container started with the README's `docker run -v ./my-config.yaml:/etc/relay/config.yaml:ro ghcr.io/aossie-org/thrubox-server`, which mounts nothing on `/data`
- redeploying on a platform whose filesystem does not persist by default

Clients that never send `after` re-download the whole mailbox and are not affected.

### Steps to Reproduce

Build the relay with `go build -o relay ./cmd/relay`.

**A. Fresh database**

```bash
# Run 1, database A
RELAY_SERVER_PORT=38220 RELAY_STORAGE_PATH=./a/relay.db ./relay &
for p in old1 old2 old3; do
  curl -s -o /dev/null -w "%{http_code} " -X POST localhost:38220/api/messages \
    -H 'Content-Type: application/json' -d "{\"to\":\"0xbob\",\"from\":\"0xalice\",\"payload\":\"$p\"}"
done
curl -s localhost:38220/api/messages/0xbob   # seq 1, 2, 3 -> the client's cursor is 3
kill %1

# Run 2, empty database B (same address)
RELAY_SERVER_PORT=38220 RELAY_STORAGE_PATH=./b/relay.db ./relay &
# POST new1, new2 the same way, then:
curl -s "localhost:38220/api/messages/0xbob?after=3"
curl -s localhost:38220/api/messages/0xbob
# POST new3, new4, then:
curl -s "localhost:38220/api/messages/0xbob?after=3"
```

**B. Restored backup**

1. Send `m1` and `m2`, then stop the server and copy `relay.db` (`last_seq` is 2).
2. Start the server again and send `m3`, `m4` and `m5`. The client's cursor is now 5.
3. Stop the server and put the backup copy back.
4. Start the server, send `x1`, `x2` and `x3`, then request `?after=5`.

**C. With the SDK.** Run `relay.poll('0xbob', cb, { after: 0, intervalMs: 300 })` across a restart onto an empty database.

### Logs and Screenshots

A, fresh database:

```
POST old1..old3: 201 201 201
GET (no cursor): [(1, 'old1'), (2, 'old2'), (3, 'old3')]
-- restart on empty database --
POST new1,new2: 201 201
GET ?after=3:    []
GET (no cursor): [(1, 'new1'), (2, 'new2')]
GET /health:     {"status":"ok","timestamp":"2026-11-19T19:31:49Z"}
POST new3,new4: 201 201
GET ?after=3:    [(4, 'new4')]          <- new1, new2, new3 are never returned
```

B, restored backup (`seq` 3-5 reused for different messages):

```
GET: [(1, 'm1'), (2, 'm2'), (3, 'm3'), (4, 'm4'), (5, 'm5')]   client cursor = 5
-- backup restored --
POST x1..x3: 201 201 201
GET ?after=5: []
GET:          [(1, 'm1'), (2, 'm2'), (3, 'x1'), (4, 'x2'), (5, 'x3')]
```

C, `@aossie-org/thrubox-client` 1.1.0 `poll()` with a cursor:

```
0.1s delivered [[1,"old1"],[2,"old2"],[3,"old3"]]
1.6s onError RelayNetworkError        (server restarting)
1.9s onError RelayNetworkError
2.2s onError RelayNetworkError
4.3s delivered [[4,"new4"]]           <- new1..new3 never delivered, no error after reconnect
```

### Environment Details

- ThruBox Server `main` @ `f026f22` (`VERSION` 1.0.0), built locally with Go 1.24.6, Linux x86_64, default config
- ThruBox Client `main` @ `9afc707` (1.1.0), Node.js 24

### Impact

Medium - Feature works but has issues

### Proposed fix

The server is the source of truth for `seq`, so the fix starts here. The SDK then needs a follow-up. This changes the API, so it needs a maintainer decision on the shape.

1. **Database ID.** Create a random ID once, together with the schema: a one-row `meta` table filled with `crypto/rand`, so no new dependency. Return it on every message response, for example as an `X-ThruBox-Db-Id` header (no change to the JSON body), and in `/health`. Clients store `(dbId, cursor)` and reset the cursor to 0 when the ID changes. Content-hash `id`s make de-duplication after a reset cheap. A fresh database is then always detected.
2. **Cursor ahead of the counter.** A restored backup brings its old ID back with it, so (1) does not catch that case. Also return the mailbox's `last_seq`, for example as an `X-ThruBox-Last-Seq` header. A client whose cursor is greater than `last_seq` then knows its cursor is stale. A `409` for `after > last_seq` would also work, but SDK 1.1 would turn that into an endless `onError` loop, so a header is the safer choice. Document that whoever restores a backup should rotate the database ID.
3. **Docs now.** Qualify the "never reused" wording in the README, `store.go`, the CHANGELOG and the client README: it holds within one database, not across a reset or restore. Tell operators to persist `/data`.
4. **Tests.** Store tests: the database ID survives a reopen and differs between two new databases. Handler test: `after` above `last_seq`.
