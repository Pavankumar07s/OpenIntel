# [BUG]: `RelayClient` accepts a base URL with a query string, fragment or credentials, then every request fails

## Bug Description

The constructor only checks that the base URL parses and uses `http:` or `https:` (`src/client.ts:53-61`). It then saves `parsedUrl.toString()` with trailing slashes removed (`:58`). `request()` builds each request URL by adding the path to the end of that string (`` `${this.baseUrl}${path}` ``, `:239`).

So any base URL that contains a query string or fragment gets its API path added after the `?` or `#`:

| Base URL passed to the constructor | URL actually requested for `health()` |
|---|---|
| `http://relay.test/?tenant=a` | `http://relay.test/?tenant=a/health` |
| `http://relay.test/relay/?k=v` | `http://relay.test/relay/?k=v/health` |
| `http://relay.test/#frag` | `http://relay.test/#frag/health` |
| `http://relay.test/?` (stray `?`) | `http://relay.test/?/health` |

Each of these sends `GET /` to the server, which answers `404 page not found`. Every call (`health`, `send`, `receive`, `poll`) then throws `RelayNotFoundError("404 page not found\n")`. Nothing points to the base URL as the cause.

A base URL with userinfo (`http://user:pass@relay.test`) also passes the constructor. But `fetch` refuses any URL that contains credentials, in Node and in browsers alike. So every call fails with `RelayNetworkError`:

- GET calls are retried first, so with the default `retries: 3` each one takes about 9 s to fail.
- The error message contains the full URL, password included: `Request cannot be constructed from a URL that includes credentials: http://user:s3cret@...`. The default `poll()` error handler writes that message to `console.error`.

A base URL with a sub-path and no query (`https://host/relay/`) works today and should keep working.

A smaller issue in the same block: protocol errors get the prefix twice, giving `Invalid base URL: Invalid base URL protocol. Must be http: or https:`. The inner `Error` thrown at `:56` is caught and wrapped again at `:60`.

## Steps to Reproduce

1. Start a ThruBox-Server on port 3000 (any config).
2. Run the script below with the package built from `main` (`npm run build`), or with `@aossie-org/thrubox-client@1.1.0`:

   ```js
   import { RelayClient } from '@aossie-org/thrubox-client';

   const bases = [
     'http://127.0.0.1:3000',            // control: works
     'http://127.0.0.1:3000/?tenant=x',
     'http://127.0.0.1:3000/#frag',
     'http://127.0.0.1:3000/?',
     'http://user:s3cret@127.0.0.1:3000',
   ];
   for (const base of bases) {
     const relay = new RelayClient(base, { retries: 0 }); // no constructor error for any of these
     try {
       await relay.health();
       console.log(base, 'OK');
     } catch (e) {
       console.log(base, '->', e.constructor.name, JSON.stringify(e.message));
     }
   }
   ```

3. **Expected:** the constructor throws a clear "invalid base URL" error for the last four values, or the requests are built correctly.
4. **Actual:** the constructor accepts all of them, and every request fails as shown below.

## Logs and Screenshots

Output against a real ThruBox-Server (`main` at `f026f22`):

```
http://127.0.0.1:3000 OK
http://127.0.0.1:3000/?tenant=x -> RelayNotFoundError "404 page not found\n"
http://127.0.0.1:3000/#frag -> RelayNotFoundError "404 page not found\n"
http://127.0.0.1:3000/? -> RelayNotFoundError "404 page not found\n"
http://user:s3cret@127.0.0.1:3000 -> RelayNetworkError "Request cannot be constructed from a URL that includes credentials: http://user:s3cret@127.0.0.1:3000/health"
```

With the default `retries: 3`, the credentials case takes about 8.9 s to fail for `health()`, against about 1 ms with `retries: 0`.

## Environment Details

- `@aossie-org/thrubox-client` `main` at `9afc707` (1.1.0)
- Node.js v24.22.1, Linux
- ThruBox-Server `main` at `f026f22`
- The `fetch` refusal of credentialed URLs is standard Fetch behaviour, so browsers fail the same way.

## Impact

Low - Minor inconvenience

## Proposed fix

In the constructor (`src/client.ts:52-61`):

- Reject `parsedUrl.username` / `parsedUrl.password` when either is non-empty. The error should say credentials are not supported and must not echo the URL.
- Reject a non-empty `parsedUrl.search` / `parsedUrl.hash`.
- Build the stored base from `parsedUrl.origin + parsedUrl.pathname`, with trailing slashes removed, instead of `parsedUrl.toString()`. This matters because `new URL('http://h/?').search` and `new URL('http://h/#').hash` are both `""`. A truthiness check alone therefore misses a stray `?` or `#`, but `origin + pathname` drops them.
- Throw each validation error once, so the message is not double-prefixed. For example, parse inside `try`/`catch` and do the protocol and component checks after it.

Do not switch to `new URL(path, base)` for request URLs. With a path that starts with `/`, it drops both the base sub-path (`/relay/`) and the query string.

Add tests for:

- the query, fragment, empty-`?`/`#` and credentials cases (the constructor throws)
- a sub-path base such as `http://host/relay/` (the request goes to `http://host/relay/health`)
- the existing trailing-slash case

A prototype of the change above passes the existing 30 tests unchanged.
