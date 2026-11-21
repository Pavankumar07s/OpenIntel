# `POST /api/messages` accepts addresses of any length: a ~500 KB `to` is stored 5 times, and 2 copies are never removed by the TTL purge

## Bug Description

`CreateMessageRequest.Validate` (`internal/model/message.go:46-58`) only rejects empty fields. Nothing limits the length or content of `to` and `from`. The only bound is the request-body cap (`max_payload_size`, 524288 by default), so one request can carry a recipient address of about 500 KB.

The recipient address is stored five times:

- `messages.to_addr`
- the two indexes on it (`idx_messages_to_addr`, `idx_messages_to_addr_seq`)
- `mailboxes.to_addr`
- the `mailboxes` primary-key index

`PurgeExpired` (`internal/store/sqlite.go:282-297`) deletes only from `messages`. The `mailboxes` row is kept on purpose, so that `seq` is never reused. The README says sequence numbers "only go up and are never reused, even after a message expires". With normal addresses a `mailboxes` row is about 100 bytes, so keeping it is fine. With no length cap, though, every distinct long `to` leaves about 1 MB on disk that `ttl_days` never frees.

With the default config (no API key, `rate_limit: 30`, `ttl_days: 7`), 10 such requests used 24 MB of database pages. After the purge, 9.6 MB was still in use. At 30 requests per minute, that is about 29 MB per minute, or roughly 42 GB per day per client IP, that stays on disk for good.

The same function has some smaller validation gaps:

- **Whitespace and control characters are accepted.** `{"to":"   ","from":" ","payload":" "}` returns `201`. So does a `to` that contains a newline or a NUL byte. `" 0xabc "` and `"0xabc"` are stored as two different mailboxes.
- **`.` and `..` are accepted as addresses.**
  - `GET /api/messages/..` answers `301` to `/api`, and `GET /api/messages/.` answers `301` to `/api/messages`. These mailboxes can be read only through the percent-encoded forms (`/api/messages/%2E%2E`, `/api/messages/%2E`).
  - The ThruBox client builds the path with `encodeURIComponent`, which does not encode dots, and the WHATWG `URL` parser resolves the dot segment before the request is sent. So these mailboxes cannot be reached from the SDK or from a browser.

**Related:** `max_payload_size` limits the whole JSON request body (`internal/handler/message.go:35`), but the README table calls it "Max payload" and `config.yaml` says "500KB max payload". With `max_payload_size: 1000` and 42-character addresses, a 884-byte `payload` returns `201` and a 885-byte `payload` returns `413 payload too large`.

## Steps to Reproduce

1. Build and start the relay with the default `config.yaml`:
   ```sh
   go build -o relay ./cmd/relay
   RELAY_STORAGE_PATH=/tmp/tb/relay.db RELAY_SERVER_PORT=3000 ./relay
   ```
2. Send 10 messages, each to a different 500,001-character address:
   ```sh
   for i in 0 1 2 3 4 5 6 7 8 9; do
     python3 -c "import json; print(json.dumps({'to':'Z'*500000+'$i','from':'0xSender','payload':'cGF5bG9hZA=='}))" > big.json
     curl -s -o /dev/null -w "%{http_code}\n" -H 'Content-Type: application/json' --data-binary @big.json localhost:3000/api/messages
   done
   ```
   Every request returns `201`.
3. Stop the relay. Simulate the TTL running out, then start the relay again. `purgeLoop` runs `PurgeExpired` once at startup.
   ```sh
   sqlite3 /tmp/tb/relay.db "UPDATE messages SET expires_at='2000-01-01T00:00:00Z';"
   ```
4. Stop the relay and inspect the database:
   ```sh
   sqlite3 /tmp/tb/relay.db "SELECT COUNT(*) FROM messages; SELECT COUNT(*), SUM(LENGTH(to_addr)) FROM mailboxes;"
   sqlite3 /tmp/tb/relay.db "SELECT name, SUM(pgsize)/1024 FROM dbstat GROUP BY name ORDER BY 2 DESC LIMIT 5;"
   ```
5. Address edge cases (relay running):
   ```sh
   curl -s -w ' %{http_code}\n' -H 'Content-Type: application/json' -d '{"to":"..","from":"0xS","payload":"p"}' localhost:3000/api/messages   # 201
   curl -s --path-as-is -o /dev/null -w '%{http_code} %{redirect_url}\n' localhost:3000/api/messages/..                                  # 301 .../api
   curl -s -w ' %{http_code}\n' -H 'Content-Type: application/json' -d '{"to":"   ","from":" ","payload":" "}' localhost:3000/api/messages # 201
   ```

**Expected:** a `to` or `from` above a sane length, or one made only of whitespace, control characters, `.` or `..`, is rejected with `400`. The disk space a message uses is reclaimed (apart from a small, fixed-size counter row) once it expires.

**Actual:** all of these requests are accepted. About 1 MB per long-address request stays in `mailboxes` and its index forever.

## Logs and Screenshots

```
before purge: messages=10, mailboxes rows=10, SUM(LENGTH(mailboxes.to_addr))=5000010, live DB pages=24476 KB
   messages: 4896 KB | sqlite_autoindex_mailboxes_1: 4892 KB | mailboxes: 4892 KB
   idx_messages_to_addr_seq: 4892 KB | idx_messages_to_addr: 4892 KB
msg="purged expired messages" count=10
after purge:  messages=0, mailboxes rows=10, SUM(LENGTH(mailboxes.to_addr))=5000010, live DB pages=9808 KB
   sqlite_autoindex_mailboxes_1: 4892 KB | mailboxes: 4892 KB | messages: 4 KB

POST {"to":".."} -> 201
GET /api/messages/..     -> 301 Location: /api
GET /api/messages/%2E%2E -> 200 [{"seq":1,"to":"..",...}]
POST {"to":"   ","from":" ","payload":" "} -> 201

max_payload_size=1000, to/from = 42 chars:
payload=884 bytes (body 1000) -> 201
payload=885 bytes (body 1001) -> 413 payload too large
```

## Environment Details

- ThruBox-Server `main` @ `f026f22` (VERSION 1.0.0), default `config.yaml`
- Linux, built with Go 1.24.6 (`go.mod` declares 1.23.12)
- `sqlite3` CLI 3.51 used only to inspect the database

## Impact

Medium - Feature works but has issues

## Proposed fix

- In `CreateMessageRequest.Validate`:
  - Add a maximum length for `to` and `from`, for example a `MaxAddressLen = 256` constant.
  - Reject values that are empty after trimming, that contain control characters or surrounding whitespace, or that are exactly `.` or `..`.
  - Return `400` with a clear message.
  - Keep the check format-agnostic, without enforcing `0x` + hex, because the relay is meant for any application.
- Make the payload limit match its name. Either check `len(req.Payload) <= MaxPayloadSize` after decoding and set the body cap to `MaxPayloadSize` plus a small envelope allowance (`2*MaxAddressLen` + about 1 KiB), or keep the current behaviour and document the setting as the maximum request body size.
- Add tests in `internal/model/message_test.go` and `internal/handler/message_test.go`.
- Leave `mailboxes` rows alone. Pruning them would restart `seq` at 1 and break saved client cursors. With a length cap, each row has a small, bounded size.
