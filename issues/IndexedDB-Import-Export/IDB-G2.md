# [FEATURE]: Let exportDB/importDB time out or be aborted instead of waiting forever behind a queued version change

### Feature and its Use Cases

**Problem**

IndexedDB handles the open and delete requests for a database in order. If a version change or delete is waiting because another connection is still open, every later `indexedDB.open()` or `deleteDatabase()` for that database waits behind it. These later requests get no `blocked` event of their own. They only run once the connection that blocks the first request closes.

`exportDB` and `importDB` always open (or delete) the database themselves:

- `src/core/exporter.ts:76` calls `indexedDB.open(dbName)`.
- `src/core/importer.ts:199` makes the merge probe, `indexedDB.open(dbName)`.
- `src/core/importer.ts:49` and `:166` make the overwrite `deleteDatabase`.

`ExportOptions` and `ImportOptions` (`src/types/index.ts:85-93` and `:119-163`) give no way to stop waiting. In that state the returned promise never settles: there is no result and no error, so the `onblocked` handlers in the importer never fire. A UI that awaits the call keeps its spinner forever.

This happens when some connection ignores `versionchange`, which is the default for raw IndexedDB and for `idb`'s `openDB` without a `blocking` callback, and one of these is also true:

1. **This library's own earlier import was blocked (one page).** The app keeps its connection open. `importDB` rejects with "deletion blocked" (overwrite) or "upgrade blocked" (merge, when stores are added). That rejected request cannot be cancelled and stays queued. From then on, every `exportDB`/`importDB` call in the page hangs until the app's connection closes. The pre-check that PR #130 adds stays queued the same way, so the follow-up `exportDB` still hangs with #130 applied.
2. **Another tab is upgrading.** An old tab holds a connection while a newly deployed tab calls `open(name, v + 1)`. In the old tab, `exportDB` and `importDB` (export, merge and overwrite) never settle.

The app's own new `indexedDB.open()` waits in the same way. This is how IndexedDB works, and the library behaves exactly like the raw API. The gap is that a caller has no way to give up or get an error.

**Steps to reproduce** (Vitest + `fake-indexeddb`, as in `tests/importer.test.ts`)

```ts
import { it } from 'vitest';
import { exportDB } from '../src/core/exporter.js';
import { importDB } from '../src/core/importer.js';
import { setupFakeIDB, uniqueDBName, createTestDB } from './helpers/idb-helpers.js';
setupFakeIDB();

const settled = (p: Promise<unknown>, ms = 500) =>
  Promise.race([
    p.then(() => 'resolved', (e: Error) => `rejected: ${e.message}`),
    new Promise((r) => setTimeout(() => r('pending'), ms)),
  ]);

it('export after a blocked import', async () => {
  const dbName = uniqueDBName('queued');
  const db = await createTestDB(dbName, 1, [
    { name: 'users', keyPath: 'id', records: [{ value: { id: 1, name: 'Alice' } }] },
  ]); // the app's own connection, kept open
  const backup = await exportDB({ dbName });

  console.log(await settled(importDB({ dbName, backupData: backup, strategy: 'overwrite' })));
  console.log(await settled(exportDB({ dbName })));
  db.close();
});

it('export while another connection waits to upgrade', async () => {
  const dbName = uniqueDBName('queued2');
  const db = await createTestDB(dbName, 1, [
    { name: 'users', keyPath: 'id', records: [{ value: { id: 1, name: 'Alice' } }] },
  ]);
  const backup = await exportDB({ dbName });
  const upgrade = indexedDB.open(dbName, 2); // e.g. a newer build of the app in another tab
  await new Promise((r) => (upgrade.onblocked = r));

  console.log(await settled(exportDB({ dbName })));
  console.log(await settled(importDB({ dbName, backupData: backup, strategy: 'merge' })));
  db.close();
});
```

Output:

```
rejected: Database "queued-…" deletion blocked. Close all other connections to this database and try again.
pending
pending
pending
```

**Expected:** a way for the caller to bound the wait and get an error, for example a timeout or an `AbortSignal`.

**Actual:** the promise stays pending until the blocking connection closes. The library gives no event or error, and a caller cannot cancel it.

**Environment**

- `main` at 4b90eca.
- Vitest with `fake-indexeddb` 6.2.5 (the output above), Node 24.
- Google Chrome 155.0.8059.79: both scenarios, with two pages in one profile loading the built `dist/index.js`. Each call was still pending after 10 s, and the app's existing connection could still read and write. In Chrome, closing or navigating away from the page that had requested the upgrade did not release the wait. Only closing the blocking connection did.

**Proposed fix**

- Add an optional `signal?: AbortSignal` (or `timeoutMs`) to `ExportOptions` and `ImportOptions`. On abort, reject with an `AbortError`. A queued request cannot be withdrawn, so mark it as cancelled: abort the upgrade transaction in `onupgradeneeded` and close the connection in `onsuccess` if it runs later.
- `deleteDatabase()` cannot be cancelled at all. The overwrite path must therefore check the signal only *before* it issues the delete, for example during the pre-check that PR #130 adds. Rejecting after the delete has been issued would bring back #126: the import reports a failure but deletes the database later.
- Optional: accept `db?: IDBDatabase` in `exportDB`, and in merge imports that add no stores, so the library reads through the caller's open connection, which the queue doesn't affect. Imports that change the schema still need their own version change.
- In the README, recommend that apps close their connection on `versionchange` (`db.onversionchange = () => db.close()`).
- Tests: `fake-indexeddb` reproduces the queueing, so the cases above can become Vitest tests.

### Additional Context

- #126 / PR #130: blocked overwrite import. #130 makes the blocked case reject with the database left unchanged. Its pre-check request still stays queued, so later calls hang as in scenario 1.
- Prior art: `dexie-export-import`'s `exportDB(db)` and `importInto(db, blob)` work on the caller's already-open Dexie instance.

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
