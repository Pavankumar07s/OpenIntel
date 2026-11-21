# [FEATURE]: README: document browser/CORS setup, security notes, all error classes and retry timing

### Feature and its Use Cases

The README is missing information that integrators need, and one statement is out of date. Each point below was checked against `main` (9afc707) and ThruBox Server `main` (f026f22).

**1. Using the SDK from a browser.** The README says "Works in Node.js 18+ and modern browsers" but does not mention CORS. The server README describes a same-origin reverse proxy as "the setup the ThruBox client docs assume", but the client docs never say so. Things a browser user runs into:

- The relay must be same-origin (proxied), or list the app in `security.allowed_origins`.
- Every `send()` triggers a CORS preflight, because it sends `Content-Type: application/json`. With `apiKey` set, every request does, because of `X-API-Key`.
- If `apiKey` is set but the relay has no `api_key` configured, the browser blocks every request. The preflight only allows `Content-Type` (`Access-Control-Allow-Headers: Content-Type`), and the SDK reports a generic `RelayNetworkError` with the browser's message (Chrome: "Failed to fetch"). The same client works from Node, which does not apply CORS.
- `Retry-After` is not exposed to cross-origin JavaScript (the 429 response has no `Access-Control-Expose-Headers`), so in browsers `RelayRateLimitError.retryAfter` is always the fallback value 60.

The last two would best be fixed in the server. Until then, the README should mention them.

**2. Security notes.**

- `Message.from` is whatever the sender put in the request. The relay does not authenticate it, so apps must not treat it as proof of identity. They should verify a signature inside the encrypted payload. The project's own security review (`audit/sec_review_2026-10-29T06-03-39Z_fb9f3ca.md`, "Review-specific limitations") recommends saying this in the `from` doc comment (`src/types.ts:21-22`, currently "Sender wallet address") and in the README.
- The SDK does not encrypt or decrypt anything. `payload` must already be encrypted by the app. The tagline "Send, receive, and manage encrypted messages" can be read otherwise, and "manage" is out of date since `delete()` was removed (the README itself says "there is no way to delete them").
- `apiKey` is one secret shared by every client of a relay. In a browser bundle it is public, and over `http://` it is sent in clear text.
- Addresses are matched exactly, including case: `0xAbC…` and `0xabc…` are different mailboxes. Apps should normalise addresses (for example, lowercase EVM addresses) before calling `send()`/`receive()`.

**3. Errors.** The "Error Handling" section only shows `RelayRateLimitError` and `RelayPayloadTooLargeError`. The package also exports `RelayError` (base class, with `statusCode`), `RelayNetworkError` (`statusCode` 0, used for network failures **and** timeouts), `RelayUnauthorizedError` (401) and `RelayNotFoundError` (404). A table should list each class, its status code, and whether the SDK retries it:

- GET requests are retried on network errors, timeouts and 5xx.
- `send()` is never retried.
- 4xx responses are never retried.
- `retryAfter` is 60 when the header is missing or cannot be read.

**4. Retry timing.** With the defaults (`timeout: 10000`, `retries: 3`), a `receive()` or `health()` against an unresponsive relay can take 4 x 10 s plus 1 + 2 + 4 s of backoff plus up to 3 s of jitter, about 50 s. I measured 48.9 s against a server that accepts connections and never answers. The `retries` row ("Retry attempts on network errors / 5xx") should say that timeouts count, and that `send()` is excluded.

**5. Size claim.** "Tiny: ~2.5 KB minified" does not match the 1.1.0 build. `dist/index.js` minified with esbuild is 3,943 bytes, and 1,677 bytes gzipped. Suggest "~4 KB minified, under 2 KB gzipped", or drop the number.

**6. Polling budget.** ThruBox Server's default `rate_limit` is 30 requests per minute per IP. One `poll()` at the default `intervalMs` (10 s) uses 6 requests a minute, more when GET retries kick in on 5xx or network errors. Five pollers behind one IP (several tabs or addresses, or several users behind one NAT or proxy) already use the whole budget. With more than that, some polls in every window fail with `RelayRateLimitError` and go to `onError`. The README should state this budget and suggest a longer `intervalMs`, or one poller per address.

**7. "Payload" size wording.** The README's error example says "Payload exceeds server limit", and the `send()` JSDoc says `RelayPayloadTooLargeError` is thrown "If the payload exceeds the server's max size" (`src/client.ts:83`). ThruBox Server applies `max_payload_size` to the whole JSON request body, so `to`, `from` and the JSON envelope count too (with `max_payload_size=1024`, a 1000-byte payload gets a 413). The wording should match whatever the server side settles on for that limit.

**Proposed change** (docs and JSDoc only, no behaviour change)

- New README sections "Using from a browser" (point 1, linking to the server's CORS docs) and "Security notes" (point 2).
- Complete the error table (point 3) and the retry wording (point 4). Fix the size line and the tagline (points 2 and 5).
- Add the polling-budget note (point 6), and align the payload-limit wording in the README and the `send()` JSDoc with the server (point 7).
- Update the JSDoc on `Message.from` and `ClientOptions.apiKey` in `src/types.ts`.

### Additional Context

- The `send()` retry wording overlaps with #82 / PR #83, which updates the `retries` row and the `send()` section for the opt-in `retrySend`. Whichever lands second should adjust to the other.
- The browser behaviour in point 1 was checked with `curl` against the relay running with `RELAY_SECURITY_ALLOWED_ORIGINS='*'`: the preflight answered `Access-Control-Allow-Headers: Content-Type`, and the 429 response carried `Access-Control-Allow-Origin` and `Retry-After: 60` but no `Access-Control-Expose-Headers`.

### Code of Conduct

- [ ] I have joined the Discord server and will post updates there
- [ ] I have searched existing issues to avoid duplicates
