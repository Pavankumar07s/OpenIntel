# [BUG]: `RelayNetworkError` discards the underlying fetch error, so Node network failures only say "fetch failed"

### Bug Description

When `fetch` rejects, `RelayClient.request()` keeps only the message of the original error and throws it away (`src/client.ts:330-331`):

```ts
} else if (error instanceof Error) {
  lastError = new RelayNetworkError(error.message);
}
```

In Node 18+ (undici), every connection-level failure rejects with `TypeError: fetch failed`. The actual reason (`ECONNREFUSED`, `ENOTFOUND`, `ECONNRESET`, TLS certificate errors, ...) is only available on `error.cause`. Because `RelayNetworkError` does not keep the original error, callers see the same `"fetch failed"` message for "relay is down", "hostname typo" and "bad certificate". They have no way to tell these apart or log the real reason. The original stack is lost as well.

The timeout branch has the same problem: it replaces the `AbortError` with a new `"Request timed out"` error and does not link the two.

### Steps to Reproduce

1. With Node 18+ and nothing listening on port 31150, run:

```js
import { RelayClient } from '@aossie-org/thrubox-client';

for (const base of ['http://127.0.0.1:31150', 'http://relay.invalid']) {
  try {
    await new RelayClient(base, { retries: 0 }).health();
  } catch (e) {
    console.log(base, '->', e.name, JSON.stringify(e.message), 'cause:', e.cause);
  }
  try {
    await fetch(base + '/health');
  } catch (e) {
    console.log('  plain fetch ->', e.message, '| cause.code:', e.cause?.code);
  }
}
```

2. Compare what the SDK error contains with what plain `fetch` reports.

### Logs and Screenshots

```
http://127.0.0.1:31150 -> RelayNetworkError "fetch failed" cause: undefined
  plain fetch -> fetch failed | cause.code: ECONNREFUSED
http://relay.invalid -> RelayNetworkError "fetch failed" cause: undefined
  plain fetch -> fetch failed | cause.code: ENOTFOUND
```

Expected: the `RelayNetworkError` keeps the original error as `cause`, so `err.cause.cause.code` is `ECONNREFUSED` or `ENOTFOUND`.
Actual: `err.cause` is `undefined`, and both failures look identical.

### Environment Details

- `@aossie-org/thrubox-client` 1.1.0 (`main` at `9afc707`)
- Node.js v24.22.1, Linux. The same applies to any Node 18+ release, because they all use undici's `fetch failed` + `cause` shape.
- Browsers are not affected in practice: their `TypeError: Failed to fetch` carries no extra detail.

### Impact

Low - Minor inconvenience

### Proposed fix

- Let `RelayNetworkError` accept and keep the original error. For example, add an optional `options?: { cause?: unknown }` parameter and set `this.cause` in the constructor, with a `public readonly cause?: unknown` declaration. This compiles under the current `lib: ["ES2020", "DOM"]`. Passing `{ cause }` to `super()` would instead need `lib` raised to `ES2022`.
- Pass `{ cause: error }` in both the network branch and the timeout branch of `request()`.
- Small hardening in the same lines: detect timeouts with `error?.name === 'AbortError'` instead of `error instanceof DOMException`. The current check only works where `DOMException` is a global. That holds for Node 18+ and modern browsers. Elsewhere, evaluating it throws a raw `ReferenceError` out of `request()`.
- Add a test that a rejected `fetch` produces a `RelayNetworkError` whose `cause` is the original error.

This touches the same `catch` block as #80 / PR #81, so the two changes should be coordinated.

I prototyped this change locally. `tsc --noEmit` passes, all 30 existing tests pass, and the repro above then shows `cause` set for refused, DNS and timeout failures.
