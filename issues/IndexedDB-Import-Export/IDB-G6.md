# [FEATURE]: Let exportDB reject storeNames that don't exist in the database (a typo currently produces an incomplete or empty backup)

## Feature and its Use Cases

**Current behaviour**

When `exportDB({ dbName, storeNames })` is given a name that isn't an object store in the database, it drops that name and says nothing (`src/core/exporter.ts:126-128`). If none of the names match, it returns a normal-looking envelope with `schema: {}` and `stores: {}` (`src/core/exporter.ts:130-139`). This is intended for now: there is a test for it (`tests/exporter.test.ts`, "ignores non-existent storeNames in filter"). But it isn't documented anywhere a caller would look. `ExportOptions.storeNames` (`src/types/index.ts:88-92`), the `exportDB` JSDoc and the README export example only say "export only specific stores".

Because `storeNames` is a plain `string[]`, TypeScript can't catch a typo, a wrong case, or a list that went stale after a store was renamed in a later app version. The backup records only the stores it contains, not the ones that were requested, so nothing in the file shows that a store was skipped. The caller finds out at restore time. With `strategy: 'overwrite'`, which deletes the database and recreates it from the backup as documented, the skipped stores are gone afterwards.

**Minimal repro** (browser console, on `main` / v1.1.0)

```js
import { exportDB, importDB } from '@aossie-org/idb-backup';

// Setup: DB 'mail' v1 with stores 'messages' and 'attachments', one record each
await new Promise((res, rej) => {
  const r = indexedDB.open('mail', 1);
  r.onupgradeneeded = () => {
    r.result.createObjectStore('messages', { keyPath: 'id' });
    r.result.createObjectStore('attachments', { keyPath: 'id' });
  };
  r.onsuccess = () => {
    const t = r.result.transaction(['messages', 'attachments'], 'readwrite');
    t.objectStore('messages').put({ id: 1, subject: 'hi' });
    t.objectStore('attachments').put({ id: 1, file: 'a.pdf' });
    t.oncomplete = () => { r.result.close(); res(); };
  };
  r.onerror = () => rej(r.error);
});

// 1. One misspelled name: that store is silently left out
const partial = await exportDB({ dbName: 'mail', storeNames: ['messages', 'attachements'] });
Object.keys(partial.stores);          // ["messages"]

// 2. All names wrong (case typo): resolves with an empty backup
const typo = await exportDB({ dbName: 'mail', storeNames: ['Messages'] });
// { schema: {}, stores: {} }

// 3. Restoring the partial backup with overwrite removes 'attachments'
await importDB({ dbName: 'mail', backupData: partial, strategy: 'overwrite' });
// DB is now { version: 1, stores: ["messages"] }

// 4. Restoring the empty backup with overwrite leaves an empty database
await importDB({ dbName: 'mail', backupData: typo, strategy: 'overwrite' });
// (an onBeforeImport hook would receive storeNames: [], recordCounts: {})
// DB is now { version: 1, stores: [] }
```

**Expected:** the export step tells the caller that a requested store doesn't exist, e.g.
`Unknown object store(s) "attachements" in storeNames (available: attachments, messages)`, instead of producing a backup that looks complete.

**Actual:** the export resolves without any error or warning. The missing stores only become visible after an overwrite restore has already removed them.

**Environment**

- Chrome 155.0.8059.79 (headless), Linux
- `@aossie-org/idb-backup` at `main` (4b90eca, v1.1.0)
- The same output under the repo's Vitest setup (happy-dom + fake-indexeddb)

## Additional Context

**Proposed fix** (no breaking change in 1.x)

1. Add `strict?: boolean` to `ExportOptions`. When it is `true`, `exportDB` throws before reading any data if any entry of `storeNames` isn't in `db.objectStoreNames`, and the error lists the unknown and the available names. The default stays `false`, so the existing "ignores non-existent storeNames" behaviour and test are unchanged. Making it the default could be considered for the next major version.
2. Document the current behaviour in `ExportOptions.storeNames`, the `exportDB` JSDoc and the README export section, the same way the import note already says "Names not present in the backup are ignored".
3. Tests in `tests/exporter.test.ts`: strict mode with a partly unknown list and an all-unknown list, and a check that strict mode with valid names behaves like today.
4. Optional, for symmetry: the same `strict` flag on `ImportOptions.storeNames`, which currently ignores unknown names in the same way.

Until then, callers can protect a restore with the existing hook, e.g. `onBeforeImport: (s) => s.storeNames.length > 0`. That only catches the all-unknown case, though, not a single skipped store.

Files: `src/core/exporter.ts`, `src/types/index.ts`, `README.md`, `tests/exporter.test.ts`
