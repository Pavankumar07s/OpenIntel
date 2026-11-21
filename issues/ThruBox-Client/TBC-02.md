# [BUG]: `timeout` and `intervalMs` above 2147483647 ms pass validation, then fire after ~1 ms (requests time out at once, `poll()` floods the relay)

## Bug Description

`RelayClient` checks `timeout` (`src/client.ts:67`) and `poll()`'s `intervalMs` (`src/client.ts:158`) only for "positive and finite". Both values are then passed straight to `setTimeout` (`src/client.ts:256` for the request abort timer, `src/client.ts:217` for the next poll).

`setTimeout` stores its delay as a signed 32-bit integer, so the largest delay it can hold is 2,147,483,647 ms (about 24.8 days):

- In Node.js, a larger delay is set to 1 ms, and Node prints `TimeoutOverflowWarning: ... Timeout duration was set to 1.`
- In browsers the delay wraps modulo 2^32. Values such as 2^31, 30 days or `Number.MAX_SAFE_INTEGER` fire at once. Other values fire after a shorter, unrelated delay: in Chrome, `2**32 + 5000` fires after 5 s.

What this means in practice:

- **`timeout` > 2^31-1**: every request is aborted after about 1 ms and throws `RelayNetworkError("Request timed out")`, even when the server answers in 20 ms. A user might pass a value like this to mean "no timeout": `timeout: Infinity` is rejected ("Timeout must be a positive finite number"), so `Number.MAX_SAFE_INTEGER` is the obvious next try, and it makes every request fail.
- **`intervalMs` > 2^31-1**: `poll()` re-polls about every 1 ms instead of waiting. Against a real ThruBox-Server with the default `rate_limit: 30`, the 30 allowed requests run out at once, and `onError` then gets hundreds of `RelayRateLimitError`s per second until `stop()` is called.

Values up to 2^31-1 work correctly.

## Steps to Reproduce

1. Save this as `repro.mjs` next to an install of `@aossie-org/thrubox-client` 1.1.0, or a build of `main`:

   ```js
   import http from 'node:http';
   import { RelayClient } from '@aossie-org/thrubox-client';

   let count = 0;
   let delay = 0; // how long the server waits before answering
   const server = http.createServer((req, res) => {
     count++;
     setTimeout(() => { res.writeHead(200, { 'Content-Type': 'application/json' }); res.end('[]'); }, delay);
   });
   await new Promise((r) => server.listen(31110, '127.0.0.1', r));
   const base = 'http://127.0.0.1:31110';
   const DAY = 24 * 3600 * 1000;

   // 1. Long poll interval
   const stop = new RelayClient(base, { retries: 0 })
     .poll('0xabc', () => {}, { intervalMs: 30 * DAY, onError: () => {} });
   await new Promise((r) => setTimeout(r, 500));
   stop();
   console.log('requests in 500 ms with intervalMs = 30 days:', count);

   // 2. Long request timeout (the server now answers after 20 ms)
   delay = 20;
   for (const timeout of [2 ** 31 - 1, 2 ** 31, Number.MAX_SAFE_INTEGER]) {
     const t0 = Date.now();
     try { await new RelayClient(base, { timeout, retries: 0 }).health(); console.log(timeout, 'OK'); }
     catch (e) { console.log(timeout, `${e.name}: ${e.message} after ${Date.now() - t0} ms`); }
   }
   server.close();
   ```

2. Run `node repro.mjs`.

**Expected:** one request in 500 ms (the next poll is 30 days away), and `health()` succeeds for every timeout value. Or, if such values are not supported, the constructor and `poll()` throw a validation error, the same way they already do for `0`, negative values and `Infinity`.

**Actual:** more than 100 requests in 500 ms (131 to 152 in my runs), and every timeout above 2^31-1 aborts after 1 to 4 ms.

## Logs and Screenshots

Output of the script above on `main` (`9afc707`), Node v24.22.1, run with `node --no-warnings repro.mjs`:

```
requests in 500 ms with intervalMs = 30 days: 131
2147483647 OK
2147483648 RelayNetworkError: Request timed out after 1 ms
9007199254740991 RelayNetworkError: Request timed out after 4 ms
```

With `intervalMs: 2 ** 31 - 1` the same script makes exactly 1 request.

Without `--no-warnings`, Node prints this many times:

```
(node) TimeoutOverflowWarning: 2147483648 does not fit into a 32-bit signed integer.
Timeout duration was set to 1.
```

Against a real ThruBox-Server (`f026f22`, default config) on localhost, `poll(..., { intervalMs: 30 days })` reported `{"RelayRateLimitError":465}` to `onError` within 1 second, and the server logged 465 `rate limit exceeded` lines.

## Environment Details

- `@aossie-org/thrubox-client` 1.1.0 (`main` at `9afc707`)
- Node.js v24.22.1 on Linux
- Browser timer behaviour checked in headless Google Chrome 155
- ThruBox-Server `f026f22` built with go1.24.6, default config

## Impact

Low - Minor inconvenience

It only happens with timeout or interval values above about 24.8 days. When it does happen, every request fails, or `poll()` floods the relay and runs into the rate limit.

## Proposed fix

- Add `const MAX_TIMER_MS = 2 ** 31 - 1;` in `src/client.ts`.
- Extend the existing checks so that `timeout > MAX_TIMER_MS` (constructor) and `intervalMs > MAX_TIMER_MS` (`poll()`) throw, with a message giving the limit (for example "Timeout must be at most 2147483647 ms").
- Add boundary tests in `tests/client.test.ts` (`2 ** 31 - 1` is accepted, `2 ** 31` throws) and give the maximum in the README options tables.
- Optional hardening: cap the retry backoff delay (`Math.min(2 ** attempt * 1000, <cap>)`). Today it would only overflow from the 23rd retry, after about 48 days of earlier backoff sleeps, so this is not needed for the bug above.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
