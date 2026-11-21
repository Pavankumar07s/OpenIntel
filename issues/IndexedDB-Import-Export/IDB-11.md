# [FEATURE]: Return a result from importDB (imported vs. aborted, record counts)

### Feature and its Use Cases

`importDB` resolves with `undefined` in every case (`src/core/importer.ts:393`, `Promise<void>`):

- when the import completes,
- when `onBeforeImport` returns `false` and the import is skipped (`if (!proceed) return;`, line 416),
- when the import is skipped because no selected store exists in the target database (`if (targetStores.length === 0) return;`, line 431).

Checked on `main` (4b90eca) with the repo's `fake-indexeddb` setup:

```
onBeforeImport returns false        -> importDB resolved: undefined   (nothing written)
onBeforeImport returns nothing (JS) -> importDB resolved: undefined   (nothing written)
no hook, import completes           -> importDB resolved: undefined
```

So code after `await importDB(...)` can't tell whether anything was written, unless it tracks the hook's answer in its own variable. A typical flow is a confirm dialog in `onBeforeImport`, then a "Restore complete" toast after the `await`. Right now that flow shows the toast even when the user pressed Cancel. In plain JavaScript, a hook that forgets to `return true` also skips the restore with no signal (TypeScript catches this through the `boolean` return type).

**Proposal:** resolve with a small result object instead of `void`:

```ts
interface ImportResult {
  status: 'imported' | 'aborted';
  /** Stores that received records, with the number written to each. */
  recordCounts: Record<string, number>;
  /** Stores that had to be created during the import. */
  storesCreated: string[];
  /** Version of the database after the import. */
  databaseVersion: number;
}
```

- This doesn't break existing code: callers that ignore the return value keep working, and `Promise<void>` → `Promise<ImportResult>` is source-compatible in TypeScript.
- Export the `ImportResult` type from `src/index.ts` and document it in the README.
- Tests: an aborted import returns `status: 'aborted'`; a merge that adds a store lists it in `storesCreated`; counts match what was written.

Optional, for discussion: warn, or throw a `TypeError`, when `onBeforeImport` returns something that is not a boolean. This would change behaviour for hooks that return truthy non-boolean values today, so it may be better left out.

### Additional Context

- `onBeforeImport` was added in #68 / PR #87. This builds on that hook. No existing issue or PR covers a return value.

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
