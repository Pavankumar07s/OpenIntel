# [FEATURE]: Add a page size (`limit`) to `poll()` and fetch the next page right away when a page is full

### Feature and its Use Cases

**Problem.** `poll()` calls `receive(address, { after: cursor })` without a `limit` (`src/client.ts:189-192`). The first poll, or the first poll after the app was offline, downloads the whole backlog in one response. If the client cannot download that response within its `timeout` (10 s by default), the request is aborted and the cursor does not move. The next poll asks for `after=0` again and times out again. Nothing is ever delivered, even though the server can return the same mailbox page by page (`limit`, 1-1000).

How large the backlog must be depends on the link: with the 10 s default timeout it is about bandwidth x 10 s, so roughly 2 MB at 1.6 Mbit/s. The server accepts payloads up to 500 KB by default (`max_payload_size: 524288`), so a few messages with attachments are enough. With the default `retries: 3`, each failed poll cycle also takes about 50 s instead of 10 s.

**Reproduction** (Node 24, ThruBox Server `main` at f026f22 started with `RELAY_SECURITY_RATE_LIMIT=100000` on port 3000, SDK built from `main` at 9afc707):

```js
// backlog-repro.mjs: run from the repo root after `npm run build`
import net from 'node:net';
import { RelayClient } from './dist/index.js';

// 1. A TCP proxy on :3001 that limits server->client traffic to 100 KB/s (~0.8 Mbit/s)
net.createServer((c) => {
  const up = net.connect(3000, '127.0.0.1'); c.pipe(up);
  const q = []; let ended = false;
  up.on('data', (d) => q.push(d)); up.on('end', () => (ended = true));
  const t = setInterval(() => {
    let budget = 5000;
    while (budget > 0 && q.length) {
      const chunk = q[0];
      if (chunk.length <= budget) { c.write(q.shift()); budget -= chunk.length; }
      else { c.write(chunk.subarray(0, budget)); q[0] = chunk.subarray(budget); budget = 0; }
    }
    if (ended && !q.length) { clearInterval(t); c.end(); }
  }, 50);
  const kill = () => { clearInterval(t); up.destroy(); c.destroy(); };
  c.on('error', kill); up.on('error', kill); c.on('close', kill);
}).listen(3001);

// 2. Six 400 KB messages (about 2.4 MB) for one address, sent directly to the relay
const direct = new RelayClient('http://127.0.0.1:3000');
for (let i = 0; i < 6; i++) {
  await direct.send({ to: '0xbacklog', from: '0xsender', payload: String(i).padStart(4, '0') + 'A'.repeat(399996) });
}

// 3. Poll through the slow link for 32 s (retries: 0 only keeps the run short)
const relay = new RelayClient('http://127.0.0.1:3001', { retries: 0 });
const delivered = [], errors = [];
const stop = relay.poll('0xbacklog', (m) => delivered.push(m.length), {
  after: 0, intervalMs: 500, onError: (e) => errors.push(e.message),
});
await new Promise((r) => setTimeout(r, 32_000));
stop();

// 4. The same mailbox, one page at a time, over the same link
const t0 = Date.now();
const page = await relay.receive('0xbacklog', { after: 0, limit: 1 });
console.log({ delivered, errors, pagedReceive: { got: page.length, seq: page[0].seq, ms: Date.now() - t0 } });
process.exit(0);
```

Output:

```
{ delivered: [],
  errors: [ 'Request timed out', 'Request timed out', 'Request timed out' ],
  pagedReceive: { got: 1, seq: 1, ms: 4064 } }
```

Every poll requested `?after=0` and nothing was delivered. A single `receive(..., { after: 0, limit: 1 })` over the same link returned in about 4 s.

**Proposed change**

- Add `PollOptions.limit?: number`. It is only used together with `after`, and sent as `limit` on every poll.
- When a page comes back full (`messages.length === limit`) and the callback succeeds, advance the cursor and fetch the next page straight away instead of waiting `intervalMs`. Wait `intervalMs` only after a short page. Keep the existing rule that the cursor only moves after the callback succeeds.
- Choose a default together with the maintainers. Leaving `limit` unset keeps today's behaviour exactly. A small default page size when `after` is set (for example 50) fixes the problem for apps that never set `limit`. A fixed message count does not bound the response size, because each message can be up to 500 KB.
- Leave `poll()` without `after` unchanged. Without a cursor it cannot page.
- Document the option in the README `poll()` table, next to the existing note on `receive()` that "a full page means there may be more".
- Add tests with fake timers: a full page triggers an immediate next fetch with the new `after`, a short page waits `intervalMs`, and a callback that throws on page 2 does not skip page 2.

### Additional Context

- The cursor feature came from #64 / #63. This adds paging to it.
- On the server side, `GET /api/messages/{address}` has no default page size when `limit` is omitted. If the server adds one, `poll()` will need the "fetch the next page straight away" behaviour, otherwise a backlog larger than one page is only delivered one page every `intervalMs`.
- A user-set per-call `timeout` (as an alternative way out) is a separate, larger change and is not required for this one.

### Code of Conduct

- [ ] I have joined the Discord server and will post updates there
- [ ] I have searched existing issues to avoid duplicates
