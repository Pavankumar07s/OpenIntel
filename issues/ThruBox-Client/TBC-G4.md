# [BUG]: Unread 401/413/429/5xx response bodies keep sockets open on Node until GC (bodies of 64 KiB or more)

## Bug Description

`RelayClient.request()` in `src/client.ts` never reads or cancels `response.body` on four error paths:

- 401: `throw new RelayUnauthorizedError()` (lines 275-277)
- 413: `throw new RelayPayloadTooLargeError()` (lines 284-286)
- 429: `throw new RelayRateLimitError(retryAfter)` (lines 288-303)
- 5xx: `lastError = new RelayError(...)`, then a retry (lines 311-315)

Only the 404 and generic 4xx paths call `response.text()` (lines 280 and 307).

On Node, `fetch` (undici) buffers about 64 KiB of a response body. If the error body is smaller than that, undici reads all of it and the connection goes back to the pool, so small bodies are not affected. If the body is 64 KiB or larger, undici pauses the socket once its buffer fills. The socket then cannot be reused or closed until the `Response` object is garbage-collected. The client keeps the socket's file descriptor during that time, and it is still held (in `CLOSE_WAIT`) after the server closes its side. Each later request opens a new connection, including every GET retry after a 5xx.

The undici README ("Garbage Collection") recommends always consuming or cancelling the response body, because Node's GC is not deterministic enough to release connections promptly.

When this happens: ThruBox-Server's own error bodies are short `http.Error` strings, so a client talking straight to it is not affected. It happens when something in front of the relay returns a large error page, such as a reverse proxy, CDN, hosting platform or maintenance page, or when the relay is a different implementation. Sockets are released at the next GC, so the effect is a temporary burst of held connections, not an unbounded leak.

## Steps to Reproduce

1. `npm run build`
2. Save as `repro.mjs` in the repo root:

```js
import http from 'node:http';
import { RelayClient } from './dist/index.js';

const status = Number(process.argv[2] ?? 503);
const size = Number(process.argv[3] ?? 200_000);
const body = Buffer.alloc(size, 'x');

const server = http.createServer((req, res) => {
  res.writeHead(status, { 'Content-Length': String(size) });
  res.end(body);
});
await new Promise((r) => server.listen(0, '127.0.0.1', r));

const relay = new RelayClient(`http://127.0.0.1:${server.address().port}`, { retries: 0 });
for (let i = 0; i < 20; i++) {
  try { await relay.receive('0xA'); } catch (e) { var name = e.constructor.name; }
}
await new Promise((r) => setTimeout(r, 300));
server.getConnections((_, n) => {
  console.log(`status=${status} body=${size}B -> ${name}, open sockets after 20 calls: ${n}`);
  server.closeAllConnections();
  server.close();
});
```

3. Run `node repro.mjs 503 200000`, then the same with `401`, `429`, `413`, and with `404` or a 60000-byte body for comparison.

**Expected:** each failed call releases its connection, so only the 1-2 pooled keep-alive sockets stay open, the same as the 404 path.

**Actual:** every call leaves its socket open when the body is 64 KiB or larger and the status is 401, 413, 429 or 5xx.

## Logs and Screenshots

On current `main` (9afc707):

```
status=503 body=200000B -> RelayError, open sockets after 20 calls: 20
status=503 body=60000B -> RelayError, open sockets after 20 calls: 2
status=401 body=200000B -> RelayUnauthorizedError, open sockets after 20 calls: 20
status=429 body=200000B -> RelayRateLimitError, open sockets after 20 calls: 20
status=413 body=200000B -> RelayPayloadTooLargeError, open sockets after 20 calls: 20
status=404 body=200000B -> RelayNotFoundError, open sockets after 20 calls: 2
```

The threshold is the 64 KiB buffer: a 65000-byte body leaves 2 sockets open, and a 65536-byte body leaves 20. Chunked responses with no `Content-Length` behave the same. After `global.gc()` (`node --expose-gc`) the count drops to 0. Counting socket file descriptors inside the client process gives 20 held after 20 calls, still 20 more than 5 s after the server closed its side (half of them in `CLOSE-WAIT`), and 0 after GC.

With `poll()` and default options against a 100 KB 503, the server saw between 0 and 3 sockets open at any 5 s sample over 60 s. The held sockets pile up between GCs, but the count stays bounded.

## Environment Details

- OS: Linux 6.17
- Node.js v24.22.1 (built-in `fetch`/undici)
- `@aossie-org/thrubox-client` at `main` 9afc707 (1.1.0)
- Browsers manage connections themselves, so the visible effect is on Node.

## Impact

Low - Minor inconvenience

## Proposed fix

Release the body before throwing or recording `lastError` on the four paths:

```ts
await response.body?.cancel().catch(() => {});
```

Add it before `throw new RelayUnauthorizedError()`, `throw new RelayPayloadTooLargeError()`, `throw new RelayRateLimitError(retryAfter)`, and before `lastError = new RelayError(...)` in the 5xx branch. `cancel()` is better than `text()` here because it does not download an arbitrarily large error page. If the 5xx message should include the server text (as the 4xx path does), read a bounded amount of it instead.

With this change, the repro above leaves 0 sockets open for 401, 413, 429, 500 and 503, and the existing 30 tests still pass. A regression test can mock `fetch` with `new Response("...", { status })` for each status and assert `response.bodyUsed === true` after the call rejects. That test fails on `main` and passes with the fix.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
