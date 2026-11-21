<!-- Team note, not issue text: do not file. Merged into TBC-13.md; this file is kept only as evidence. -->

# [BUG]: CONTRIBUTING says Node.js 18 is enough, but `npm test` cannot start on Node 18 (and nothing checks the declared `engines: >=18` floor)

### Bug Description

`package.json` declares `"engines": { "node": ">=18.0.0" }`, and CONTRIBUTING.md ("Getting Started > Prerequisites") tells contributors they need "Node.js 18 or later". The dev dependencies no longer support Node 18:

| Package (version in `package-lock.json`) | `engines.node` |
| --- | --- |
| vitest 5.0.5 | `^22.12.0 \|\| ^24.0.0 \|\| >=26.0.0` |
| eslint 10.12.0, @eslint/js 10.0.1 | `^20.19.0 \|\| ^22.13.0 \|\| >=24` |
| vite 8.3.5, rolldown 1.2.15 (used by vitest) | `^20.19.0 \|\| >=22.12.0` |
| tsup 8.5.1 | `>=18` |

On Node 18.20.8:

- `npm test` fails at startup (exit 1). rolldown imports `styleText` from `node:util`, and Node 18 does not have it.
- `npm run lint` works as long as ESLint has nothing to report. When it has a problem to print, it crashes with `TypeError: util.styleText is not a function` (exit 2) instead of printing the problem. That happens on `main` today because of the existing `preserve-caught-error` error at `src/client.ts:60`.
- `npm run build` (tsup) works.
- The built package works. I loaded `dist/index.js` and `dist/index.cjs` and ran `health()`, `send()` and `receive()` against a local `node:http` stub. I also checked that a hung server gives `RelayNetworkError: Request timed out` after the configured 400 ms and that a 503 is retried. Node 18.20.8 gave the same results as Node 24.22.1.

So "Works in Node.js 18+" (README, AGENTS.md) is still true for users of the package. But no CI job ever runs on Node 18: `version-release.yml` uses Node 22, and `danger.yml` and `codeql.yml` use Node 20. The test suite cannot run on Node 18 at all, so a future change that uses a newer runtime API would break Node 18 users without anyone noticing.

Node 18 reached end-of-life on 2025-04-30. Node 20 reached end-of-life on 2026-04-30.

### Steps to Reproduce

On `main` (9afc707):

```bash
npm ci
# Node 18 (any install works; the official image is used here)
docker run --rm -v "$PWD":/app -w /app node:18.20.8-bookworm-slim npm test
docker run --rm -v "$PWD":/app -w /app node:18.20.8-bookworm-slim npm run lint
docker run --rm -v "$PWD":/app -w /app node:18.20.8-bookworm-slim npm run build
```

**Expected:** a contributor who follows CONTRIBUTING's prerequisites can run the test suite and the linter.

**Actual:** `npm test` fails to start, and `npm run lint` crashes instead of reporting the problem it found. With Node 24.22.1 all 30 tests pass and lint reports the problem normally.

(Side note: Node 20.20.2 also passes all 30 tests, even though vitest 5 does not list Node 20 in its `engines`, so Node 20 support is not guaranteed.)

### Logs and Screenshots

`npm test` on Node 18.20.8:

```
 Startup Error
file:///.../node_modules/rolldown/dist/shared/create-bundler-option-DzNR1xQS.mjs:8
import { formatWithOptions, styleText } from "node:util";
                            ^^^^^^^^^
SyntaxError: The requested module 'node:util' does not provide an export named 'styleText'
    at ModuleJob._instantiate (node:internal/modules/esm/module_job:123:21)
    ...
    at async start (file:///.../node_modules/vitest/dist/chunks/cac.CLrZKo1g.js:2367:27)
```

`npm run lint` on Node 18.20.8:

```
Oops! Something went wrong! :(

ESLint: 10.12.0

TypeError: util.styleText is not a function
    at util.styleText.validateStream (.../node_modules/eslint/lib/cli-engine/formatters/stylish.js:22:9)
```

### Environment Details

- Linux x64, `main` at 9afc707
- Node 18.20.8 (npm 10.8.2), compared with Node 20.20.2 and 24.22.1
- vitest 5.0.5, eslint 10.12.0, tsup 8.5.1

### Impact

Low - Minor inconvenience

### Proposed fix

1. In CONTRIBUTING.md "Prerequisites", say that development needs Node.js 22.12 or later, which is what vitest 5 requires. Keep the README/AGENTS.md runtime statement, or change it together with option 3.
2. If the project keeps `engines: >=18`: once a pull-request CI workflow exists, run build, lint and test there on Node 22 and 24. Add a small dependency-free smoke job on the declared floor (Node 18) that loads the built `dist/index.js` and `dist/index.cjs` from the Node 22 job and calls the client against a `node:http` stub server, for example `node tests/smoke/dist.mjs`.
3. Or raise `engines.node` to `>=22` and update README, AGENTS.md and CONTRIBUTING. `>=20` would point at a line that is already end-of-life. Users with `engine-strict` on older Node would then fail to install, so release the change as a breaking change.

### Code of Conduct

- [ ] I have joined the Discord server and will post updates there
- [ ] I have searched existing issues to avoid duplicates
