# [BUG]: Any web page can store messages through a visitor's browser: `POST /api/messages` accepts `text/plain`, so no CORS preflight is sent

## Bug Description

`HandleCreate` (`internal/handler/message.go:33-52`) reads the body and runs `json.Unmarshal` on it. It never looks at `Content-Type`. Browsers treat a `POST` whose `Content-Type` is `text/plain`, `application/x-www-form-urlencoded` or `multipart/form-data` as a CORS "simple request" and send it with **no preflight**. So any web page can make its visitors' browsers store messages in a relay that has no `api_key`. The CORS setting makes no difference:

- **Default (`allowed_origins: []`).** The README's CORS section says *"By default the relay serves no CORS headers, so a browser calling it from another origin is blocked at the preflight."* That holds for `application/json` requests. A `text/plain` POST never gets a preflight, so it reaches the handler and is stored.
- **With an allowlist.** `internal/middleware/cors.go:80-83` sends non-preflight requests from unlisted origins on to the handler. Only the response is hidden from the page. The write still happens.
- **With `api_key` set**, the write is blocked. A browser cannot add `X-API-Key` without a preflight, and the relay answers that preflight with `403`.

The page cannot read the response, but it can still write. For example, it can flood a mailbox with forged messages from many visitors' IPs, where each IP gets its own rate-limit bucket. It can also post to a relay on loopback or a private network when the operator's own browser visits the page. Recent Chromium versions ask the user before a public site may reach a local address, so that second case depends on the browser. The relay is open to non-browser clients by design, so the impact is low. The problem is the gap between what operators expect and what happens: the docs and the allowlist suggest that other origins cannot use the relay from a browser, but that is only true for reads and for JSON requests.

## Steps to Reproduce

1. Start the relay with an allowlist and no API key:
   ```bash
   RELAY_SERVER_PORT=3000 RELAY_SECURITY_ALLOWED_ORIGINS=https://app.example.com go run ./cmd/relay
   ```
2. Send a preflight from an unlisted origin. It is rejected, as expected:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' -X OPTIONS http://localhost:3000/api/messages \
     -H 'Origin: https://evil.example' -H 'Access-Control-Request-Method: POST' \
     -H 'Access-Control-Request-Headers: content-type'
   # 403
   ```
3. Send the same write as a simple request (what a browser sends for `fetch(url, {method: 'POST', mode: 'no-cors', headers: {'Content-Type': 'text/plain'}, body})`):
   ```bash
   curl -s -i -X POST http://localhost:3000/api/messages \
     -H 'Origin: https://evil.example' -H 'Content-Type: text/plain' \
     -d '{"to":"0xvictim","from":"0xforged","payload":"x"}'
   # HTTP/1.1 201 Created   (Vary: Origin, no Access-Control-Allow-Origin)
   ```
4. `curl http://localhost:3000/api/messages/0xvictim` shows that the message was stored.

The relay with no CORS configuration gives the same result: the preflight gets `405` and the `text/plain` POST gets `201`. Leaving out the `Content-Type` header (so curl sends `application/x-www-form-urlencoded`) also gives `201`.

**In a real browser:** a page served from a different origin (`http://127.0.0.1:<port>`) ran the `no-cors` `text/plain` fetch above against relays on `http://localhost`. The message was stored on the allowlist relay (origin not listed) and on the default relay. Nothing was stored on a relay with `api_key` set. The same body sent with `Content-Type: application/json` failed with `TypeError: Failed to fetch` (blocked at the preflight) and was not stored.

## Logs and Screenshots

Output of the headless Chrome page (the opaque results are expected for `no-cors`):
```
browser-textplain-allowlist-relay: resolved type=opaque status=0   -> stored
browser-textplain-default-relay:   resolved type=opaque status=0   -> stored
browser-textplain-apikey-relay:    resolved type=opaque status=0   -> not stored
browser-json-allowlist-relay:      rejected TypeError: Failed to fetch -> not stored
browser-json-default-relay:        rejected TypeError: Failed to fetch -> not stored
```

## Environment Details

- ThruBox-Server `main` @ `f026f22` (VERSION 1.0.0), built with Go 1.24.6
- Linux x86_64
- Google Chrome 155 (headless) for the browser check, curl for the rest

## Impact

Low - Minor inconvenience

## Proposed fix

- In `HandleCreate`, before reading the body, require a JSON media type and return `415 Unsupported Media Type` otherwise:
  ```go
  if mt, _, err := mime.ParseMediaType(r.Header.Get("Content-Type")); err != nil || mt != "application/json" {
      http.Error(w, "Content-Type must be application/json", http.StatusUnsupportedMediaType)
      return
  }
  ```
  `application/json` is not a CORS-safelisted type, so every cross-origin browser write then needs a preflight, which the existing CORS policy already handles. This uses only the standard library (`mime`).
- Tests: set `Content-Type: application/json` in the `post()` helper in `internal/handler/message_test.go`, which every existing POST test uses. Add cases showing that `text/plain`, form-encoded and missing content types get `415` and store nothing.
- Docs: change the README's CORS paragraph so it no longer says that all cross-origin browser calls are stopped at the preflight. Add a CHANGELOG note that POSTs without `Content-Type: application/json` (for example a bare `curl -d`) now get `415`. The official TypeScript client (`client.ts:242-244`) and the README `curl` example already send the header.

I tried this change locally. `go test ./...` passes after the one-line helper update. The same browser page's `text/plain` POST is rejected and nothing is stored. A JSON POST from an allowlisted origin still gets `201`. The client SDK's `send()` still works.
