# [FEATURE]: Machine-readable error responses (stable error codes) so clients don't have to match English text

### Feature and its Use Cases

**What happens today.** Every error is written with `http.Error`. The body is `text/plain; charset=utf-8`: an English sentence with a trailing newline. A client can only branch on the status code, and several different failures share one status:

| Status | Bodies the relay can send |
|---|---|
| 400 | `field 'to' is required`, `field 'from' is required`, `field 'payload' is required`, `invalid JSON body`, `failed to read request body`, `after must be a non-negative integer`, `limit must be an integer from 1 to 1000`, `address is required` |
| 401 | `unauthorized: invalid or missing API key` |
| 403 | `origin not allowed by CORS policy` (preflight) |
| 404 | `404 page not found` (Go mux default: unknown route or wrong base URL) |
| 405 | `Method Not Allowed` (Go mux default) |
| 413 | `payload too large` |
| 429 | `rate limit exceeded` |
| 500 | `internal server error` |

**Why this is a problem.**

- The official client copies the body verbatim into `RelayError.message` (`ThruBox-Client/src/client.ts:307-308`), so apps see `"field 'to' is required\n"`, newline included.
- To tell "missing field" from "malformed JSON" from "bad `limit`", an app or the SDK has to match on the English text. Any change to the wording silently breaks that.
- The SDK cannot map validation errors to a typed error class, or say which field was wrong.
- New error kinds, such as a future per-mailbox quota, cannot be added in a way clients can detect.

**Proposal**

1. Add a small helper, for example `writeError(w, status, code, message)`. It writes:
   ```json
   {"error":{"code":"missing_field","message":"field 'to' is required","field":"to"}}
   ```
   with `Content-Type: application/json`. Use it in the handlers and in the middleware (API key, rate limiter, CORS preflight rejection). No new dependency is needed.
2. Define a short list of stable codes and document it in the README API section: `missing_field`, `invalid_json`, `body_read_failed`, `invalid_after`, `invalid_limit`, `payload_too_large`, `unauthorized`, `rate_limited`, `origin_not_allowed`, `not_found`, `method_not_allowed`, `internal`. After that, changes to the list go in the CHANGELOG.
3. **Keep backward compatibility.** Clients released today, including SDK 1.1.0, read the error body as text and put it into the error message, so switching every response to JSON would turn their messages into raw JSON. Return the JSON body only when the request's `Accept` header includes `application/json`, and keep the plain-text body otherwise. `Accept` is a CORS-safelisted header, so the SDK can start sending it without causing extra preflights.
4. Give unknown routes and wrong methods the same shape when JSON is requested. One way is to register a catch-all `/` handler. Otherwise the mux's default `404 page not found` / `405` stay plain text.
5. Follow-up in ThruBox-Client:
   - send `Accept: application/json`
   - parse the error body when the response `Content-Type` is JSON
   - expose `RelayError.code` (and `field`)
   - fall back to the trimmed text for older servers

### Additional Context

Current behaviour on `main` (`f026f22`):

```
$ curl -si -X POST :3000/api/messages -H 'Content-Type: application/json' -d '{"to":"","from":"x","payload":"y"}'
HTTP/1.1 400 Bad Request
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff

field 'to' is required

$ curl -si :3000/api/nope
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8

404 page not found
```

The same errors seen through `@aossie-org/thrubox-client` 1.1.0:

```
send({ to: '', ... })        -> RelayError  status 400  message "field 'to' is required\n"
receive('x', { limit: 0 })   -> RelayError  status 400  message "limit must be an integer from 1 to 1000\n"
```

This changes the API contract across both repos, so it needs a maintainer decision on the shape and the code names before anyone writes code. Trimming the trailing newline in the client can be done now, independently of this proposal.
