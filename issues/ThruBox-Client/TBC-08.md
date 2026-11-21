# [BUG]: Empty, "." or ".." address fails with a misleading `RelayNotFoundError("404 page not found\n")`; server error messages keep a trailing newline

## Bug Description

`receive()` and `poll()` do not check the `address` argument before building the URL (`src/client.ts:108-117`).

- **Empty address.** `receive('')` requests `/api/messages/`, which matches no server route. The server's own `address is required` check never runs. The caller gets `RelayNotFoundError` with the message `"404 page not found\n"`.
- **`"."` or `".."`.** `encodeURIComponent` leaves dots unescaped, so URL normalisation turns the path into `/api/messages/` or `/api/` before the request is sent. The caller gets the same `404 page not found`.
- **Polling.** `poll('', cb, { onError })` reports that 404 on every cycle. The message points to a missing resource, not to the empty address the caller passed in.
- **Unreadable mailbox.** The server accepts `send({ to: '.' })` (201). `receive('.')` can never read that mailbox back.

Separately, the 404 and other 4xx branches (`src/client.ts:280-281`, `:307-308`) copy the response body into `RelayError.message` unchanged. The server writes those bodies with Go's `http.Error`, which adds `\n`, so every 4xx message ends in a newline. Examples: `"field 'to' is required\n"`, `"limit must be an integer from 1 to 1000\n"`.

Other invalid inputs (`limit: 0`, `after: -1`, an empty `to`/`from`/`payload` in `send()`) already work: the server rejects them with a clear `400` that the SDK surfaces as `RelayError` and does not retry. Apart from the trailing newline, they need no client change.

## Steps to Reproduce

1. Start a ThruBox-Server (`main`), e.g. `RELAY_SERVER_PORT=3000 ./relay`.
2. Run this with the built SDK:

```js
import { RelayClient } from '@aossie-org/thrubox-client';
const c = new RelayClient('http://127.0.0.1:3000', { retries: 0 });
const show = (p) => p.then(
  (r) => console.log('OK', r),
  (e) => console.log(e.name, e.statusCode, JSON.stringify(e.message)),
);
await show(c.receive(''));
await show(c.receive('.'));
await show(c.receive('..'));
await show(c.send({ to: '.', from: 'b', payload: 'x' }));      // stored (201)
await show(c.receive('.'));                                     // still 404
await show(c.receive('0xA', { limit: 0 }));                     // trailing \n
```

## Logs and Screenshots

```
RelayNotFoundError 404 "404 page not found\n"
RelayNotFoundError 404 "404 page not found\n"
RelayNotFoundError 404 "404 page not found\n"
OK { id: 'fbfa13ac…', seq: 1, to: '.', from: 'b', payload: 'x', … }
RelayNotFoundError 404 "404 page not found\n"
RelayError 400 "limit must be an integer from 1 to 1000\n"
```

`poll('', () => {}, { intervalMs: 500, onError })` called `onError` 5 times in about 2.3 s, each time with `RelayNotFoundError: "404 page not found\n"`.

Raw server responses:

```
$ curl -si http://127.0.0.1:3000/api/messages/
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8
...
404 page not found
```

**Expected:** an empty, `"."` or `".."` address is rejected before any request is sent, with a message that names the problem (for example `address is required`). Error messages taken from the server contain no trailing whitespace.

**Actual:** a request is sent, a misleading `RelayNotFoundError("404 page not found\n")` comes back, and every server-supplied error message ends in `\n`.

## Environment Details

- `@aossie-org/thrubox-client` 1.1.0 (`main` at `9afc707`), built with `tsup`
- ThruBox-Server `main` at `f026f22`, built with Go 1.24.6
- Node.js v24.22.1, Linux

## Impact

Low - Minor inconvenience

## Proposed fix

In `src/client.ts`:

- In `receive()`, reject an empty, `"."` or `".."` address before calling `fetch`. `poll()` gets this through `receive()`, or it can throw synchronously, as it already does for an invalid `after`. Follow the existing argument-validation style; whether to use a new error class is the maintainers' choice.
- Apply `.trim()` to the response text in the 404 and 4xx branches before building the error.
- Add Vitest cases: an empty address and `"."` reject without calling `fetch`, and a `400` body ending in `\n` gives a message without it.

Out of scope for this fix:

- **Server side.** The server should stop accepting `"."`/`".."` as `to` on `POST`, so such mailboxes cannot be created.
- **`limit` cap.** Do not hard-code the reference server's `limit` cap of 1000 in the client. `ReceiveOptions.limit` documents it as server-specific, and the server already returns a clear `400`.
