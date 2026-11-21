# [FEATURE]: Add `headers` and `fetch` options to `ClientOptions`

### Feature and its Use Cases

`RelayClient` can only send two headers: `Content-Type` (for POST) and `X-API-Key` (when `apiKey` is set) (`src/client.ts:241-248`). It always calls the global `fetch` (`src/client.ts:259`). `ClientOptions` has only `apiKey`, `timeout` and `retries`, and anything else passed in is silently ignored:

```js
// run from the repo root after `npm run build`
import http from 'node:http';
import { RelayClient } from './dist/index.js';

const srv = http.createServer((req, res) => {
  console.log('custom headers received:', Object.keys(req.headers).filter((h) => h.startsWith('x-') || h.startsWith('cf-')));
  res.writeHead(200, { 'Content-Type': 'application/json' }).end('{"status":"ok","timestamp":"t"}');
}).listen(3003, async () => {
  const relay = new RelayClient('http://127.0.0.1:3003', {
    apiKey: 'k',
    headers: { 'CF-Access-Client-Id': 'id', 'X-Request-Id': '1' }, // ignored
    fetch: () => { throw new Error('custom fetch was used'); },     // ignored
  });
  await relay.health();
  srv.close();
});
```

Output: `custom headers received: [ 'x-api-key' ]`. The custom `fetch` is never called.

**Use cases**

- **Extra headers.** A relay deployed behind an authenticating proxy needs its own header. For example, Cloudflare Access service tokens use `CF-Access-Client-Id` / `CF-Access-Client-Secret`, and nginx basic auth uses `Authorization`. The server README recommends running the relay behind nginx or a Cloudflare Tunnel, and both can have such auth in front. Request-ID or tracing headers are another case.
- **Custom `fetch`.** Use an undici `ProxyAgent` or another dispatcher for this client only, add logging or metrics, or pass a mock in tests without `vi.stubGlobal`. Today the only workaround is to replace `globalThis.fetch`, which affects the whole application.

**Proposed API (additive, still zero dependencies)**

```ts
interface ClientOptions {
  // existing: apiKey, timeout, retries
  /** Extra headers sent with every request. The SDK's own Content-Type and X-API-Key take precedence. */
  headers?: Record<string, string>;
  /** fetch implementation to use. Default: globalThis.fetch */
  fetch?: typeof fetch;
}
```

- Default to `globalThis.fetch`, and do not call it as a method of the client. In browsers, `this.fetchImpl(url)` with the native `fetch` throws "Illegal invocation". Store a bound copy (`globalThis.fetch.bind(globalThis)`), or copy it to a local variable and call that.
- Merge `headers` under the SDK's own headers. Header names are case-insensitive, so a user's `x-api-key` must not override or duplicate `apiKey`. Building a `Headers` object makes this straightforward.
- Validate in the constructor that `fetch` is a function when given, and throw a clear error otherwise.
- Document both options in the README options table. Add tests: custom headers arrive, SDK headers win on conflict, and the custom `fetch` is used for every attempt including retries.

### Additional Context

- Browsers: any extra header triggers a CORS preflight, and the relay only allows `Content-Type` (plus `X-API-Key` when a key is configured) in `Access-Control-Allow-Headers`. So in browsers this option mainly helps same-origin or proxied setups. The README should say so.
- Node 18+ and modern browsers all have a global `fetch`, so the `fetch` option is about control, not about runtimes without `fetch`.

### Code of Conduct

- [ ] I have joined the Discord server and will post updates there
- [ ] I have searched existing issues to avoid duplicates
