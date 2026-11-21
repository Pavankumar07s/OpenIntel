# [BUG]: `npm run lint` and `npm run format:check` fail on a clean checkout of `main`

### Bug Description

CONTRIBUTING.md ("3. Test Your Changes") asks contributors to run `npm run lint` and `npm run format:check` before opening a PR. On a clean checkout of `main`, both commands exit with code 1. Contributors cannot tell whether their own change broke them, and recent PRs (#81, #83) report them as "pre-existing failures".

1. **ESLint**, 1 error, from the `preserve-caught-error` rule in the ESLint 10 recommended config:
   ```
   src/client.ts
     60:7  error  There is no `cause` attached to the symptom error being thrown  preserve-caught-error
   ```
   The obvious fix, `new Error(msg, { cause: err })`, does not compile with the current `tsconfig.json` (`"lib": ["ES2020", "DOM"]`). The `ErrorOptions` overload is only in the ES2022 lib:
   ```
   src/client.ts(60,96): error TS2554: Expected 0-1 arguments, but got 2.
   ```
   With that change, `npx tsc --noEmit` fails, and so does `npm run build` ("DTS Build error"). Open PR #83 includes this same one-line change at line 60 and does not change `tsconfig.json`.

2. **Prettier**, 4 files: `src/client.ts`, `src/errors.ts`, `src/index.ts`, `tests/client.test.ts`. The main cause is that `.prettierrc` sets `"singleQuote": true` while all of `src/` and `tests/` use double quotes. Line wrapping also differs: with `--single-quote=false`, `src/client.ts` and `tests/client.test.ts` still fail.

Related: `tsconfig.json` only includes `src/`, so the test file is never type-checked. Type-checking it with the project's own strict options (`noUncheckedIndexedAccess`) gives 13 errors (12 x TS2532 "Object is possibly 'undefined'", 1 x TS2488).

### Steps to Reproduce

1. `git clone https://github.com/AOSSIE-Org/ThruBox-Client && cd ThruBox-Client && npm ci`
2. `npm run lint`: exits 1 with the error above.
3. `npm run format:check`: exits 1 and lists 4 files.
4. Optional: put `{ cause: err }` on `src/client.ts:60`, then run `npx tsc --noEmit` (TS2554) and `npm run build` (DTS build error).

### Logs and Screenshots

```
$ npm run format:check
Checking formatting...
[warn] src/client.ts
[warn] src/errors.ts
[warn] src/index.ts
[warn] tests/client.test.ts
[warn] Code style issues found in 4 files. Run Prettier with --write to fix.
```

### Environment Details

- `main` at 9afc707, dependencies from `package-lock.json` (eslint 10.12.0, @eslint/js 10.0.1, prettier 3.9.10, typescript 5.9.3)
- Node.js v24.22.1, npm 11.19.1, Linux

### Impact

Low - Minor inconvenience

### Proposed fix

- **Lint:** add `"ES2022.Error"` to `lib` (`"lib": ["ES2020", "ES2022.Error", "DOM"]`) and pass `{ cause: err }` at `src/client.ts:60`. I checked this locally: `tsc --noEmit`, `npm run build` (including DTS), `npm run lint` and all 30 tests pass. Runtimes that do not know `cause` ignore the option, so nothing breaks on Node 18.
- **Prettier:** the maintainers should decide on the quote style.
  - Run `npm run format` once in a formatting-only commit. This changes about 200 lines in all 4 files (+197 / -228), so it will conflict with open PRs #81 and #83.
  - Or set `"singleQuote": false` in `.prettierrc` to match the existing code, then run `npm run format`. This changes 2 files (+30 / -61).
- **Tests type-check (optional):** add a `tsconfig.test.json` that includes `tests/` and fix the 13 errors, so a future CI job can type-check the tests too.

### Code of Conduct

- [ ] I have joined the Discord server and will post updates there
- [ ] I have searched existing issues to avoid duplicates
