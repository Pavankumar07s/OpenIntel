# [FEATURE]: Define how addresses are matched: the checksummed and lowercase forms of one `0x` address are two separate mailboxes

### Feature and its Use Cases

**Current behaviour**

The relay matches the recipient address byte for byte. `POST /api/messages` stores `to` exactly as it was sent (`internal/handler/message.go:60-64`). `GET /api/messages/{address}` looks it up with `WHERE to_addr = ?` (`internal/store/sqlite.go:249`), and the `mailboxes` table is keyed on the same string. The official client passes addresses through unchanged (`src/client.ts:90` and `:115` in ThruBox-Client).

So the EIP-55 checksummed form of an Ethereum address (`0x5aAe…BeAed`) and its lowercase form (`0x5aae…beaed`) are two different mailboxes, each with its own `seq` counter. Neither side gets an error: the reader just receives `[]` with `200 OK`.

Wallet tooling does not agree on casing. ethers and viem `getAddress()` return the checksummed form, while MetaMask's `eth_requestAccounts`, many indexers and many backends use lowercase. If an app takes the recipient's address from one source and the recipient polls with an address from another, messages are silently never delivered. Nothing in the README says that addresses are case-sensitive, exact-match keys. The README examples (`0xRecipient`) are not real addresses either, so integrators get no hint.

This is consistent with the relay's "dumb mailbox" design, and other chains use case-sensitive addresses (for example, base58 Solana addresses), so the server cannot simply lowercase everything. The matching rule does need to be decided and written down, though.

**Steps to reproduce** (relay on `main`, default config)

```bash
go build -o relay-server ./cmd/relay && ./relay-server &
B=http://localhost:3000
CS=0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed   # EIP-55 checksummed
LC=0x5aaeb6053f3e94c9b9a09f33669435e7ef1beaed   # same address, lowercase

curl -s -X POST $B/api/messages -H 'Content-Type: application/json' \
  -d "{\"to\":\"$CS\",\"from\":\"0xSender\",\"payload\":\"aGVsbG8=\"}"
curl -s $B/api/messages/$CS    # -> [{"id":"8e34…","seq":1,"to":"0x5aAeb6…",…}]
curl -s $B/api/messages/$LC    # -> []
```

Sending the same payload to `$LC` returns `201` with a different `id` and `seq: 1`. The `mailboxes` table then has two rows:

```
0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed|1
0x5aaeb6053f3e94c9b9a09f33669435e7ef1beaed|1
```

**Expected:** either the server treats the two spellings of one EVM address as the same mailbox, or the docs say clearly that addresses are opaque, case-sensitive keys and that clients must normalise them.

**Actual:** they are different mailboxes, and this is not documented anywhere.

**Environment:** ThruBox-Server `main` @ `f026f22` (VERSION 1.0.0), Go 1.23.12, Linux. Tested with the built binary on loopback and curl.

### Proposed fix

This needs a maintainer decision, so here are two options:

- **Option A: normalise on the server.** In `HandleCreate` and `HandleGetByAddress`, lowercase any `to` or `{address}` that matches `^0x[0-9a-fA-F]{40}$`, before `ContentID` is computed and before the store is called. Leave every other address untouched. The open question is existing data. Mixed-case rows written before the change would no longer be found under their lowercase key. Merging them would mean renumbering `seq`, which breaks the "never reused, only goes up" promise for clients that already hold a cursor. The simplest choice is to leave old rows alone and let the TTL purge them (7 days by default), and to put that in the changelog. It could also sit behind a config switch.
- **Option B: document it.** State in the README's API section that addresses are opaque, exact-match and case-sensitive, and that apps should normalise EVM addresses (for example to lowercase) before they send and read. Add the same note to the ThruBox-Client README, and optionally an address-normaliser option in the SDK.

Either way, add a handler test that pins the chosen behaviour (POST with the checksummed form, then GET with the lowercase form).

### Additional Context

- The same behaviour shows up from the client side: the ThruBox-Client SDK forwards addresses unchanged, so any normalisation is left to each app.
- Related: `Validate` (`internal/model/message.go:46-58`) only checks that `to` and `from` are not empty. Option A would go next to any future address validation.

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
