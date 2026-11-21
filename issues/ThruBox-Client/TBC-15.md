# [FEATURE]: Add integration tests that run the SDK against a real ThruBox Server

### Feature and its Use Cases

All 30 tests in `tests/client.test.ts` replace the global `fetch` with a mock (`vi.stubGlobal("fetch", mockFetch)`) and use hand-written `Response` objects. Nothing automated checks that the SDK and the server agree on the wire format. Maintainers currently check this by hand: #63, for example, lists a manual run of the built SDK against a server build.

Running the built SDK against ThruBox Server `main` (f026f22) shows behaviour the mocks do not model:

- Error bodies come from Go's `http.Error` and end with a newline, so SDK error messages do too. For example `POST /api/messages` without `to` returns `field 'to' is required\n`, and the mocks use `"bad request"`.
- `max_payload_size` limits the whole JSON request body, not just `payload`. With `RELAY_MESSAGES_MAX_PAYLOAD_SIZE=1024`, `send()` with a 1000-byte payload throws `RelayPayloadTooLargeError` (413).
- A fresh `send()` returns `createdAt`/`expiresAt` with nanosecond precision. `receive()`, or a duplicate `send()`, returns the stored second-precision value for the same message.
- A duplicate `send()` gets `200` with the existing message, and a new one gets `201`.

A real-server test suite would have caught these. It would also catch future server changes that break the SDK, such as a default page size, a new error format or new status codes.

**Proposed change**

- Add `tests/integration/*.test.ts`. They only run when `THRUBOX_URL` is set and are skipped otherwise, so `npm test` keeps working offline. Add an npm script such as `test:integration`.
- Scenarios:
  - `health()`
  - `send()`, and a duplicate `send()` returning the same `id`/`seq`
  - `receive()` with `after` and `limit`
  - the typed error for 400, 401 (relay with an API key), 404, 413 (small `max_payload_size`) and 429 (small `rate_limit`), with the server's real messages
  - `poll({ after })` delivering each message exactly once, and `stop()`
- Coverage gate: `vitest.config.ts` has no `coverage.include` and no `coverage.thresholds`, so coverage can drop without anyone noticing. Keep the integration files out of the unit coverage run, set `coverage.include: ['src/**']` and thresholds just below the numbers measured after the pending unit tests for the uncovered `src/client.ts` branches land (today: 93.7 % lines, 83.78 % branches), and run `npm run coverage` in CI.
- CI job, once a basic CI workflow exists: start `ghcr.io/aossie-org/thrubox-server` (published by the server's `docker-publish.yml`, tags such as `1`, `1.0`, `sha-<commit>`) as a service container. Configure it through env vars (`RELAY_SECURITY_RATE_LIMIT`, `RELAY_SECURITY_API_KEY`, `RELAY_MESSAGES_MAX_PAYLOAD_SIZE`), using one container per configuration. Pin the image tag to the server version the SDK targets.

### Additional Context

- These tests run in Node, where `fetch` does not apply CORS. Browser-only problems therefore stay out of scope, such as `Retry-After` not being readable cross-origin, or `X-API-Key` being rejected in the preflight by a relay without a key. They need either a headless-browser job or direct assertions on the server's CORS response headers.
- Where the server's current behaviour is itself questionable (payload limit wording, timestamp precision), the test should assert today's behaviour and link the server issue, so the suite stays green and only flags changes.

### Code of Conduct

- [ ] I have joined the Discord server and will post updates there
- [ ] I have searched existing issues to avoid duplicates
