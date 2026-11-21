# [BUG]: `send()` fails with a misleading `405 Method Not Allowed` when the relay URL answers with a 301/302 redirect

### Bug Description

`request()` calls `fetch` without a `redirect` option (`src/client.ts:259-263`), so redirects are followed automatically. Under the Fetch spec, a `301`/`302` answer to a `POST` (and a `303` to anything) is re-sent as a `GET` with no body. ThruBox Server only registers `POST /api/messages` for that path (`cmd/relay/main.go:92-93` in ThruBox-Server), so the re-sent `GET` gets `405 Method Not Allowed` with `Allow: POST`. The 4xx branch (`src/client.ts:306-308`) turns that into `RelayError(405, "Method Not Allowed\n")`.

`health()` and `receive()` are `GET`s, so they keep working through the same redirect. The relay looks healthy, and only `send()` fails, with an error that gives no hint that a redirect happened or what URL to use instead.

A common way to hit this is to use `http://relay.example.com` as `baseUrl` for a relay deployed the way the ThruBox Server README describes. Running `certbot --nginx` there adds an HTTP-to-HTTPS `return 301`. Other proxies that send `301`/`302` for HTTP-to-HTTPS, or for a moved host, behave the same way. Redirects that keep the method (`307`/`308`, such as Caddy's default HTTPS redirect) are not affected: `send()` works through them today.

This applies to Node.js (and other server runtimes using standard `fetch`). In browsers, a cross-origin `send()` sends a CORS preflight first, because of `Content-Type: application/json`. Per the Fetch spec a redirected preflight fails, so there the call fails with a generic network error instead (not tested here).

### Steps to Reproduce

1. Start ThruBox Server on `127.0.0.1:3000` (`RELAY_SERVER_PORT=3000 RELAY_SERVER_HOST=127.0.0.1 ./relay`).
2. Start a redirector that stands in for an HTTP-to-HTTPS redirect (`redirector.mjs`):
   ```js
   import http from 'node:http';
   http.createServer((req, res) => {
     res.writeHead(301, { Location: `http://127.0.0.1:3000${req.url}` });
     res.end();
   }).listen(8080, '127.0.0.1');
   ```
3. Run this against `main` (`npm run build` first):
   ```js
   import { RelayClient } from './dist/index.js';
   const c = new RelayClient('http://127.0.0.1:8080', { retries: 0 });
   const run = async (l, f) => { try { console.log(l, 'OK', JSON.stringify(await f())); } catch (e) { console.log(l, 'FAIL', e.name, e.statusCode, JSON.stringify(e.message)); } };
   await run('health ', () => c.health());
   await run('receive', () => c.receive('0xabc'));
   await run('send   ', () => c.send({ to: '0xabc', from: 'me', payload: 'aGk=' }));
   ```
4. Change the redirector status to `308` and run again: `send()` succeeds.

**Expected:** `send()` either works or fails with an error that says the relay redirected the request and names the target URL, so the user can fix `baseUrl`.

**Actual:** `health()` and `receive()` succeed, `send()` throws `RelayError 405 "Method Not Allowed\n"`, and nothing is stored.

### Logs and Screenshots

```
health  OK {"status":"ok","timestamp":"2026-11-19T19:20:28Z"}
receive OK []
send    FAIL RelayError 405 "Method Not Allowed\n"
```

The relay log has no `message stored` line for the `301` run. With `308`, the same script prints `send OK` and the relay logs `message stored`.

A raw `fetch` of the same `POST` shows what happened. The redirect is visible on the response but is never reported by the SDK:

```
405 redirected= true url= http://127.0.0.1:3000/api/messages allow= POST
```

### Environment Details

- OS: Linux (kernel 6.17)
- Node.js 24.22.1
- ThruBox-Client `main` @ `9afc707`
- ThruBox-Server `main` @ `f026f22` (built with Go 1.24.6)

### Impact

Low - Minor inconvenience

### Proposed fix

- **Minimal, non-breaking:** in `request()`, when a response is not 2xx and `response.redirected` is `true`, throw a non-retryable `RelayError` whose message includes `response.url`. For example: `Request was redirected to <url> and failed with 405; set baseUrl to the final URL`. Add a unit test that uses a mocked response with `redirected: true`.
- Keep following redirects as today (`307`/`308` setups work and should keep working).
- Add a short README note to use the relay's final `https://` URL as `baseUrl`.
