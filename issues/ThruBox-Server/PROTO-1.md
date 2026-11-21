# [BUG]: CORS preflight rejects `X-API-Key` on a relay without an API key, so browser clients that pass `apiKey` fail every request

### Bug Description

The CORS middleware lists `X-API-Key` in `Access-Control-Allow-Headers` only when `security.api_key` is set (`internal/middleware/cors.go:43-46`, the `requireAPIKey` argument).

The official client (`@aossie-org/thrubox-client`) adds `X-API-Key` to every request whenever `apiKey` is passed (`src/client.ts:246-248`). That includes `GET /health` and `GET /api/messages/{address}`. `X-API-Key` is not a CORS-safelisted header, so the browser sends a preflight before each of those requests. On a relay with no API key, the preflight response leaves the header out of the allowed list, and the browser blocks the real request.

So when an origin is allowlisted, a browser app that passes `apiKey` to a relay with no key gets `RelayNetworkError("Failed to fetch")` from `health()`, `receive()` and `send()`. The same code works from Node, where CORS does not apply, and it also works against a relay that does have a key. The relay would accept these requests anyway: with no key configured, `APIKeyAuth` passes everything through and ignores the header. Only the preflight response blocks them.

Ways this happens in practice:

- An operator removes or temporarily clears `security.api_key`, for example during key rotation or when opening a relay to the public. Every deployed web client stops working, while server-side clients carry on.
- One frontend build talks to a staging relay with no key and a production relay with one.
- A developer runs the relay locally with no key but leaves the key in the app's `.env`.

Chrome blames CORS rather than the key, and the SDK only reports `Failed to fetch`. That makes the cause hard to find.

### Steps to Reproduce

1. Start a relay with CORS enabled and no API key:
   ```bash
   RELAY_SECURITY_ALLOWED_ORIGINS='*' go run ./cmd/relay
   ```
2. Send the preflight that a browser sends before a request carrying the key:
   ```bash
   curl -i -X OPTIONS http://127.0.0.1:3000/api/messages \
     -H 'Origin: http://localhost:5173' \
     -H 'Access-Control-Request-Method: POST' \
     -H 'Access-Control-Request-Headers: content-type,x-api-key'
   ```
3. Or run this in a page served from another origin:
   ```js
   import { RelayClient } from '@aossie-org/thrubox-client';
   const relay = new RelayClient('http://127.0.0.1:3000', { apiKey: 'any-key', retries: 0 });
   await relay.health(); // RelayNetworkError: Failed to fetch
   ```

**Expected:** the preflight allows `X-API-Key`, so the real request goes through, and the relay ignores the header because no key is configured. With a key configured, nothing changes: a missing or wrong key still gets `401` from `APIKeyAuth`.

**Actual:** the preflight returns `Access-Control-Allow-Headers: Content-Type`, and the browser blocks the request before it is sent.

### Logs and Screenshots

Preflight, relay without a key compared with a relay with `RELAY_SECURITY_API_KEY=s3cret`:

```
--- relay without api_key
HTTP/1.1 204 No Content
Access-Control-Allow-Headers: Content-Type
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Origin: http://localhost:39505
--- relay with api_key
HTTP/1.1 204 No Content
Access-Control-Allow-Headers: Content-Type, X-API-Key
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Origin: http://localhost:39505
```

A non-browser request to the relay without a key, sending `X-API-Key: whatever`, gets `200`, because the header is ignored.

Headless Chrome, page origin `http://localhost:39505`, relays on `127.0.0.1`, client 1.1.0:

| Relay | Client options | `health()` | `receive()` | `send()` |
|---|---|---|---|---|
| no key | none | ok | ok | ok |
| no key | `apiKey: 'any-key'` | `RelayNetworkError: Failed to fetch` | `Failed to fetch` | `Failed to fetch` |
| key `s3cret` | `apiKey: 's3cret'` | ok | ok | ok |
| no key, with the fix below | `apiKey: 'any-key'` | ok | ok | ok |

Chrome console:

```
Access to fetch at 'http://127.0.0.1:39500/health' from origin 'http://localhost:39505' has been blocked by CORS policy: Request header field x-api-key is not allowed by Access-Control-Allow-Headers in preflight response.
```

### Environment Details

- ThruBox-Server `main` at `f026f22`, built with Go 1.24.6 on Linux, built-in defaults plus `RELAY_*` env overrides
- `@aossie-org/thrubox-client` 1.1.0 (`main` at `9afc707`), built with `npm run build`
- Google Chrome 155 (headless), Node v24.22.1

### Proposed fix

Always allow the header in `CORS()`:

```go
allowHeaders := "Content-Type, X-API-Key"
```

Allowing a request header grants no access on its own. Authentication still happens in `APIKeyAuth`, and a relay without a key ignores the header. The `requireAPIKey` parameter can then be removed, along with the argument passed to it in `cmd/relay/main.go:108`.

- Update `TestCORS_AllowHeadersTracksAPIKey` to expect `X-API-Key` in both cases. I tried the change locally, and that is the only test that fails. Every other middleware test still passes.
- Update the README CORS section, which currently says "plus `X-API-Key` when `security.api_key` is set".

Background: #32 and PR #36 made the header conditional on purpose. Neither looked at the case of a client configured with a key talking to a relay without one.

### Impact

Medium - Feature works but has issues
