# [FEATURE]: Add per-record `filter` and `transform` options to `exportDB` and `importDB`

## Feature and its Use Cases

**What the feature is**

Today the only way to narrow a backup is by whole store (`storeNames` on both `ExportOptions` and
`ImportOptions`). There is no option for selecting or rewriting individual records:

- `ExportOptions` (`src/types/index.ts:85-93`) is `{ dbName, storeNames? }`.
- `ImportOptions` (`src/types/index.ts:119-162`) is `{ dbName, backupData, strategy, storeNames?, onBeforeImport? }`.

Proposal: add two optional callbacks to both option types.

```ts
filter?: (storeName: string, value: unknown, key: unknown) => boolean;
transform?: (storeName: string, value: unknown, key: unknown) => { value: unknown; key?: unknown };
```

- On export they run on the value read from the cursor, **before** `serialize()`.
- On import they run **after** `deserialize()`, before `put()`/`add()`.
- Callbacks receive native values (`Date`, `Map`, `Uint8Array`, ...), not the `__type` tagged form.
- When neither option is set, behaviour does not change.

Optionally, in the same change: `includeData?: boolean` (default `true`) on `ExportOptions`. With
`false`, `exportDB` returns the full `schema` and empty `stores` without reading any records.

**How users would benefit / scenarios**

- Drop tokens or other secrets from a backup before it leaves the device.
- Export or restore only one account's rows from a shared store.
- Leave out cache rows that match a predicate instead of the whole store.
- Rename or migrate a field while importing.
- Export a schema-only template (with `includeData: false`).

**Why not do this in user code?**

You can do it today, but only by editing the envelope yourself. Callers have to loop over
`backup.stores[name]`, and they work with the tagged values unless they call the exported
`deserialize`/`serialize` on every record:

```ts
import { exportDB, importDB, serialize, deserialize } from '@aossie-org/idb-backup';

const backup = await exportDB({ dbName: 'app' });
backup.stores.users = backup.stores.users
  .filter((r) => (deserialize(r.value) as any).account === 'a')
  .map((r) => {
    const v = deserialize(r.value) as any;
    delete v.token;
    return { key: r.key, value: serialize(v) };
  });
```

This works. With Vitest + fake-indexeddb on `main` (4b90eca), the filtered backup restores one user
with `at` still a `Date` and no `token` field. But it ties callers to the `stores` and tagged-value
layout. #67 gave the same reason for adding `storeNames` to `ImportOptions`: "filter
`backupData.stores` before calling `importDB` ... works but couples the caller to the envelope shape."
These options do for records what #67/#73 did for stores. Built-in filtering also means dropped
records are never serialized.

There is also no schema-only export today. `exportDB({ dbName, storeNames: [] })` returns
`{"schema":{},"stores":{}}` (`src/core/exporter.ts:130-139`). To get a schema-only template you have
to export everything and then empty `stores` yourself.

**Current type-check against the built `dist/index.d.ts` (v1.1.0)**

```ts
await exportDB({ dbName: 'db', filter: (store: string, value: unknown) => true });
// error TS2353: Object literal may only specify known properties, and 'filter' does not exist in type 'ExportOptions'.

await importDB({ dbName: 'db', backupData: {} as any, strategy: 'merge',
  transform: (s: string, v: unknown) => ({ value: v }) });
// error TS2353: Object literal may only specify known properties, and 'transform' does not exist in type 'ImportOptions'.

await exportDB({ dbName: 'db', includeData: false });
// error TS2353: Object literal may only specify known properties, and 'includeData' does not exist in type 'ExportOptions'.
```

## Additional Context

**Design points to settle**

- **Sync vs async.** `readAllRecords` serializes inside the cursor's `onsuccess` while a read-only
  transaction is open. If an export callback awaits non-IndexedDB work, the transaction auto-commits.
  So either keep the callbacks synchronous, or read raw values first and apply the callbacks after the
  cursor finishes. This also means a sync `transform` cannot read `Blob` contents on the main thread,
  so it is not a substitute for fixing unsupported types in the serializer (#131, #132, #124).
- **Keys.** For inline-key stores, IndexedDB takes the key from the value. A `key` returned by
  `transform` should only be honoured for out-of-line stores (or rejected otherwise).
- **`onBeforeImport` summary.** Decide whether `recordCounts` comes before or after `filter`, and
  document it. Pre-filter is cheaper, post-filter is more accurate.
- **`overwrite` + `filter`.** Records filtered out on an `overwrite` import are absent afterwards,
  because the database is recreated. This is the same caveat already documented for `storeNames`.

**Encryption (docs only, no API change needed)**

PRIVACY.md (lines 75-79) suggests encrypting backups "in your application before it leaves the
device". This already works with the current helpers, because `downloadJSON<T>` and
`readFileAsJSON<T>` are generic. Encrypt `JSON.stringify(backup)` with WebCrypto AES-GCM, pass
`{ iv, data }` (base64) to `downloadJSON`, and reverse it after `readFileAsJSON`. I checked this round
trip: the downloaded file contains no plaintext, and the decrypted backup imports correctly. A short
README recipe would make this easier to find. The file helpers do not need new hooks for it.

**Prior art:** dexie-export-import 4.4.2 has `filter(table, value, key)` and
`transform(table, value, key) => { value, key }` on both export and import options.

**Files likely touched:** `src/types/index.ts`, `src/core/exporter.ts`, `src/core/importer.ts`,
`README.md`, `tests/exporter.test.ts`, `tests/importer.test.ts`, `tests/round-trip.test.ts`.

**Environment:** `@aossie-org/idb-backup` 1.1.0 (`main` @ 4b90eca), TypeScript strict type-check
against `dist/index.d.ts`; runtime checks with Vitest 5.0.1, happy-dom and fake-indexeddb.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [x] I have searched existing issues to avoid duplicates
