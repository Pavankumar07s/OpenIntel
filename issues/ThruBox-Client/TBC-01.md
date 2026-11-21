# [BUG]: `poll()` stop function does not abort the in-flight request or its retries

### Bug Description

The function returned by `relay.poll()` only sets a `stopped` flag and clears the *next* scheduled poll (`src/client.ts:225-228`). It does not cancel the `receive()` call that is already running (`src/client.ts:189-192`).

If that request is failing (5xx, network error, or timeout) when `stop()` is called, `request()` continues through its whole retry loop. The backoff sleep is a plain `setTimeout` promise that nothing can cancel (`src/client.ts:337-342`). So after `stop()`:

- up to `retries` more GET requests are sent (3 by default), spread over roughly 7-10 s of backoff;
- an attempt that is already open is not aborted. It stays open until the per-request `timeout` (10 s by default) fires. With a relay that accepts connections but never answers, activity can continue for about 4 x 10 s plus the backoff;
- in Node, the pending timers and sockets keep the process alive for that whole time.

The results and errors from these requests are dropped silently: the callback and `onError` are not called after `stop()`. The only effects are the extra network traffic and the delay before the process can exit. This shows up when a component unmounts, a user logs out, or a test tears down while the relay is unreachable or returning errors.

When the relay is healthy, the in-flight GET finishes normally and its result is thrown away, and no extra requests are sent.

### Steps to Reproduce

1. `npm ci && npm run build` on `main` (9afc707).
2. Save the script below as `stop-repro.mjs` in the repo root and run `node stop-repro.mjs`. It starts a local server that always answers 503, starts polling, and calls `stop()` after 200 ms.

```js
import http from 'node:http';
import { RelayClient } from './dist/index.js';

const t0 = Date.now();
const ts = () => ((Date.now() - t0) / 1000).toFixed(2) + 's';
let hits = 0, stopped = false;

const server = http.createServer((req, res) => {
  hits++;
  console.log(`${ts()} hit #${hits} ${req.method} ${req.url}${stopped ? '  (after stop)' : ''}`);
  res.writeHead(503).end('unavailable');
});

server.listen(31100, '127.0.0.1', () => {
  const relay = new RelayClient('http://127.0.0.1:31100'); // default retries: 3
  const stop = relay.poll('0xabc', () => {}, { intervalMs: 60_000, onError: () => {} });
  setTimeout(() => { stopped = true; stop(); console.log(`${ts()} stop() called`); }, 200);
  setTimeout(() => { console.log(`${ts()} total hits: ${hits}`); server.close(); }, 12_000);
});
```

The same thing as a Vitest case (fails on `main`):

```ts
import { it, expect, vi } from "vitest";
import { RelayClient } from "../src/client";

it("sends no requests after stop() while a receive is retrying", async () => {
  vi.useFakeTimers();
  const fetchMock = vi.fn(async () => new Response("unavailable", { status: 503 }));
  vi.stubGlobal("fetch", fetchMock);
  const client = new RelayClient("http://relay.test");
  const stop = client.poll("0x1", vi.fn(), { intervalMs: 60_000, onError: vi.fn() });
  await vi.advanceTimersByTimeAsync(0); // first attempt got 503, client is in backoff
  stop();
  await vi.advanceTimersByTimeAsync(30_000);
  expect(fetchMock).toHaveBeenCalledTimes(1);
});
```

**Expected:** after `stop()`, no more requests are sent, the open request is aborted, and nothing keeps the process alive.

**Actual:** three more requests reach the server after `stop()`, and the Vitest case fails with `expected "vi.fn()" to be called 1 times, but got 4 times`.

### Logs and Screenshots

Output of the script above:

```
0.02s hit #1 GET /api/messages/0xabc
0.22s stop() called
1.63s hit #2 GET /api/messages/0xabc  (after stop)
4.09s hit #3 GET /api/messages/0xabc  (after stop)
8.88s hit #4 GET /api/messages/0xabc  (after stop)
12.02s total hits: 4
```

With nothing listening on the port (connection refused), a script that only calls `poll()` and then `stop()` at 0.14 s does not exit until 8.19 s, after 3 more `fetch` calls.

With a server that accepts the connection but never responds (`timeout: 2000, retries: 1`), the first request stays open after `stop()` until the client timeout closes it at 2.0 s, and a retry is then opened at 3.5 s.

### Environment Details

- `@aossie-org/thrubox-client` 1.1.0, `main` at 9afc707
- Node.js v24.22.1, Linux
- Affects any runtime: the code path is the same in browsers

### Impact

Low - Minor inconvenience

### Proposed fix

- Create an `AbortController` for each `poll()` call, and have the stop function call `controller.abort()`.
- Pass an internal `signal` from `doPoll` through `receive()` into `request()`. This can be a private parameter, so the public API does not change.
- In `request()`:
  - Link that signal to the per-attempt timeout controller with an `abort` event listener. `AbortSignal.any()` only exists from Node 18.17 and Safari 17.4, and `engines` allows `>=18.0.0`.
  - Check the signal before each attempt.
  - Make the backoff sleep resolve or reject as soon as the signal aborts, and clear its timer.
- Do not report an error to `onError` when the abort came from `stop()` (`reportError` already returns early once `stopped` is set).
- Add a regression test like the one above.

### Code of Conduct

- [ ] I have joined the Discord server and will post updates there
- [ ] I have searched existing issues to avoid duplicates
