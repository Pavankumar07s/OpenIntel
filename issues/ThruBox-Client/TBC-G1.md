# [BUG]: Error `instanceof` checks fail when both the CJS and ESM builds are loaded (dual-package hazard)

### Bug Description

`package.json` sends `import` to `dist/index.js` and `require` to `dist/index.cjs`. tsup bundles each of these separately (`format: ['esm', 'cjs']`, `splitting: false`), so each file has its own copy of `RelayClient` and of every error class.

If a single Node process or bundle loads both files, an error thrown by one copy is not an `instanceof` the other copy's classes. For example, an ESM app might use a CommonJS module or dependency that `require`s the SDK. In that case `e instanceof RelayRateLimitError` / `RelayPayloadTooLargeError` / `RelayError` is `false`, even though `e.name` is correct. The README's Error Handling section recommends exactly this pattern, so those branches are skipped without any warning, and rate-limit or payload-size handling never runs.

TypeScript gives no clean fallback. `name` is declared as the inherited `string`, so checking `e.name === 'RelayRateLimitError'` does not narrow `e`. Also, `RelayNetworkError`, `RelayNotFoundError`, `RelayPayloadTooLargeError` and `RelayUnauthorizedError` add no members of their own, so TypeScript treats them as structurally identical. For example, it accepts a `RelayNotFoundError` where a `RelayUnauthorizedError` parameter is declared.

### Steps to Reproduce

1. On `main`, run `npm run build && npm pack`. Extract the tarball into `node_modules/@aossie-org/thrubox-client` of an empty project whose `package.json` is `{"type":"module"}`.
2. `relay-factory.cjs`:
   ```js
   const { RelayClient } = require('@aossie-org/thrubox-client');
   module.exports.makeRelay = (u) => new RelayClient(u, { retries: 0, timeout: 500 });
   ```
3. `app.mjs` (nothing listens on the port, so `health()` throws a `RelayNetworkError`):
   ```js
   import { RelayClient, RelayError, RelayNetworkError } from '@aossie-org/thrubox-client';
   import { createRequire } from 'node:module';
   const { makeRelay } = createRequire(import.meta.url)('./relay-factory.cjs');
   for (const [label, mk] of [['CJS-created', makeRelay], ['ESM-created', (u) => new RelayClient(u, { retries: 0, timeout: 500 })]]) {
     try { await mk('http://127.0.0.1:38111').health(); }
     catch (e) { console.log(label, e.name, e instanceof RelayError, e instanceof RelayNetworkError); }
   }
   ```
   Run `node app.mjs`.
4. Bundler variant: `app2.mjs` imports `{ RelayError }` from the package and `{ makeRelay }` from `./relay-factory.cjs`, then checks `e instanceof RelayError`. Bundle it with `esbuild app2.mjs --bundle --platform=browser --format=esm --outfile=out.mjs` and run `node out.mjs`.
5. Types (`tsc --strict`, `module`/`moduleResolution: nodenext`):
   ```ts
   function f(e: RelayError) { if (e.name === 'RelayRateLimitError') return e.retryAfter; }  // TS2339
   function needs401(e: RelayUnauthorizedError) {}
   needs401(new RelayNotFoundError('x'));  // compiles
   ```

**Expected:** an error thrown by the SDK passes `instanceof` against the classes the app imports, however the SDK was loaded.
**Actual:** `instanceof` is `false` whenever the error comes from the other module format's copy.

### Logs and Screenshots

```
$ node app.mjs
CJS-created RelayNetworkError false false
ESM-created RelayNetworkError true true

$ node out.mjs
RelayNetworkError instanceof RelayError = false
```

The esbuild bundle contains both copies of the code: `// node_modules/@aossie-org/thrubox-client/dist/index.cjs` (`var RelayError2 = class extends Error`) and `// node_modules/@aossie-org/thrubox-client/dist/index.js` (`var RelayError = class extends Error`).

```
t.mts(2,83): error TS2339: Property 'retryAfter' does not exist on type 'RelayError'.
```

### Environment Details

- `@aossie-org/thrubox-client` 1.1.0, built from `main` (9afc707)
- Node.js 24.22.1, Linux
- esbuild 0.27.7, TypeScript 5.9.3

### Impact

Low - Minor inconvenience. This only happens when both builds end up in the same process or bundle. When it does, the documented error handling is skipped without any warning.

### Proposed fix

- **Keep one copy of the classes.** Make the ESM entry a thin wrapper around the CJS build, and keep `import` pointing at it:
  ```js
  // dist/index.js
  export { RelayClient, RelayError, RelayNetworkError, RelayRateLimitError, RelayNotFoundError, RelayPayloadTooLargeError, RelayUnauthorizedError } from './index.cjs';
  ```
  With this file in place, both lines of the Node repro print `true true`, and the esbuild bundle has one `class extends Error`. tsup's CJS output already annotates its export names, so Node detects the named exports. The cost is that ESM bundlers get CommonJS and lose tree-shaking, which hardly matters for a package of about 10 KB. Both `import` and `require` entry points still exist, so this does not drop the ESM entry. It should land after the `require` → `index.d.cts` types fix, because both change the same `exports` block.
- **Alternative, only if the wrapper is not acceptable:** keep both builds and make the checks tolerant, with a `Symbol.for(...)` brand and `static [Symbol.hasInstance]` on `RelayError`, or exported guards such as `isRelayError(e)` / `isRelayRateLimitError(e)`. That changes `src/errors.ts` and the public API, so it would need its own discussion.
- **Types (separate follow-up, not part of this fix):** declare a literal `name` on each subclass (e.g. `override readonly name = 'RelayRateLimitError' as const`). This alone stops TypeScript from treating the subclasses as interchangeable. To make `switch (e.name)` narrow, also export a union type of the subclasses (or the guards above). A literal `name` does not narrow a value typed as `RelayError` or `unknown`.
- Add a smoke test (next to the dist smoke test, e.g. under `tests/smoke/`) that loads the built package through both `require` and `import` and checks `instanceof`. With the wrapper, the README's documented `instanceof` pattern works as written, so the fix itself only touches `package.json`, `tsup.config.ts` and the smoke test.
