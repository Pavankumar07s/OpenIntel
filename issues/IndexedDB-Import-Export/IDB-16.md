# [FEATURE]: Add tests for the untested exportDB/importDB error paths, and type-check the test files

### Feature and its Use Cases

Coverage measured on `main` (4b90eca) with `@vitest/coverage-v8@5.0.1` (`npx vitest run --coverage --coverage.include='src/**'`). All 98 tests pass:

```
File          | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
All files     |   91.66 |    90.62 |   77.92 |   91.66 |
 exporter.ts  |   83.67 |       60 |   76.92 |   83.67 | 60,79-81,90-94
 importer.ts  |   88.48 |    84.09 |   69.04 |   88.63 | 56,60,109-110,182,186,224,226,244,248,269,277,334,451,454
 serialization|   98.97 |    98.57 |     100 |   98.92 | 115
 file.ts      |   95.45 |      100 |      80 |   95.45 | 73
```

Concrete gaps:

1. **`exportDB` on a database that doesn't exist** (`exporter.ts:78-94`) has no test. This path was added in PR #100: it aborts the implicit creation and rejects with `Database "…" does not exist`. The importer side of that PR is tested, the exporter side is not. Checked by hand: it rejects with that message and leaves no database behind. A test would lock that in.
2. **Insert failure inside the import transaction** (`importer.ts:334`, plus the transaction `onerror`/`onabort` handlers at 451/454): no test makes a write fail. Examples: two backup records with the same primary key under `"overwrite"`, which uses `add()`, or two records that share a value on a `unique` index.
3. **`readFileAsJSON` read error** (`file.ts:73`, `FileReader.onerror`): untested.
4. **Defensive overwrite branch in `createStoresFromSchema`** (`importer.ts:105-113`): unreachable in practice, because `"overwrite"` always deletes the database before recreating it. Either remove it or cover it.
5. **Test files are never type-checked.** `tsconfig.json` has `"include": ["src"]` and `"exclude": [..., "tests"]`, so `npx tsc --noEmit` skips `tests/`. They type-check cleanly today, checked with a temporary config that includes `tests/`, but nothing keeps it that way.

The `blocked` branches (`importer.ts:60`, 186, 226, 248, 272) are being reworked in PR #130, and the export serialization-failure path in draft PR #128. Those PRs bring their own tests, so this issue leaves them out.

**Proposal:**
- Add the tests for items 1–3 to `tests/exporter.test.ts`, `tests/importer.test.ts` and `tests/utils/file.test.ts` with the existing `fake-indexeddb`/`happy-dom` helpers.
- Remove the unreachable branch in item 4, or add a comment explaining why it stays.
- Add a `tsconfig.test.json` that extends `tsconfig.json` and includes `tests/`, plus a `typecheck` script (`tsc -p tsconfig.test.json`), so CI can run it.

### Additional Context

- `vitest.config.ts` already has a `coverage` block, but `@vitest/coverage-v8` is not a devDependency and there is no script for it. Adding both would make these numbers easy to reproduce.

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
