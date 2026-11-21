<!-- Team note, not issue text: do not file. Merged into TBC-03.md; this file is kept only as evidence. -->

# [BUG]: `receive()` does not check that a 200 response is an array, so `poll()` with `after` crashes the Node process instead of calling `onError`

## Bug Description

`receive()` (`src/client.ts:114-116`) returns whatever JSON a 2xx response contains, cast to `Message[]` (`request()` does `(await response.json()) as T` at `:271`). Nothing checks the shape. The ThruBox server always returns an array (`make([]*model.Message, 0)`), so this never happens against a correctly configured relay. It does happen when `baseUrl` points at something that answers `GET /api/messages/{address}` with `200` and a JSON object, a number, `null`, or an array containing `null`. For example: a wrong `baseUrl`, a different JSON API, or a proxy/gateway that returns a JSON status object with 200.

What happens in `poll()` then:

- **With `after` set (cursor mode):** the cursor loop at `:200-205` (`for (const m of messages)` / `m.seq`) runs outside both `try/catch` blocks. A non-iterable body throws `TypeError: messages is not iterable`, and `[null]` throws `TypeError: Cannot read properties of null (reading 'seq')`. `doPoll()` is called without `await` or `.catch` (`:217`, `:222`), so the rejection is unhandled. On Node this terminates the process with exit code 1. `onError` is never called. In a browser, polling stops silently.
- **Body `null`:** `if (!stopped && messages)` (`:197`) skips it. Polling keeps running, but the callback and `onError` are never called, so a broken endpoint looks exactly like an empty mailbox.
- **Without `after`:** the callback is called with the raw object (typed as `Message[]`) and `onError` is never called.

This is separate from #80 / PR #81. Those cover bodies that are not valid JSON (`SyntaxError`). Here the body is valid JSON with the wrong shape, and PR #81 does not check the shape.

## Steps to Reproduce

1. Save as `repro.mjs` next to an install of `@aossie-org/thrubox-client` 1.1.0 (or a build of `main`):

   ```js
   import http from 'node:http';
   import { RelayClient } from '@aossie-org/thrubox-client';

   const body = process.argv[2];            // e.g. '{"error":"maintenance"}'
   const useCursor = process.argv[3] === '1';

   const srv = http.createServer((req, res) => {
     res.writeHead(200, { 'Content-Type': 'application/json' });
     res.end(body);
   });
   await new Promise((r) => srv.listen(8787, '127.0.0.1', r));

   let calls = 0; const errs = [];
   process.on('exit', (code) =>
     console.log(`exit=${code} callbacks=${calls} onError=${JSON.stringify(errs)}`));

   const relay = new RelayClient('http://127.0.0.1:8787', { retries: 0 });
   const stop = relay.poll('0xA', () => { calls++; }, {
     intervalMs: 100,
     onError: (e) => errs.push(e.message),
     ...(useCursor ? { after: 0 } : {}),
   });
   setTimeout(() => { stop(); srv.close(); }, 600);
   ```

2. Run `node repro.mjs '{"error":"maintenance"}' 1`, then `node repro.mjs '[null]' 1`, `node repro.mjs 'null' 1` and `node repro.mjs '{"error":"maintenance"}' 0`.

**Expected:** each poll reports a typed `RelayError` (for example "Unexpected response from relay: expected an array of messages") to `onError`, the callback is not called with a non-array, polling keeps running, and the process is not terminated.

**Actual:** see below.

## Logs and Screenshots

```
$ node repro.mjs '{"error":"maintenance"}' 1     # same for '{"messages":[]}' and '42'
TypeError: messages is not iterable
    at doPoll (.../dist/index.js:183:27)
exit=1 callbacks=0 onError=[]

$ node repro.mjs '[null]' 1
TypeError: Cannot read properties of null (reading 'seq')
    at doPoll (.../dist/index.js:184:26)
exit=1 callbacks=0 onError=[]

$ node repro.mjs 'null' 1
exit=0 callbacks=0 onError=[]

$ node repro.mjs '{"error":"maintenance"}' 0
exit=0 callbacks=6 onError=[]          # callback received a plain object
```

A body of `[]` behaves correctly (`exit=0 callbacks=6 onError=[]`).

## Environment Details

- `@aossie-org/thrubox-client` 1.1.0 / `main` at 9afc707
- Node.js 24.22.1, Linux
- Affects any runtime. On Node 15+ the unhandled rejection terminates the process. In browsers polling stops without calling `onError`.

## Impact

Low - Minor inconvenience. It requires an endpoint that is not a ThruBox server, or an intermediary that rewrites the response. When it does happen in cursor mode on Node, the whole host process exits.

## Proposed fix

- In `receive()`, check the parsed body before returning it. Throw a `RelayError` (statusCode = the response status, or a dedicated typed error, per the typed-errors convention in AGENTS.md) unless `Array.isArray(result)` and every element is a non-null object. Doing it in `receive()` (not in `poll()`) also protects direct `receive()` callers.
- I tried this locally. With the check, all four bodies above send "Unexpected response from relay: expected an array of messages" to `onError` on every cycle (6 times in 600 ms), polling keeps running, and the process exits 0. The existing 30 Vitest tests still pass, and send/receive/poll against a real ThruBox server are unchanged.
- Add Vitest cases for `{}`, `null`, `[null]` and a number, for both `receive()` and `poll({ after })`.
- Optionally apply the same kind of check to `send()` and `health()`. It may also make sense to use the same error for this as for the JSON parse failures in #80 / PR #81.
- Separately, attaching a `.catch` to the `doPoll()` calls would stop any other unexpected throw in the loop from becoming an unhandled rejection.
