# [FEATURE]: Configurable storage limits: nothing caps how much one mailbox or the whole database can hold

## Feature and its Use Cases

**Current behaviour**

The only abuse control on `POST /api/messages` is the per-IP rate limiter (`security.rate_limit`, 30 requests/min by default). Every request it lets through is stored until its TTL runs out (`messages.ttl_days`, 7 days by default; with `ttl_days: 0` it is never removed). Nothing limits how many messages or bytes one recipient address can hold, or how large the database can get.

With the default config, one client IP can add 30 messages of up to 512 KiB each (`max_payload_size: 524288`) to **any single address** per minute. That is about 15 MiB/min, or about 21 GiB/day, kept for 7 days. Each additional source IP adds the same again. Anyone can post to any address, and `from` is not authenticated.

**Reproduce** (`main` @ `f026f22`)

```sh
go build -o relay-server ./cmd/relay

# 1. Defaults: the per-IP limiter is the only limit
RELAY_CONFIG_PATH=/nonexistent RELAY_SERVER_HOST=127.0.0.1 RELAY_SERVER_PORT=3000 \
  RELAY_STORAGE_PATH=/tmp/quota/relay.db ./relay-server &
for i in $(seq 1 35); do
  curl -s -o /dev/null -w '%{http_code}\n' -H 'Content-Type: application/json' \
    -d "{\"to\":\"0xVictim\",\"from\":\"0xA\",\"payload\":\"p$i\"}" http://127.0.0.1:3000/api/messages
done | sort | uniq -c
#   30 201
#    5 429
```

To stand in for many source IPs, restart with `RELAY_SECURITY_RATE_LIMIT=100000` and send 100 bodies of about 380 KB to the same address. All 100 return `201`, and `relay.db` grows to about 38 MB. Nothing rejects them at any size.

**Proposal** (open for discussion before any code is written)

1. **Global cap (recommended first).** Add `storage.max_db_bytes` (or a maximum total message count), with `0` meaning unlimited so existing deployments keep working. When the cap is reached, reject new messages with `507 Insufficient Storage` and a clear body. This protects the disk, and an attacker cannot use it to aim at one mailbox.
2. **Optional per-mailbox caps.** `messages.max_per_mailbox` and/or `messages.max_bytes_per_mailbox`, default `0` (unlimited). Check them inside the existing `BEGIN IMMEDIATE` transaction in `SQLiteStore.SaveMessage` by counting or summing the recipient's live messages, so concurrent sends cannot overshoot the cap.
   - **Trade-off to decide:** anyone can post to any address and `from` is not authenticated. So once a mailbox is full, an attacker who filled it blocks that recipient's real mail until the messages expire. That is why this should be opt-in and documented, not on by default.
3. Document the new keys in the README configuration table and `config.yaml`. Add a matching error class to ThruBox-Client in a follow-up.
4. Tests: cap reached → `507`, duplicate send of an already-stored message still returns `200`, expired messages do not count, plus a concurrency test along the lines of `TestSaveMessage_ConcurrentSeqsAreUnique`.

## Additional Context

- Relevant code: `internal/handler/message.go` (`HandleCreate`), `internal/store/sqlite.go` (`SaveMessage`, `PurgeExpired`), `internal/config/config.go`.
- Environment: Linux, built with Go 1.24.6 (`go.mod` says `go 1.23.12`), ThruBox-Server `VERSION` 1.0.0.
- I could not find an existing issue or PR about storage or mailbox limits.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
