# [FEATURE]: Let callers tell whether `send()` stored a new message or got back an existing one

### Feature and its Use Cases

ThruBox Server 1.0+ deduplicates sends by content. The message `id` is the SHA-256 of `to`, `from` and `payload`. A new message gets `201 Created`. An identical one gets `200 OK` with the **existing** message, which carries its original `seq` and `createdAt`.

`send()` returns only the parsed body (`src/client.ts:87-93`, `:266-271`), so the caller cannot tell which of the two happened. The README says "sending an identical message twice is safe: it is stored once and the existing message is returned", but an app has no way to observe it.

Why that matters:

- **A repeated payload is not a new message.** Sending the same payload to the same recipient later does not add anything to the mailbox. In the repro below, the third `send()` resolves normally with `seq: 1`, and the mailbox still holds only `[1, 2]`. This affects apps whose payloads can repeat: deterministic encryption, signed acks or receipts, or plaintext payloads during development. The repeated message is silently not delivered as new. The app gets no signal that it should, for example, add a nonce and send again.
- **Deduplicated retries.** Once opt-in send retries land (#82 / PR #83), apps may want to log or count retries that the server deduplicated.
- **UI.** An app may want to show "already sent" rather than "sent".

**Proposal** (no breaking change):

- Add a method such as `sendWithStatus(params): Promise<{ message: Message; created: boolean }>`. The name is open for discussion. Implement `send()` on top of it, with its signature unchanged.
- Set `created` to `response.status === 201`. This needs the private `request()` helper to return the HTTP status along with the parsed body.
- Document two edge cases:
  - Servers older than 1.0 always answer `201`, so `created` is always `true` there.
  - With send retries enabled, `created: false` can also mean that an earlier attempt of the same call stored the message.
- Add tests (201 → `true`, 200 → `false`, 413/429 still throw) and a short note in the README `send()` section.

### Additional Context

Repro against ThruBox-Server `main` (`f026f22`) with `@aossie-org/thrubox-client` 1.1.0 (`main` at `9afc707`), Node v24.22.1:

```js
import { RelayClient } from '@aossie-org/thrubox-client';
const relay = new RelayClient('http://127.0.0.1:3000', { retries: 0 });
const to = '0xbob-' + Date.now().toString(36);
const first = await relay.send({ to, from: '0xalice', payload: 'b2s=' });
await relay.send({ to, from: '0xalice', payload: 'other' });
const again = await relay.send({ to, from: '0xalice', payload: 'b2s=' }); // same payload, sent again later
const box = await relay.receive(to);
console.log(JSON.stringify({
  first: { seq: first.seq, createdAt: first.createdAt },
  again: { seq: again.seq, createdAt: again.createdAt },
  mailboxSeqs: box.map((m) => m.seq),
  returnedKeys: Object.keys(again),
}));
```

Output:

```
{"first":{"seq":1,"createdAt":"2026-11-19T18:21:54.48653662Z"},
 "again":{"seq":1,"createdAt":"2026-11-19T18:21:54Z"},
 "mailboxSeqs":[1,2],
 "returnedKeys":["id","seq","to","from","payload","createdAt","expiresAt"]}
```

On the wire, the same two sends return `201` and then `200`. The two response bodies differ only in timestamp precision, which is a separate server-side inconsistency. They contain no field that says whether a new message was stored.

Overlap: #82 and PR #83 add opt-in retries for `send()` but do not expose the status. This request complements them.
