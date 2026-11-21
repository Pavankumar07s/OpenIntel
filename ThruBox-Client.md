# ThruBox-Client audit: contribution opportunities

- **Repo:** AOSSIE-Org/ThruBox-Client, `main` at `9afc707` (2026-11-07), package `@aossie-org/thrubox-client@1.1.0`
- **Server used for e2e:** AOSSIE-Org/ThruBox-Server `main` at `f026f22`, built with go1.24.6
- **Audit date:** 2026-11-19. Node v24.22.1, npm 11.19.1, headless Google Chrome via puppeteer-core for the browser checks
- **Method:** I read every file in `src/` and `tests/` and the server's handler, middleware, store and config code. I ran every npm script plus publint, arethetypeswrong (attw) and knip. I then drove four live server instances with the built `dist/` bundle from Node and from headless Chrome. Every bug below marked VERIFIED has a failing test or script, and its output is quoted.
- **Read-only on GitHub:** nothing was filed, commented, forked or pushed. Both working trees are clean (`git status` is empty), and every server process I started has been stopped.

---

## 1. Summary

| Check | Result |
|---|---|
| `npm ci` | OK. Installs 11 dev-only vulnerabilities (see audit) |
| `npm run build` (tsup) | OK. ESM 10.09 KB, CJS 11.38 KB, `index.d.ts` and `index.d.cts` 7.95 KB each. Minified ESM is 3.9 KB, 1.7 KB gzip (the README says "~2.5 KB minified") |
| `npm test` (vitest 5.0.5) | **30/30 pass**, 1 file |
| `npm run lint` | **FAIL**, 1 error: `src/client.ts:60:7 preserve-caught-error` (rule in the ESLint 10 recommended set) |
| `npm run format:check` | **FAIL**, 4 files: `src/client.ts`, `src/errors.ts`, `src/index.ts`, `tests/client.test.ts`. The code uses double quotes; `.prettierrc` says `singleQuote: true` |
| `npx tsc --noEmit` | Pass, but it checks `src/` only (`tsconfig.json:29`). `tests/client.test.ts` gives **13 type errors** under the project's own strict flags (`noUncheckedIndexedAccess`) |
| `npm audit` | 11 dev-only: 1 high (brace-expansion), 9 moderate (minimatch chain via eslint and typescript-eslint), 1 low (esbuild). `--omit=dev`: **0** (no runtime dependencies) |
| Coverage (`vitest --coverage`, v8) | Statements 91.91%, **branches 83.78%**, functions 94.11%, lines 93.7%. The gaps are all in `client.ts` (see TBC-14) |
| publint | 1 warning (`types` resolves as ESM for `require`), 1 suggestion (`sideEffects`) |
| attw `--pack` | **node16 (from CJS): "Masquerading as ESM"**. Bundler, node10 and node16-ESM are fine |
| knip | Flags only `dangerfile.js`, a false positive because Danger CI uses it |
| CI | **No workflow runs build, test, lint or typecheck on pull requests.** Tests run only inside `version-release.yml`, after the tag and GitHub Release are created |
| E2E, Node vs real server (`e2e.mjs`) | 39 checks: 27 behave as expected, **12 deviations**, which became findings TBC-01, 02, 04, 06, 07 plus protocol mismatches P-1 to P-3 |
| E2E, misbehaving mock server (`mock-e2e.mjs`) | Confirms #80 (non-JSON 200 retried 3x, 3.3 s) and #82 (POST never retried). Timeouts work correctly before headers and mid-body |
| E2E, headless Chrome (`browser-e2e.mjs`, `browser-backlog.mjs`) | `health`, `send` and `receive` work cross-origin with CORS `*`. Found P-4 (`Retry-After` hidden from JS), P-5 (`X-API-Key` blocked on open relays), CORS rejections retried (TBC-06), and the poll backlog livelock (TBC-09) |
| Failing-test repro suite (`repro/repro.test.ts`) | **11/11 tests fail on main**, as intended (one per bug) |

Environment note: the root disk filled up briefly during the run, and one scratch `npm install` failed with ENOSPC. I re-ran it with the npm cache on tmpfs and it succeeded. No finding comes from that failure.

---

## 2. Findings table (ranked by value)

