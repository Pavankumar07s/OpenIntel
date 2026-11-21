# [BUG]: `poll()` ignores `RelayRateLimitError.retryAfter` and keeps polling every `intervalMs` while rate limited

## Bug Description

When a poll gets `429 Too Many Requests`, the SDK parses `Retry-After` into `RelayRateLimitError.retryAfter` (`src/client.ts:288-302`), but `poll()` never reads it. `doPoll()` passes the error to `onError` and then always schedules the next poll after the fixed `intervalMs` (`src/client.ts:193-195`, `:216-218`). So while the relay is rate limiting, `poll()` keeps sending requests that are rejected with 429 until the window ends, and `onError` (or `console.error`) fires on every one.

Errors that `request()` does not retry (401, 404, 429) also repeat at the plain `intervalMs` forever, because `poll()` has no backoff of its own. Network errors and 5xx are less affected: `request()` already retries those with 1 s / 2 s / 4 s backoff inside each poll cycle. But that delay is the same on every cycle and never grows.

How much this costs depends on `intervalMs`. With the default 10 s, it is about 6 rejected requests (and 6 `onError` calls) per rate-limited minute instead of 1. With a 1 s interval it is about 60. On the reference ThruBox-Server, the extra requests do not push back the end of the window: the limiter uses a fixed per-IP window, and in my test requests worked again exactly 60 s after the window started, even though about 110 rejected requests were sent during it. The cost is wasted round trips, error-handler noise, and one `WARN rate limit exceeded` server log line per rejected request.

## Steps to Reproduce

Against a real relay (ThruBox-Server `main`) with a low limit:

```bash
RELAY_SERVER_PORT=31130 RELAY_SECURITY_RATE_LIMIT=3 ./relay
```

```js
import { RelayClient } from '@aossie-org/thrubox-client';

const c = new RelayClient('http://127.0.0.1:31130', { retries: 0 });
for (let i = 0; i < 4; i++) { try { await c.receive('0xA'); } catch {} } // use up the 3/min budget

let requests = 0;
const realFetch = globalThis.fetch;
globalThis.fetch = (...a) => { requests++; return realFetch(...a); };

const errors = [];
const stop = c.poll('0xA', () => {}, {
  intervalMs: 500,
  onError: (e) => errors.push(`${e.name}:${e.retryAfter}`),
});
await new Promise((r) => setTimeout(r, 5000));
stop();
console.log({ requests, onErrorCalls: errors.length, first: errors[0] });
```

Or with mocked `fetch` in Vitest (fake timers):

```ts
mockFetch
  .mockResolvedValueOnce(new Response('rate limit exceeded\n', { status: 429, headers: { 'Retry-After': '60' } }))
  .mockImplementation(() => Promise.resolve(new Response('[]', { status: 200 })));
const stop = client.poll('0x1', vi.fn(), { intervalMs: 1000, onError: vi.fn() });
await vi.advanceTimersByTimeAsync(59_000);
stop();
expect(mockFetch).toHaveBeenCalledTimes(1); // actual: 60
```

**Expected:** after a `RelayRateLimitError`, the next poll waits at least `retryAfter` seconds, so there is 1 request in the 5 s run above.

**Actual:** the next poll runs after `intervalMs`, whatever the `retryAfter` value.

## Logs and Screenshots

Real server, `rate_limit=3`, `intervalMs: 500`:

```
{"windowMs":5001,"requests":10,"onErrorCalls":10,"firstErr":"RelayRateLimitError:60"}
```

Mocked `fetch`, 60 s of fake time:

```
one 429 (Retry-After: 60), intervalMs 1000      -> 60 requests in 59 s
persistent 429, default intervalMs (10 s)       -> 7 requests / 7 onError calls in 60 s
persistent 401, intervalMs 1000                 -> 61 requests in 60 s
network outage, intervalMs 1000, retries 3      -> 26 requests in 6 poll cycles in 60 s (request() backoff)
```

The only workaround today is to call `stop()` inside `onError` and start a new `poll()` after `retryAfter` seconds.

## Environment Details

- `@aossie-org/thrubox-client` 1.1.0 (`main` at 9afc707)
- ThruBox-Server `main` at f026f22 (Go 1.24.6), `RELAY_SECURITY_RATE_LIMIT=3`
- Node.js v24.22.1, Linux

## Impact

Low - Minor inconvenience

## Proposed fix

- In `doPoll()`, when the error is a `RelayRateLimitError`, schedule the next poll after `max(intervalMs, retryAfter * 1000)`.
- Clamp the delay before using it. Negative values (#75) should fall back to `intervalMs`. Non-finite or very large values (for example `Retry-After: Infinity`) must be capped, because `setTimeout` treats anything above 2^31-1 ms as 1 ms.
- Optionally, add capped exponential backoff for consecutive failed cycles, and reset it after a successful poll.
- Add tests for the 429 delay and for the clamping, and add a sentence to the `poll()` section of the README.
- Keep the current "polling continues after errors" behaviour: no auto-stop, so existing apps are not affected.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
