# [BUG]: Purging a large expired backlog blocks every send until it finishes; after 5 s, POST /api/messages returns 500 "database is locked"

## Bug Description

`PurgeExpired` (`internal/store/sqlite.go:282-297`) removes every expired message with a single autocommit statement:

```go
DELETE FROM messages WHERE expires_at IS NOT NULL AND expires_at < ?
```

SQLite holds the database write lock until that statement finishes. Each `SaveMessage` starts with `BEGIN IMMEDIATE` (`_txlock=immediate`, `sqlite.go:36`), so every send that arrives during the purge waits for it. Reads are not affected, because the database runs in WAL mode.

If the purge runs longer than `_busy_timeout=5000`, a waiting send gets `SQLITE_BUSY`. `SaveMessage` returns `saving message: database is locked`, and `HandleCreate` (`internal/handler/message.go:75-79`) answers `500 internal server error`. The client SDK does not retry `send()` (`maxRetries = 0` for POST in `ThruBox-Client/src/client.ts:250`), so the user sees a failed send. Opt-in send retries are tracked in ThruBox-Client #82 and PR #83.

The purge runs once when the server starts (`cmd/relay/main.go:75-76,155-156`), at the same time as the HTTP server comes up, and then every hour. A large backlog builds up in two main ways:

- a relay restarts after some downtime. After a week or more offline, every stored message has expired;
- a busy relay has a lot of messages expiring within one hour.

How many rows it takes depends on the storage. On an NVMe SSD with 1 KB payloads, 400,000 expired rows took 3.8 s to purge. Sends waited up to 3.5 s but none failed. 700,000 rows took 7.3–7.8 s, and sends failed. Payload size barely matters: 10,000 rows of 64 KB were purged in 158 ms. The cost comes from the number of rows, because each deleted row also updates the primary-key index and three secondary indexes. Slower storage, such as an SD card or a network volume, should reach the 5 s limit with far fewer rows (not measured).

Roughly, the number of failed sends is the send rate multiplied by (purge time − 5 s). Nothing is half-written: the transaction fails at `BEGIN`, and resending the same message is safe because IDs are content-derived.

## Steps to Reproduce

1. On `main` (f026f22), append this temporary test to `internal/store/sqlite_test.go`. It reuses that file's imports and its `newMessage` helper, and it seeds rows through `s.db`:

```go
func TestPurgeBlocksSaveMessage(t *testing.T) {
	s, err := NewSQLiteStore(filepath.Join(t.TempDir(), "relay.db"))
	if err != nil {
		t.Fatal(err)
	}
	defer s.Close()

	// 700,000 messages with ~1 KB payloads that expired an hour ago (~1 GB on disk).
	expired := time.Now().UTC().Add(-time.Hour).Format(time.RFC3339)
	if _, err := s.db.Exec(`WITH RECURSIVE n(i) AS (SELECT 1 UNION ALL SELECT i+1 FROM n WHERE i < 700000)
		INSERT INTO messages (id, seq, to_addr, from_addr, payload, created_at, expires_at)
		SELECT printf('old-%d', i), i, printf('0xmb%d', i % 1000), '0xfrom', hex(randomblob(512)), ?, ? FROM n`,
		expired, expired); err != nil {
		t.Fatal(err)
	}

	done := make(chan string)
	go func() {
		start := time.Now()
		n, err := s.PurgeExpired(context.Background())
		done <- fmt.Sprintf("PurgeExpired: deleted %d rows in %s, err=%v", n, time.Since(start).Round(time.Millisecond), err)
	}()
	time.Sleep(100 * time.Millisecond) // let the purge take the write lock

	exp := time.Now().Add(7 * 24 * time.Hour)
	msg := newMessage("0xlive", "hello")
	msg.ExpiresAt = &exp
	start := time.Now()
	_, err = s.SaveMessage(context.Background(), msg)
	t.Logf("SaveMessage during purge: took %s, err=%v", time.Since(start).Round(time.Millisecond), err)
	t.Log(<-done)
}
```

2. Run it with the temp dir on a real disk (`/tmp` is often tmpfs, which is much faster):
   `TMPDIR=/var/tmp go test -count=1 -run TestPurgeBlocksSaveMessage -v ./internal/store/`
3. If the purge finishes in under 5 s on your machine, raise the row count.

Through HTTP: I wired the same handler chain as `cmd/relay/main.go` into `httptest`, with the rate limit raised so it would not interfere. With 8 concurrent senders and one reader running while `PurgeExpired` deleted 700,000 rows, every sender's in-flight POST got a 500. GETs kept working.

## Logs and Screenshots

Store-level test above:

```
ROWS=100000: SaveMessage during purge: took 330ms, err=<nil>
             PurgeExpired: deleted 100000 rows in 571ms, err=<nil>
ROWS=700000: SaveMessage during purge: took 5.005s, err=saving message: database is locked
             PurgeExpired: deleted 700000 rows in 7.788s, err=<nil>
```

HTTP-level run (8 senders POSTing every 5 ms, 1 reader):

```
PurgeExpired deleted 700000 rows in 7.332s
POST codes=map[201:2361 500:8] max=5.005s; GET codes=map[200:417] max GET=3.2ms
level=ERROR msg="failed to save message" error="saving message: database is locked"
```

The same HTTP run with the purge changed to delete in batches of 5,000 rows (see Proposed fix):

```
PurgeExpired deleted 700000 rows in 12.409s
POST codes=map[201:6057] max=1.23s; GET codes=map[200:653]; 'database is locked' = 0
```

## Environment Details

- ThruBox-Server `main` @ f026f22, `github.com/mattn/go-sqlite3` v1.14.52
- Go 1.24.6, Linux 6.17 x86_64, Intel i5-10300H (8 threads), ext4 on NVMe SSD, machine otherwise idle
- Default config (`ttl_days: 7`). Rate limit raised in the HTTP harness only, to stand in for many separate clients

## Impact

Low - Minor inconvenience. It only happens when one purge has a very large backlog to delete. Sends fail for a few seconds and can be resent safely. Reads and stored data are not affected.

## Proposed fix

- Purge in bounded batches, so the write lock is released between them and waiting sends get through:
  ```sql
  DELETE FROM messages WHERE rowid IN (
    SELECT rowid FROM messages
    WHERE expires_at IS NOT NULL AND expires_at < ? LIMIT ?)
  ```
  Loop until a batch deletes fewer rows than the limit. Compute the cutoff once, check `ctx` between batches, and return the total. A batch of 1,000–5,000 rows keeps each lock hold to tens of milliseconds. Add a test with a small batch size that checks every expired row is removed, forever (NULL) rows are kept, and the returned count is correct.
- Optional: when the store returns `SQLITE_BUSY`, answer `503` with `Retry-After: 1` instead of `500`. The store could expose a sentinel error so the handler does not import the driver. With ThruBox-Client PR #83, clients that opt in would then retry the send.
- Optional, for heavy write load: serialize write transactions in Go (a mutex in `SQLiteStore`). SQLite's busy handler polls with backoff and is not first-come-first-served. In a stress test with hundreds of concurrent senders and no rate limit, a few sends still reached the 5 s timeout even with no purge running.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
