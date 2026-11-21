# [FEATURE]: Add an onProgress callback to exportDB and importDB for large databases

### Feature and its Use Cases

Exporting or importing a large database (an offline cache, a wallet with long transaction history, stored files) gives no feedback until the promise settles:

- `exportDB` reads each store with one cursor step per record (`readAllRecords`, `src/core/exporter.ts:41-65`) and builds the whole serialized backup in memory before resolving.
- `importDB` deserializes and queues every record of every store in one `readwrite` transaction (`insertRecords`, `src/core/importer.ts:297-340`) and resolves only when the transaction completes.

So an app can only show a spinner. It can't show a progress bar or tell the user how much is left.

**Proposal:** add an optional progress callback to both APIs:

```ts
interface ProgressInfo {
  phase: 'export' | 'import';
  storeName: string;
  done: number;   // records processed in this store
  total: number;  // from store.count() on export, from the backup on import
}

exportDB({ dbName, onProgress?: (p: ProgressInfo) => void });
importDB({ dbName, backupData, strategy, onProgress?: (p: ProgressInfo) => void });
```

- Call it at a throttled interval (for example every N records, and once at the end of each store), so it doesn't slow down the loop.
- Keep everything inside the existing single transaction. Progress reporting must not change the current atomicity: no splitting the import across several transactions.
- On export, `store.count()` can run in the same `readonly` transaction to get `total`.

### Additional Context

- No existing issue or PR covers progress reporting.
- Tests: with `fake-indexeddb`, check that `onProgress` is called with a non-decreasing `done`, that the last call for each store has `done === total`, and that the import result is unchanged when a callback is passed.

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
