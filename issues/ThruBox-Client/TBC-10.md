# [FEATURE]: Accept an `AbortSignal` in `send()`, `receive()` and `health()`

### Feature and its Use Cases

None of the public methods can be cancelled. `send(params)`, `receive(address, { after, limit })` and `health()` take no `signal` (`src/client.ts:87`, `:108`, `:124`). `request()` creates its own `AbortController` per attempt, only for the timeout (`src/client.ts:255-256`), and its `init` argument is private.

With the defaults (`timeout: 10000`, `retries: 3`), a `receive()` against a relay that accepts the connection but never answers runs for 4 timeouts plus 1 s + 2 s + 4 s of backoff plus up to 3 s of jitter. That is about 47-50 s, and the caller cannot stop it:

```js
// hang-repro.mjs: run from the repo root after `npm run build`
import net from 'node:net';
import { RelayClient } from './dist/index.js';

const socks = [];
const srv = net.createServer((s) => { socks.push(s); s.on('error', () => {}); }).listen(3002, async () => { // accepts, never answers
  const relay = new RelayClient('http://127.0.0.1:3002'); // defaults
  const ac = new AbortController();
  setTimeout(() => ac.abort(), 1000);
  const t0 = Date.now();
  try {
    await relay.receive('0xabc', { signal: ac.signal }); // not supported: the option is ignored
  } catch (e) {
    console.log(`rejected after ${((Date.now() - t0) / 1000).toFixed(1)}s: ${e.name}: ${e.message}`);
  }
  socks.forEach((s) => s.destroy()); srv.close();
});
```

Output (Node 24, `main` at 9afc707):

```
rejected after 48.9s: RelayNetworkError: Request timed out
```

Typical places where a caller needs to cancel:

- a React effect cleanup or a route change, where the result is no longer needed;
- a server handler with its own deadline, which wants to give up on the relay and answer its own client;
- tests and graceful shutdown in Node, where open sockets and timers keep the process alive.

Today the only options are a lower `timeout`/`retries` for the whole client, or ignoring the result while the requests keep running.

**Proposed API (additive, no breaking change)**

- `send(params, options?: { signal?: AbortSignal })`
- `receive(address, { after?, limit?, signal? })`, with `signal` added to `ReceiveOptions`
- `health(options?: { signal?: AbortSignal })`

**Behaviour**

- Before each attempt, check `signal.aborted` and stop.
- Abort the in-flight `fetch` when the signal fires. Link it to the per-attempt timeout controller with an `abort` event listener rather than `AbortSignal.any()`, which needs Node 18.17+ / Safari 17.4+ while `engines` allows Node `>=18.0.0`. Remove the listener after each attempt.
- Make the backoff sleep end as soon as the signal fires, and clear its timer.
- Never retry after a caller abort. Re-throw the caller's abort (`signal.reason`, normally a `DOMException` named `AbortError`) instead of wrapping it as `RelayNetworkError("Request timed out")`, so callers can tell "I cancelled" from "the relay timed out".
- Tests with fake timers: abort before the call, during a pending fetch, and during the backoff sleep. Each should make no further `fetch` calls and reject with the abort reason.

### Additional Context

- `poll()`'s stop function has the same problem internally (it cannot cancel the in-flight `receive()` and its retries). Both need the same plumbing through `request()`, so they could be done in one PR, or this one right after the `poll()` fix.
- Optional per-call `timeout`/`retries` overrides could use the same options object later. They are not part of this request.

### Code of Conduct

- [ ] I have joined the Discord server and will post updates there
- [ ] I have searched existing issues to avoid duplicates