| ID | Title | Type | Sev | Status | Effort | GitHub |
|---|---|---|---|---|---|---|
| TBC-01 | `poll()` stop function does not cancel the in-flight request and its retries | bug | medium | VERIFIED | S | NEW |
| TBC-02 | `timeout`/`intervalMs` above 2^31-1 ms overflow to 1 ms: `poll()` floods the server, requests abort instantly | bug | medium | VERIFIED | S | NEW |
| TBC-03 | `poll()` loop runs as an un-awaited promise: a throw inside `reportError` crashes Node with an unhandled rejection | bug | medium | VERIFIED | S | NEW |
| TBC-04 | `poll()` ignores `RelayRateLimitError.retryAfter` and never backs off on repeated errors | bug | medium | VERIFIED | S–M | NEW (related to #75) |
| TBC-05 | CommonJS TypeScript consumers get ESM type declarations (attw "Masquerading as ESM", TS1479) | bug (packaging) | medium | VERIFIED | S | NEW |
| TBC-06 | Network errors drop their cause (`"fetch failed"` only), break without global `DOMException`, and deterministic failures are retried | bug | medium | VERIFIED | S–M | NEW (same code region as #80/#81) |
| TBC-07 | Constructor accepts base URLs with query, hash or credentials that produce broken requests | bug | low | VERIFIED | S | NEW |
| TBC-08 | No input validation in `receive()`/`send()`/`poll()`; raw server text with trailing `\n` leaks into errors; validation throws a generic `Error` | bug | low | VERIFIED | S | NEW |
| TBC-09 | Add page size (`limit`) and backlog draining to `poll()`: large backlogs never get delivered on slow links | feature | medium | VERIFIED (failure mode) | M | NEW |
| TBC-10 | Support `AbortSignal` on `send`/`receive`/`health` | feature | medium | n/a (feature) | M | NEW |
| TBC-11 | Add `fetch` and `headers` options to `ClientOptions` | feature | low | n/a (feature) | S | NEW |
| TBC-12 | `npm run lint` and `npm run format:check` fail on main | tooling | low | VERIFIED | S | NEW (mentioned in PR #81/#83 text only) |
| TBC-13 | Add a pull-request CI workflow (build, test, lint, format, typecheck, attw) and run tests before tagging releases | CI | medium | VERIFIED | S–M | NEW |
| TBC-14 | Cover untested branches (real timeout, 204, Retry-After date, POST no-retry, poll error paths) and enforce coverage thresholds | tests | low | VERIFIED | S–M | NEW |
| TBC-15 | Add contract/integration tests against a real ThruBox-Server in CI | tests | medium | n/a | M | NEW |
| TBC-16 | Docs: browser/CORS usage, security caveats (`from` is unauthenticated, shared API key), full error list, retry timing | docs | low–medium | VERIFIED (doc gaps) | S–M | NEW |
| TBC-17 | Cleanup bundle: naming, mojibake, stale checklist, `sideEffects`, dev-dependency audit, action pinning | chore | low | VERIFIED | S | NEW |

Protocol mismatches P-1 to P-6 (section 4) are mostly server-side or docs items, written up for the server team or as client-doc follow-ups.

---

## 3. Findings

### TBC-01: `poll()` stop function does not cancel the in-flight request and its retries

- **Type:** bug · **Severity:** medium · **Status:** VERIFIED · **Effort:** S · **GitHub:** NEW
- **Evidence:**
  - `src/client.ts:225-228`: `stop()` only sets `stopped = true` and clears the *next* scheduled timer.
  - `src/client.ts:189-192`: the in-flight `this.receive()` keeps running through `request()`'s whole retry loop.
  - `src/client.ts:337-342`: the backoff sleep is a bare `setTimeout` promise, and nothing can cancel it.
  - The result is that up to `retries` more HTTP requests go out after `stop()`, spread over about 7 s or more. In Node, that also keeps the process alive after the app has "stopped" polling. The same happens in React effects and test teardown, where `stop()` is expected to end all network activity.
- **Repro, real server** (`e2e.mjs` step 18, `retries: 2` against a dead port): `{"requestsAtStop":1,"requestsAfter4500ms":3}`.
- **Repro, failing vitest** (fails on main: `expected 4 to be 1`):
  ```ts
  it('makes no requests after stop()', async () => {
    vi.useFakeTimers();
    mockFetch.mockRejectedValue(new TypeError('fetch failed'));
    const c = new RelayClient('http://relay.test', { retries: 3 });
    const stop = c.poll('0x1', vi.fn(), { intervalMs: 60_000, onError: vi.fn() });
    await vi.advanceTimersByTimeAsync(10);   // first attempt failed, client is sleeping in backoff
    stop();
    const atStop = mockFetch.mock.calls.length;
    await vi.advanceTimersByTimeAsync(20_000);
    expect(mockFetch.mock.calls.length).toBe(atStop);
  });
  ```
- **Proposed fix:**
  - Give each `poll()` an `AbortController`, and have `stop()` call `controller.abort()`.
  - Thread an internal `signal` through `receive()` into `request()`.
  - In `request()`, combine the caller's signal with the per-attempt timeout controller. Use a manual listener, because `AbortSignal.any` is not available on Node 18.
  - Make the backoff sleep abortable, so it rejects or returns at once when the signal aborts.
  - Do not report errors caused by the abort.
  - This also lays the groundwork for TBC-10.

### TBC-02: `timeout`/`intervalMs` above 2^31-1 ms overflow to 1 ms

- **Type:** bug · **Severity:** medium · **Status:** VERIFIED · **Effort:** S · **GitHub:** NEW
- **Evidence:**
  - `src/client.ts:67-69` and `:158-160` only check `> 0 && Number.isFinite`.
  - `setTimeout` in Node and in browsers clamps delays above 2,147,483,647 ms (about 24.8 days) to 1 ms. Node prints `TimeoutOverflowWarning: ... Timeout duration was set to 1`.
  - `intervalMs: 30 days`: `setTimeout(doPoll, intervalMs)` (`:217`) re-polls every ~1 ms, a self-inflicted flood that immediately trips the server's rate limiter.
  - `timeout: 30 days`: every request aborts after ~1 ms with `RelayNetworkError("Request timed out")` (`:256`).
  - `retries >= 23`: the backoff `Math.pow(2, attempt) * 1000` (`:339`) passes 2^31 and collapses to 1 ms, so retries spin. There is also no cap on backoff delay.
- **Repro, real server** (`e2e.mjs` steps 14-15):
  - `poll(intervalMs: 30 days) makes only 1 request in 500ms -> {"requestsIn500ms":139}`
  - `timeout: 30 days -> {"type":"RelayNetworkError","msg":"Request timed out","ms":5}`
- **Repro, failing vitest** (main: `expected 126 to be 1`, and the constructor does not throw):
  ```ts
  it('poll(intervalMs: 30 days) must not hammer the server', async () => {
    mockFetch.mockImplementation(() => Promise.resolve(json([])));
    const c = new RelayClient('http://relay.test', { retries: 0 });
    const stop = c.poll('0x1', vi.fn(), { intervalMs: 30 * 24 * 3600 * 1000 });
    await new Promise((r) => setTimeout(r, 200));
    stop();
    expect(mockFetch.mock.calls.length).toBe(1);
  });
  it('timeout: 30 days must be rejected up front', () => {
    expect(() => new RelayClient('http://relay.test', { timeout: 30 * 24 * 3600 * 1000 })).toThrow();
  });
  ```
- **Proposed fix:**
  - Add `const MAX_TIMER_MS = 2 ** 31 - 1`.
  - Reject `timeout` and `intervalMs` above it with a typed validation error (see TBC-08), or clamp them.
  - Cap the backoff, for example `Math.min(2 ** attempt * 1000, 30_000)`.
  - Add tests at the boundary.

### TBC-03: `poll()` loop is an un-awaited promise, so any throw escaping `reportError` crashes the process

- **Type:** bug · **Severity:** medium · **Status:** VERIFIED · **Effort:** S · **GitHub:** NEW
- **Evidence:**
  - `src/client.ts:222` runs `doPoll();`, and `:217` runs `setTimeout(doPoll, intervalMs)`. Neither attaches a `.catch`.
  - Inside `reportError` (`:169-182`), two calls sit outside the guarded `onError` call:
    - `new Error(String(err))` (`:171`) throws for values that cannot become a string, such as `Object.create(null)` or objects with a throwing `toString`.
    - `console.error(...)` (`:180`) throws in the common test setup that fails tests on `console.error`.
  - Either throw rejects `doPoll`'s promise, and nothing catches it. On Node 15+ an unhandled rejection kills the process. Polling also stops silently.
- **Repro** (`poll-crash.mjs`, real server):
  ```js
  relay.poll('0xNobody', () => { throw Object.create(null); }, { intervalMs: 200, onError: (e) => console.log(e.message) });
  ```
  Output:
  ```
  TypeError: Cannot convert object to primitive value
      at String (<anonymous>)
      at reportError (.../dist/index.js:158:60)
      at doPoll (.../dist/index.js:191:11)
  process exit code 1
  ```
  `onError` is never called.
- **Failing vitest:** the TBC-03 block in `repro/repro.test.ts`. On main, `onError` is never called and the loop dies after the first poll.
- **Proposed fix:**
  - Wrap the loop: `const run = () => { doPoll().catch(reportErrorSafely) }`, and use `run` for both the first call and the `setTimeout`.
  - Make error normalisation total, for example `try { String(err) } catch { 'Unknown error' }`, and wrap the `console.error` fallback in try/catch.
  - Add tests for both triggers.

### TBC-04: `poll()` ignores `RelayRateLimitError.retryAfter` and never backs off on repeated errors

- **Type:** bug · **Severity:** medium · **Status:** VERIFIED · **Effort:** S–M · **GitHub:** NEW. Issue #75 (negative `retryAfter`) touches the same value but is a different bug.
- **Evidence:**
  - `src/client.ts:193-195` and `:216-218`: after any error, including `RelayRateLimitError`, the next poll runs after the fixed `intervalMs`.
  - The SDK parses `retryAfter` (`:288-302`) but never uses it.
  - Persistent failures such as 401 (bad key), 404 (wrong base URL) or a network outage are retried forever at full rate, logging every cycle.
  - On the reference server the limit is per-IP and shared across all clients behind one NAT or proxy. A polling client that keeps hitting a limited relay wastes its own budget and everyone else's.
  - Related, minor: `request()` also ignores `Retry-After` on 503 (mock: 3 attempts in 3.8 s, though the server asked for 1 s spacing).
- **Repro, real server C with `rate_limit=2`** (`e2e.mjs` step 19): `poll backs off after RelayRateLimitError(retryAfter=60) -> {"requestsIn1s":10,"errors":["RelayRateLimitError:60"]}`.
- **Failing vitest** (main: `expected 11 to be 1`):
  ```ts
  mockFetch.mockResolvedValueOnce(text('rate limit exceeded\n', 429, { 'Retry-After': '60' }))
           .mockImplementation(() => Promise.resolve(json([])));
  const stop = c.poll('0x1', vi.fn(), { intervalMs: 1000, onError: vi.fn() });
  await vi.advanceTimersByTimeAsync(10_000); stop();
  expect(mockFetch.mock.calls.length).toBe(1);
  ```
- **Proposed fix:**
  - In `doPoll`, compute the next delay as `max(intervalMs, retryAfter * 1000)` after a `RelayRateLimitError`.
  - Add capped exponential backoff on consecutive failures, reset after a success.
  - Optionally add `PollOptions.stopOn?: (err) => boolean`, or stop automatically on `RelayUnauthorizedError`.
  - Document the behaviour.
  - Coordinate with whoever fixes #75: clamp first, then honour the value.

### TBC-05: CommonJS TypeScript consumers resolve ESM type declarations ("Masquerading as ESM")

- **Type:** bug (packaging) · **Severity:** medium · **Status:** VERIFIED · **Effort:** S · **GitHub:** NEW
- **Evidence:**
  - `package.json:11` has one `"types": "./dist/index.d.ts"` condition for both `import` and `require`. Under `"type": "module"`, `.d.ts` means ESM types.
  - tsup already emits `dist/index.d.cts`, but nothing references it.
  - The AGENTS.md hard constraint says the package must keep shipping both ESM and CJS with declarations.
- **Repro:**
  - `npx @arethetypeswrong/cli --pack .` reports `node16 (from CJS) 👺 Masquerading as ESM`.
  - publint: `pkg.exports["."].types types is interpreted as ESM when resolving with the "require" condition`.
  - A real CJS consumer (`cjs-consumer/`: `"type": "commonjs"`, `module`/`moduleResolution: node16`, installing the `npm pack` tarball) fails to type-check:
    ```
    index.ts(1,50): error TS1479: The current file is a CommonJS module whose imports will produce 'require'
    calls; however, the referenced file is an ECMAScript module and cannot be imported with 'require'.
    ```
    At runtime, `require('@aossie-org/thrubox-client')` works (`CJS runtime OK ok`). Only the types are wrong.
- **Proposed fix:**
  ```json
  "exports": {
    ".": {
      "import": { "types": "./dist/index.d.ts", "default": "./dist/index.js" },
      "require": { "types": "./dist/index.d.cts", "default": "./dist/index.cjs" }
    },
    "./package.json": "./package.json"
  }
  ```
  Then add `attw --pack` and `publint` to CI (TBC-13) so this does not regress.

### TBC-06: Network errors drop their cause, break without a global `DOMException`, and deterministic failures are retried

- **Type:** bug · **Severity:** medium · **Status:** VERIFIED · **Effort:** S–M · **GitHub:** NEW. It edits the same catch block as #80 and open PR #81 (non-JSON 2xx), so coordinate to avoid conflicts.
- **Evidence** (`src/client.ts:319-335`):
  1. **Cause dropped.** `:330-331` does `new RelayNetworkError(error.message)`. In Node, undici puts the real reason (ECONNREFUSED, ENOTFOUND, certificate errors) on `error.cause` and the message is just `"fetch failed"`. Callers cannot tell "server down" from "DNS typo" from "bad TLS cert".
  2. **`DOMException` assumed global.** `:326` runs `error instanceof DOMException`. In a fetch-capable runtime without a global `DOMException` (reported for some React Native/Hermes setups), every network failure escapes as a raw `ReferenceError` instead of a `RelayNetworkError`.
  3. **Deterministic `TypeError`s retried with backoff as if transient.** Examples: a URL with credentials (see TBC-07), and a browser CORS rejection. Both always fail, yet each costs the full retry budget (4.2–4.5 s) before surfacing.
- **Repro:**
  - `e2e.mjs` step 13 against a closed port and a `.invalid` host:
    ```
    network error preserves underlying cause (ECONNREFUSED) -> {"type":"RelayNetworkError","msg":"fetch failed","status":0}
    DNS error preserves underlying cause (ENOTFOUND)       -> {"type":"RelayNetworkError","msg":"fetch failed","status":0}
    ```
  - `no-domexception.mjs` (deletes `globalThis.DOMException`, then calls a dead port): `error surfaced to caller: ReferenceError - DOMException is not defined`.
    - The code path is VERIFIED. Which real runtimes lack `DOMException` is SUSPECTED; it was not tested on a device.
  - Headless Chrome, origin not on server B's allowlist, `retries: 2`: `{"type":"RelayNetworkError","msg":"Failed to fetch","ms":4521}` after 3 attempts.
  - Credentials in the URL (`e2e.mjs` step 17): `attempts: 3, ms: 4190, msg: "Request cannot be constructed from a URL that includes credentials..."`.
  - Failing vitest: the TBC-06 block. `cause` is `undefined`.
- **Proposed fix:**
  - Pass the original error as `cause`: `new RelayNetworkError(msg, { cause: error })`. This needs `lib: ["ES2022", "DOM"]` in `tsconfig.json:5`, or setting `this.cause` manually.
  - Detect timeouts with `(error as {name?: string})?.name === 'AbortError'` instead of `instanceof DOMException`.
  - Optionally add `code` (`'ECONNREFUSED'` etc.) from `cause.code` to the message.
  - Optionally add a `RelayTimeoutError` subclass.
  - Retrying CORS failures cannot be avoided in general, because browsers make them indistinguishable from offline errors. Document it, and validate the URL up front (TBC-07).

### TBC-07: Constructor accepts base URLs with a query, hash or credentials that produce broken requests

- **Type:** bug · **Severity:** low · **Status:** VERIFIED · **Effort:** S · **GitHub:** NEW
- **Evidence:**
  - `src/client.ts:58`: `parsedUrl.toString().replace(/\/+$/, "")`, after which paths are appended by string concatenation (`:239`).
  - `http://relay/?tenant=a` produces `http://relay/?tenant=a/health`, and `http://relay/#x` produces `http://relay/#x/health`.
  - `http://user:pass@relay` passes validation, then `fetch` throws on every call and the call is retried (TBC-06).
  - A sub-path base (`https://host/relay/`) works and should keep working.
- **Repro** (`e2e.mjs` step 16, real server):
  ```
  baseUrl http://127.0.0.1:3901/?tenant=x accepted by constructor but requests break -> RelayNotFoundError "404 page not found\n"
  baseUrl http://127.0.0.1:3901/#frag     accepted by constructor but requests break -> RelayNotFoundError "404 page not found\n"
  ```
  The failing vitest gets `'http://relay.test/?tenant=a/health'`, and the constructor does not throw for credentials.
- **Proposed fix:**
  - Throw a validation error when `parsedUrl.search`, `parsedUrl.hash`, `parsedUrl.username` or `parsedUrl.password` is set.
  - Alternatively, build request URLs with `new URL(path, base)` and keep the search params; rejecting is simpler.
  - Also fix the double-prefixed message `Invalid base URL: Invalid base URL protocol...` (`:56` and `:60`).

### TBC-08: No input validation in `receive()`/`send()`/`poll()`; raw server text with trailing newline in errors; generic `Error` for validation

- **Type:** bug · **Severity:** low · **Status:** VERIFIED · **Effort:** S · **GitHub:** NEW
- **Evidence:**
  - `receive()` (`src/client.ts:108-117`) passes any `address`, `after` or `limit` straight to the server. `poll()` validates `after` but not `address`. `send()` sends empty fields.
  - The server's `http.Error` bodies end in `\n`, and `:307-308` copies them verbatim into `RelayError.message`.
  - An empty address, `.` or `..` is normalised away by URL parsing and comes back as a misleading `RelayNotFoundError("404 page not found\n")`. So `poll('')` loops forever on "not found".
  - Validation errors (`:56`, `:68`, `:71`, `:159`, `:163`) are plain `Error`. That goes against AGENTS.md ("Errors are typed... rather than throwing generic `Error`"), and callers cannot `instanceof`-check them.
- **Repro** (`e2e.mjs` steps 5, 6, 8, real server):
  ```
  receive({"limit":0})  -> RelayError 400 "limit must be an integer from 1 to 1000\n"
  receive({"after":1.5})-> RelayError 400 "after must be a non-negative integer\n"
  receive("") / (".") / ("..") -> RelayNotFoundError 404 "404 page not found\n"
  send({"to":"",...})   -> RelayError 400 "field 'to' is required\n"
  ```
  The failing vitest block TBC-08 has three tests: an empty address and `limit: 0` should reject without calling `fetch`, and the message should be trimmed.
- **Proposed fix:**
  - Add `RelayValidationError extends RelayError` (statusCode 0, or a separate `code`).
  - Validate a non-empty `address` that is not `.` or `..`, an integer `after >= 0`, an integer `limit` from 1 to 1000, and non-empty `to`/`from`/`payload`.
  - Use the new error for the constructor and `poll` checks.
  - `.trim()` server text.
  - Keep the 1000 cap in sync with the server's `MaxPageSize` (`internal/handler/message.go:130`) or make it configurable.

### TBC-09: Add page size (`limit`) and backlog draining to `poll()`

- **Type:** feature (fixes a real failure mode) · **Severity:** medium · **Status:** VERIFIED failure mode · **Effort:** M · **GitHub:** NEW
- **Evidence:**
  - `poll()` calls `receive(address, { after: cursor })` with no `limit` (`src/client.ts:189-192`), so the first poll, or any poll after an outage, downloads the whole backlog in one response.
  - The server allows 500 KB per message (`internal/config/config.go:73`), and `limit` (1–1000) exists for exactly this case. The client has a 10 s timeout and the server a 15 s `WriteTimeout`, so a large backlog on a slow link can never be fetched.
  - The cursor never moves, so the client re-downloads from `after=0` forever: a livelock.
- **Repro** (`browser-backlog.mjs`):
  - Setup: real server D, 20 messages × 400 KB (8,000,080 bytes), headless Chrome throttled to 1.6 Mbit/s (a "Fast 3G"-class link), `poll({ after: 0, intervalMs: 500 })`, client defaults otherwise.
  - Output after 32 s:
    ```
    { "pollDeliveries": 0, "pollErrors": ["Request timed out","Request timed out","Request timed out"],
      "urls": ["?after=0","?after=0","?after=0","?after=0","?after=0&limit=1"],
      "pagedReceive": { "got": 1, "seq": 1, "ms": 3982 } }
    ```
  - The same mailbox fetched with `receive(..., { after: 0, limit: 1 })` succeeds in 4 s.
- **Proposed design:**
  - Add `PollOptions.limit` (default, for example, 100 when `after` is set).
  - When a page comes back full, deliver it, advance the cursor, and immediately fetch the next page without waiting `intervalMs`.
  - Also export a `receiveAll(address, { after, pageSize })` async iterator helper.
  - Keep today's behaviour when `after` is not set. Tests use fake timers.
  - Optionally let the caller raise `timeout` per call.

### TBC-10: Support `AbortSignal` on `send`/`receive`/`health`

- **Type:** feature · **Severity:** medium · **Status:** n/a · **Effort:** M · **GitHub:** NEW
- **Evidence:**
  - None of the public methods take options for cancellation (`src/client.ts:87`, `:108`, `:124`). `request()`'s `init` is private.
  - Worst-case latency of one `receive()` with defaults is 4 × 10 s timeouts plus 1+2+4 s backoff plus up to 3 s jitter, about 50 s, and the caller cannot cancel it. That matters for UI navigation, React effect cleanup and server request handlers with deadlines.
- **Proposed design:**
  - Add an optional `{ signal?: AbortSignal }` (and optionally `timeout?` and `retries?` per call) to each method: `send(params, opts?)`, `receive(address, { after, limit, signal })`, `health(opts?)`.
  - Merge the signal with the per-attempt timeout. Make the backoff sleep abortable.
  - On user abort, throw a distinct error (re-throw the `AbortError`, or add `RelayAbortError`) and never retry.
  - Implement after TBC-01, or together with it, since they share the plumbing.

### TBC-11: Add `fetch` and `headers` options to `ClientOptions`

- **Type:** feature · **Severity:** low · **Status:** n/a · **Effort:** S · **GitHub:** NEW
- **Evidence:**
  - `src/client.ts:259` calls the global `fetch`.
  - `:241-248` builds headers internally, and only `Content-Type` and `X-API-Key` can ever be sent.
  - The server README recommends deploying behind Nginx or Cloudflare Tunnel. Cloudflare Access service tokens need `CF-Access-Client-Id`/`CF-Access-Client-Secret` headers, which the SDK cannot send.
  - Proxies (undici `ProxyAgent` via a custom fetch), instrumentation and tests (no `vi.stubGlobal`) also need an injectable `fetch`.
  - Runtimes without a global `fetch` cannot use the SDK at all.
- **Proposed design:**
  - `ClientOptions.fetch?: typeof fetch`, defaulting to `globalThis.fetch`, bound so that `this` does not matter.
  - `ClientOptions.headers?: Record<string, string>`, merged under the SDK's own headers.
  - Keep zero dependencies. Document both, and add tests.

### TBC-12: `npm run lint` and `npm run format:check` fail on main

- **Type:** tooling · **Severity:** low · **Status:** VERIFIED · **Effort:** S · **GitHub:** NEW. PR #81 and #83 descriptions mention "pre-existing failures", but no issue exists.
- **Evidence:**
  - `npm run lint` exits 1: `src/client.ts:60:7 error There is no 'cause' attached to the symptom error being thrown preserve-caught-error`.
  - `npm run format:check` exits 1 on 4 files. All source uses double quotes, but `.prettierrc` sets `singleQuote: true`.
  - CONTRIBUTING.md (section "3. Test Your Changes") tells contributors to run both.
- **Fix:**
  - Run `npm run format` once, in a single formatting-only commit.
  - Fix `:60` by passing `{ cause: err }`. The project's `lib: ["ES2020","DOM"]` (`tsconfig.json:5`) does not type `Error` options: I checked that `new Error("x", { cause })` gives `TS2554: Expected 0-1 arguments` under ES2020 lib. So bump `lib` to ES2022 (which still runs on Node 18 and modern browsers), or assign `cause` manually.
  - Add `tests/` to a `tsconfig.test.json` type-check and fix the 13 `possibly undefined` errors.
  - Note: open PR #83 adds `{ cause: err }` at `:60` without changing `tsconfig`. Based on the check above, it probably fails `tsc`/the DTS build: SUSPECTED, because I did not execute the PR's code.

### TBC-13: Add a pull-request CI workflow and run tests before tagging releases

- **Type:** CI · **Severity:** medium · **Status:** VERIFIED · **Effort:** S–M · **GitHub:** NEW
- **Evidence:**
  - `.github/workflows/` has 14 workflows: danger, labels, scorecard, codeql, stale, release and others. None runs `npm ci && build && test && lint` on `pull_request`.
  - Because of that, the failing lint and format checks (TBC-12) and the packaging bug (TBC-05) went unnoticed.
  - `version-release.yml` creates and pushes the tag (step at line 91) and publishes the GitHub Release (step at line 103) *before* the `build` job runs `npm test` (step at line 230). A failing test leaves a published tag and release with no npm package.
  - The same file uses non-SHA-pinned `actions/checkout@v4` (lines 33 and 205) and `actions/github-script@v7` (39 and 105). Other steps in that file are SHA-pinned, and OpenSSF Scorecard (which the README badges) flags unpinned actions.
  - BestPracticesChecklist marks `warnings_fixed` and `tests_are_added` as unmet for lack of CI.
- **Proposed fix:**
  - Add `.github/workflows/ci.yml` on `pull_request` and `push: main` with a Node matrix of 18, 20, 22 and 24 (Node 18 is the declared `engines` floor).
  - Steps: `npm ci`, `npm run build`, `npx tsc --noEmit` (including tests), `npm test -- --coverage`, `npm run lint`, `npm run format:check`, `npx publint`, `npx @arethetypeswrong/cli --pack .`.
  - Pin actions by SHA, set `permissions: contents: read`, and use harden-runner like `scorecard.yml`.
  - In `version-release.yml`, move build and test ahead of tag creation, or gate the release job on the CI job.

### TBC-14: Cover untested branches and enforce coverage thresholds

- **Type:** tests · **Severity:** low · **Status:** VERIFIED (coverage run) · **Effort:** S–M · **GitHub:** NEW
- **Evidence** (v8 coverage on main): branch coverage is 83.78%. Uncovered code in `src/client.ts`:
  - `:256`: the real timeout path is never exercised. The existing test (`tests/client.test.ts:198-203`) makes `fetch` reject with an `AbortError` and never lets the timer fire.
  - `:269`: 204 handling.
  - `:291`, `:296-298`: `Retry-After` missing, or given as an HTTP date.
  - `:308`: 4xx with an empty body.
  - `:329`, `:333`: re-thrown `RelayError` and non-`Error` rejections.
  - `:159`: invalid `intervalMs`.
  - `:170`, `:185`: errors after stop.
  - `:180`: the default `console.error` path.
  - Nothing asserts that `send()` is *not* retried on 5xx, which is the documented safety property.
  - `vitest.config.ts` sets no `thresholds`.
- **Proposed fix:**
  - Add tests for each branch. Use fake timers with a never-resolving `fetch` for the real-timeout case, and `Retry-After: <HTTP-date>` with a fixed `Date.now`.
  - Add `coverage.thresholds` (for example lines 95, branches 90) and `include: ['src/**']`.
  - Publish coverage in CI (TBC-13).
  - Also fix the `test_most` item in BestPracticesChecklist.

### TBC-15: Add contract/integration tests against a real ThruBox-Server in CI

- **Type:** tests · **Severity:** medium · **Status:** n/a · **Effort:** M · **GitHub:** NEW
- **Rationale:**
  - All 30 unit tests mock `fetch` with hand-written responses. Several real behaviours were only visible against the real server:
    - Error bodies end with `\n` (TBC-08).
    - `max_payload_size` counts the whole JSON body (P-2).
    - Timestamp precision differs between POST and GET (P-1).
    - CORS hides `Retry-After` (P-4).
    - `X-API-Key` is blocked on open relays (P-5).
  - The server publishes a Docker image (its `docker-publish.yml`) and is easy to `go run`.
- **Proposed design:**
  - Add `tests/integration/*.test.ts`, gated on `THRUBOX_URL`.
  - Add a CI job that starts `ghcr.io/aossie-org/thrubox-server` as a service container (or checks out the server and runs it with `go run ./cmd/relay`), with env overrides for a small `RELAY_SECURITY_RATE_LIMIT`, a `RELAY_SECURITY_API_KEY` instance and a small `RELAY_MESSAGES_MAX_PAYLOAD_SIZE`.
  - Port the scenarios from `e2e.mjs`: send, duplicate send, cursor and limit, 400/401/404/413/429 mapping, `poll` exactly-once delivery, `stop()` semantics.
  - This also guards future server releases against breaking the SDK.

### TBC-16: Docs: browser/CORS usage, security caveats, full error list, retry timing

- **Type:** docs · **Severity:** low–medium · **Status:** VERIFIED (gaps checked against code and server behaviour) · **Effort:** S–M · **GitHub:** NEW
- **Gaps:**
  1. **Browser usage is undocumented.** The server README says the "ThruBox client docs assume" a same-origin reverse proxy, but the client README never mentions CORS, `allowed_origins`, or that every `send()` triggers a preflight.
  2. **Security caveats.** The client's own security review (`audit/sec_review_2026-10-29...md`, "Review-specific limitations") recommends these notes, and they are not done yet:
     - `Message.from` (`src/types.ts:21`) is whatever the sender claimed. The relay does not authenticate it, so apps must verify a signature inside the encrypted payload.
     - The SDK does **no** encryption, yet the tagline says "Send, receive, and manage encrypted messages", and "manage" is stale since `delete()` was removed.
     - The `X-API-Key` is one shared secret. In a browser bundle it is public, and it goes over plain `http:` if configured that way.
     - Wallet addresses are exact-match and case-sensitive (see P-6). Normalise them before `send`/`receive`.
  3. **Errors.** The README shows only `RelayRateLimitError` and `RelayPayloadTooLargeError`. `RelayNetworkError`, `RelayUnauthorizedError`, `RelayNotFoundError`, base `RelayError.statusCode`, and which errors are retried are undocumented.
  4. **Retry timing.** GET retries also fire on timeouts. Worst case is about 50 s per call with defaults. POST is never retried (#82 covers the `retries` row only).
  5. **Size claim.** "~2.5 KB minified" is actually 3.9 KB minified, 1.7 KB gzip.
- **Fix:**
  - Add README sections "Using from a browser" and "Security notes".
  - Complete the error table.
  - Update the JSDoc on `Message.from` and `ClientOptions.apiKey`.

### TBC-17: Cleanup bundle (one PR)

- **Type:** chore · **Severity:** low · **Status:** VERIFIED · **Effort:** S · **GitHub:** NEW
- Items:
  - **Naming.** "AOSSIE Relay Server/SDK" and the non-existent `https://relay.aossie.org` in JSDoc (`src/client.ts:23,27,49`, `src/index.ts:1-2`) should say ThruBox and use `relay.example.com` like the README.
  - **CONTRIBUTING.md:**
    - Broken emoji bytes (U+FFFD) in the headings at lines 7 and 18.
    - Template leftover "### Python / PEP 8" at lines 248-249.
    - No explicit "new functionality must include tests" policy. AGENTS.md has one, and CONTRIBUTING should mirror it to tick `test_policy` in the checklist.
  - **BestPracticesChecklist.md / checklist-status.json** are stale:
    - `vulnerability_report_process` is marked unmet, but `AOSSIE-Org/.github` provides an org-level `SECURITY.md` that GitHub applies to this repo.
    - `test_most` says coverage is "not measured", but it is 93.7% lines / 83.8% branches.
  - **`package.json`:** add `"sideEffects": false` (publint suggestion) and `"./package.json"` to `exports`.
  - **Dev-dependency audit:** `npm audit fix` clears all 11 dev-only advisories (brace-expansion high, minimatch moderate, esbuild low). Review and merge or close the stale Dependabot PRs #73, #76 and #78 in the same pass.
  - **`.pre-commit-config.yaml`:** `detect-secrets` runs without a `.secrets.baseline`, which is noted as a TODO in the file.
  - **5xx error bodies are discarded** (`src/client.ts:312-315`, `"Server error: 503"`). Include the trimmed body text, as the 4xx path does.

---

## 4. Client/Server protocol mismatches

All of these were observed against the real server built from ThruBox-Server `f026f22`. "Owner" says which repo should change.

| # | Mismatch | Evidence | Status | Owner / suggested action |
|---|---|---|---|---|
| P-1 | **Timestamp precision differs between POST and GET.** A fresh `send()` returns nanosecond RFC3339 (`createdAt: "2026-11-19T17:39:00.101067072Z"`), but `receive()`, and a duplicate `send()`, return the stored, second-truncated value (`"2026-11-19T17:39:00Z"`). The same holds for `expiresAt`. Code that compares or dedups by `createdAt` sees two values for one message | Server `internal/handler/message.go:59` (`time.Now()`), `:92` encodes the in-memory value; `internal/store/sqlite.go:231` stores `Format(time.RFC3339)`. Observed in `e2e.mjs` step 3 | VERIFIED | **Server:** truncate `now` to seconds before responding (or store RFC3339Nano). **Client:** document that the fields are ISO-8601 strings of varying precision |
| P-2 | **`max_payload_size` limits the whole JSON request body, not `payload`.** With `max_payload_size=1024`, a 1000-byte payload is rejected with 413, because `to`, `from` and the JSON envelope count too. Client docs ("Payload exceeds server limit") and server docs ("500KB max payload") both say payload | Server `internal/handler/message.go:35` `http.MaxBytesReader(w, r.Body, MaxPayloadSize)`. `e2e.mjs` step 9: `RelayPayloadTooLargeError 413` | VERIFIED | **Server:** either check `len(req.Payload)` after decoding (with a separate small envelope allowance) or rename and document it as a max *body* size. **Client:** fix the `RelayPayloadTooLargeError` and README wording once decided |
| P-3 | **Error responses are `text/plain` with a trailing newline, and there is no machine-readable code.** The client copies them verbatim (`"field 'to' is required\n"`). Unknown routes return Go's `"404 page not found\n"` | Server uses `http.Error` throughout `message.go`/middleware; client `src/client.ts:307-308` | VERIFIED | **Client:** trim (TBC-08). **Server (optional):** a JSON error shape such as `{"error":"...","code":"..."}` would let the SDK map errors without string matching |
| P-4 | **`Retry-After` is invisible to browsers cross-origin.** The CORS middleware never sends `Access-Control-Expose-Headers: Retry-After`, so `response.headers.get('Retry-After')` is `null` in the browser. The SDK silently falls back to its hard-coded 60. That matches the reference server's fixed 60 s only by coincidence: the real remaining window is shorter, and any other value a proxy sends is lost | Server `internal/middleware/cors.go:89-94` (no Expose-Headers), `ratelimit.go:54`. Chrome: `"raw429": {"status":429,"retryAfterVisibleToJs":null}`. curl shows the 429 carries `Access-Control-Allow-Origin` but no Expose-Headers | VERIFIED | **Server:** add `Access-Control-Expose-Headers: Retry-After` on non-preflight CORS responses. **Client:** document the fallback |
| P-5 | **Client with `apiKey` against a relay without `api_key` fails in browsers.** The server lists `X-API-Key` in `Access-Control-Allow-Headers` only when a key is configured, so the preflight rejects it and the SDK reports an opaque `RelayNetworkError("Failed to fetch")`. The same client works fine from Node | Server `internal/middleware/cors.go:43-46`. Chrome console: `Request header field x-api-key is not allowed by Access-Control-Allow-Headers in preflight response`. Result `{"type":"RelayNetworkError","msg":"Failed to fetch"}` | VERIFIED | **Server:** always allow `X-API-Key` (harmless when unused). **Client:** document it (TBC-16) |
| P-6 | **Addresses are exact-match, case-sensitive strings.** `0xabcdef…` and its checksum-cased form are different mailboxes. EIP-55 checksummed and lowercased wallet strings from different libraries will silently split a user's mailbox | Server `sqlite.go` `WHERE to_addr = ?`. `e2e.mjs` step 7: `found: 0` | VERIFIED (by design) | **Docs (both):** state that addresses are opaque, exact-match keys. **Client (optional):** an `addressNormalizer` option, or a note to lowercase EVM addresses |
| P-7 | **Duplicate send returns `200`, a new send returns `201`, and the SDK hides which one happened.** Apps that want to know whether a retry was deduplicated cannot tell | Server `message.go` (status 200 vs 201); client `src/client.ts:266-271` returns only the body | VERIFIED | **Client (optional, low):** expose `created: boolean` via a `sendWithMeta()`, or document it. Relevant to #82 and PR #83 (opt-in send retry) |
| P-8 | **POST is never retried although the server is idempotent.** Already tracked | `src/client.ts:250`; mock: `send() 503 -> attempts 1` | VERIFIED | Covered by **#82 / PR #83** |

Also checked, with no mismatch found:
- The message JSON fields (`id`, `seq`, `to`, `from`, `payload`, `createdAt`, `expiresAt`) and `HealthResponse` match the server exactly.
- `after`/`limit` query names and semantics match. The cursor `poll` delivered seq `1,2,3,4` exactly once (`e2e.mjs` step 12).
- Reserved and Unicode characters in addresses (`a/b`, `x?y#z`, `üñî✓`, spaces, `%2F`) round-trip correctly through `encodeURIComponent` and Go's `PathValue`.
- `/health` is reachable without an API key, and 401/429 map to the right typed errors.

---

## 5. Pick-up-able existing issues and PR status

Open issues: 3. Open PRs: 6, of which 4 are Dependabot.

| Issue | Assignee | Linked PR | Verified? | Notes |
|---|---|---|---|---|
| **#75** Negative `retryAfter` when `Retry-After: -5` | none | **none** | Yes. Code at `src/client.ts:292-294` has no clamp, while the date path at `:298` does | **Free to pick up** (S). The reporter asked to work on it in a comment but is not assigned. Fix: `Math.max(0, parsedNum)` and also reject non-finite values. Add tests for `-5`, `1.5`, `""`, an HTTP date in the past and a missing header. Pair it with TBC-04 |
| #80 Non-JSON 2xx retried and misclassified as `RelayNetworkError` (#79 is its closed duplicate) | none | PR #81 (open, unreviewed, no tests) | Yes. The mock gave 3 attempts in 3.3 s and `RelayNetworkError` | Not free. **Review opportunity:** PR #81 changes only `src/client.ts` and adds no test. A reviewer could ask for a regression test and for the empty-body 200 case |
| #82 Opt-in retry for `send()` | none | PR #83 (open) | Yes (`send() 503 -> attempts 1`) | Not free. **Review opportunity:** PR #83 adds `{ cause: err }` at `src/client.ts:60` under `lib: ES2020`, which probably fails `tsc` (TS2554, see TBC-12). SUSPECTED, because the PR code was not executed |

Dependabot PRs #73 (typescript-eslint), #76 (vitest 5.1.0), #77 (harden-runner) and #78 (eslint 10.14.0) are open. None of them clears the `npm audit` advisories by itself (see TBC-17).

---

## 6. Suggested split for a 3-person team

- **Person A, poll loop correctness:** TBC-01 → TBC-03 → TBC-04 (plus #75) → TBC-09. These all touch `poll()`/`request()`, so one owner avoids merge conflicts.
- **Person B, transport and API surface:** TBC-06 (coordinate with PR #81) → TBC-07 → TBC-08 → TBC-02 → TBC-10 → TBC-11.
- **Person C, tooling, packaging, tests, docs:**
  - First: TBC-12, then TBC-13, then TBC-05. Land these first, because they make everyone else's PRs verifiable.
  - Then: TBC-14, TBC-15, TBC-16, TBC-17.
  - File the P-1, P-2, P-4 and P-5 server issues after checking the server audit for duplicates.

---

## Appendix: repro artifacts

The scripts are in `/tmp/claude-1000/-home-pavan-Desktop-gsoc/67cea752-264c-491c-8bf0-bb277aad86e9/scratchpad/thrubox-client/`. It is a session scratch directory and may be cleared, so the key code for each finding is quoted above.

- `start-servers.sh`: three relay instances, all built from ThruBox-Server `f026f22` and configured through env vars:
  - A: open, `max_payload_size=1024`, CORS `*`
  - B: `api_key=s3cret`, CORS allowlist `http://app.example`
  - C: `rate_limit=2`
  - A fourth instance D (default limits) was started separately for the backlog test.
- `e2e.mjs`: 39 Node checks of every public API against the live servers.
- `mock-e2e.mjs`: slow-header and slow-body timeouts, a non-JSON 200, a 503 on POST and GET.
- `poll-crash.mjs` (TBC-03), `no-domexception.mjs` (TBC-06), `seed-big.mjs` plus `browser-backlog.mjs` (TBC-09), `browser-e2e.mjs` (P-4, P-5, CORS retry).
- `cjs-consumer/`: a node16 CJS TypeScript project that installs the `npm pack` tarball (TBC-05).
- `repro/repro.test.ts`: 11 vitest cases that import `ThruBox-Client/src` and **all fail on main**. Run it with `vitest run --root repro`, using the repo's `node_modules`.
