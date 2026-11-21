# [BUG]: `poll()` loop promise is never caught: a non-array response (or an unstringifiable throw) exits the Node process and stops polling

### Bug Description

`poll()` starts its loop with a bare `doPoll();` (`src/client.ts:222`) and schedules the next run with `setTimeout(doPoll, intervalMs)` (`:217`). Neither call attaches a `.catch`. Most failures are caught inside `doPoll`. Since #63, a throwing callback or a throwing `onError` no longer stops the loop. But a few code paths still throw outside those `try` blocks. When one does, the promise `doPoll` returns rejects and nothing handles it:

- **Node.js (>= 15, so every supported version):** the default `--unhandled-rejections=throw` mode ends the process with exit code 1.
- **Browsers:** the rejection is only logged, but no next poll is scheduled, so polling stops silently. `onError` is never called.

Paths that can still throw outside the guarded blocks:

1. **The server answers 200 with JSON that is not an array, and `after` is set.** The cursor loop `for (const m of messages)` (`:202`) throws `TypeError: messages is not iterable`. A JSON element that is `null` makes `m.seq` (`:203`) throw in the same way. `receive()` does not check the response shape, so any server that answers `GET /api/messages/{address}` with an object triggers this. Examples: a wrong `baseUrl`, a proxy or gateway that returns a JSON body with status 200, or a future server version that wraps the list in an object. ThruBox-Server itself always returns an array, so this needs a misbehaving or misconfigured endpoint. It is still the case most likely to happen in practice. Related shapes behave badly too, even without a crash:
   - A body of `null` is skipped by `if (!stopped && messages)` (`:197`). Polling keeps running, but neither the callback nor `onError` is called, so a broken endpoint looks exactly like an empty mailbox.
   - Without `after`, the callback receives the raw object (typed as `Message[]`), and `onError` is never called.

   This is separate from #80 / PR #81, which cover bodies that are not valid JSON (`SyntaxError`). Here the body is valid JSON with the wrong shape, and PR #81 does not check the shape.
2. **The callback throws a value that `String()` cannot convert**, such as `Object.create(null)` or an object whose `toString` throws. `new Error(String(err))` in `reportError` (`:171`) then throws.
3. **`console.error` throws**, for example when a test setup replaces it with a throwing function. The fallback logging calls at `:177` and `:180` are not guarded.

### Steps to Reproduce

Path 1, with a stub server that returns a JSON object:

```js
// stub.mjs
import http from 'node:http';
http.createServer((req, res) => {
  res.setHeader('Content-Type', 'application/json');
  res.end(JSON.stringify({ messages: [] }));
}).listen(31120, '127.0.0.1');
```

```js
// poll.mjs
import { RelayClient } from '@aossie-org/thrubox-client';
const relay = new RelayClient('http://127.0.0.1:31120', { retries: 0 });
relay.poll('0xabc', () => {}, {
  intervalMs: 100,
  after: 0,
  onError: (e) => console.log('onError:', e.message),
});
setTimeout(() => process.exit(0), 1000);
```

Run `node stub.mjs &` and then `node poll.mjs; echo "exit $?"`. A body of `[null]` crashes the same way (`TypeError: Cannot read properties of null (reading 'seq')`). With a body of `null`, or without `after`, the process stays alive but `onError` is never called.

Path 2, against any relay that returns `[]`:

```js
relay.poll('0xabc', () => { throw Object.create(null); }, {
  intervalMs: 100,
  onError: (e) => console.log('onError:', e.message),
});
```

**Expected:** the error goes to `onError` (or to `console.error` when there is no handler), and polling keeps running every `intervalMs`, as it already does when the callback throws an ordinary `Error`.

**Actual:** `onError` is never called. Only one request is made. Node exits with code 1. With `node --unhandled-rejections=warn`, which behaves like a browser here, the process stays alive, but no second poll ever happens.

### Logs and Screenshots

Path 1:
```
TypeError: messages is not iterable
    at doPoll (.../dist/index.js:183:27)
    at process.processTicksAndRejections (node:internal/process/task_queues:104:5)
Node.js v24.22.1
exit 1
```

Path 2:
```
TypeError: Cannot convert object to primitive value
    at String (<anonymous>)
    at reportError (.../dist/index.js:158:60)
    at doPoll (.../dist/index.js:191:11)
exit 1
```

Same test under Vitest, with fake timers: `expected "vi.fn()" to be called at least once` for `onError`, plus `Vitest caught 2 unhandled errors ... Unhandled Rejection: TypeError: messages is not iterable ❯ doPoll src/client.ts:202:27`.

For comparison, when the callback throws `new Error('normal')`, `onError` is called on every cycle and polling continues (6 polls in 550 ms at `intervalMs: 100`).

### Environment Details

- `@aossie-org/thrubox-client` 1.1.0 (`main` at `9afc707`), built with `npm run build`
- Node.js v24.22.1, Linux
- Applies to all supported Node versions (`engines: >=18`), where an unhandled rejection exits the process by default

### Impact

Low - Minor inconvenience. It needs an unexpected server response or an unusual thrown value, but when it happens a long-running Node process exits, or polling in a browser stops without any error reaching `onError`.

### Proposed fix

In `src/client.ts` (plus `src/errors.ts` if a dedicated error class is added). No new dependencies.

- Never leave the loop promise floating. Either wrap the body of `doPoll` in `try { ... } finally { if (!stopped) timeoutId = setTimeout(run, intervalMs); }` and start it with `const run = () => { doPoll().catch(reportError); }`, or catch, report and reschedule in one place. The aim is that an unexpected throw is reported **and** the loop continues.
- Check the response shape in `receive()`, so direct callers get a typed error too: unless the parsed body is an array whose elements are all non-null objects, throw a `RelayError` such as "Unexpected response from relay: expected an array of messages", with `statusCode` set to the real response status (or a dedicated typed error, per the typed-errors convention in AGENTS.md). `poll()` then reports it to `onError` and skips the callback for that cycle.
- Make `reportError` total. Wrap `String(err)` in `try/catch` with a fallback message, and wrap both `console.error` fallbacks in `try/catch`.
- Add tests next to the existing "should keep polling when onError itself throws" test: 200 bodies of `{}`, `null`, `[null]` and a number, for both `receive()` and `poll({ after: 0 })`, and a callback that throws `Object.create(null)`. Each `poll()` case should call `onError` and keep polling. A prototype of the changes above is about 30 lines. The existing 30 tests and `tsc --noEmit` still pass with it.
