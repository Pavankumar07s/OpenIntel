# [BUG]: A blocked merge import still applies its schema upgrade after the other connection closes, and leaves a connection open

### Bug Description

When `importDB` with `strategy: 'merge'` needs to add stores, it opens the database at `currentVersion + 1` (`src/core/importer.ts:235`). If another connection is open and doesn't close on `versionchange` (another tab, or the app's own connection), the request fires `blocked` and `importDB` rejects with:

```
Database "x" upgrade blocked. Close all other connections and try again.
```

But `blocked` doesn't cancel an open request (`src/core/importer.ts:247`). The request stays queued. As soon as the other connection closes, three things happen:

1. `onupgradeneeded` still runs `createStoresFromSchema`, so the version is bumped and the new stores are created **empty**, even though the import was reported as failed.
2. `onsuccess` calls `resolve()` on the promise that already rejected, so the new `IDBDatabase` is never closed. It has no `versionchange` handler, so until it is garbage-collected or the page reloads it blocks every later version change or delete of that database. That includes the app's next schema upgrade and a later `importDB(..., 'overwrite')`, which then fails with "deletion blocked".
3. Until the other connection closes, the abandoned request also holds the database's connection queue, so every later `indexedDB.open(dbName)` waits with no event. A retry of `importDB` waits too.

Retrying the merge after the other connection has closed does import the records, so no data is lost. The problems are the half-applied schema change behind a rejected promise and the connection that stays open.

This is the merge-path counterpart of #126 (the overwrite `deleteDatabase` path). PR #130 only changes the overwrite path.

### Steps to Reproduce

Vitest, with `fake-indexeddb` as in `tests/importer.test.ts`:

```ts
import { it, expect } from 'vitest';
import { importDB } from '../src/core/importer.js';
import { setupFakeIDB, uniqueDBName, createTestDB } from './helpers/idb-helpers.js';
import type { ExportFormat } from '../src/types/index.js';
setupFakeIDB();

const openDB = (name: string) =>
  new Promise<IDBDatabase>((resolve, reject) => {
    const req = indexedDB.open(name);
    req.onsuccess = () => resolve(req.result);
    req.onerror = () => reject(req.error);
  });

it('a blocked merge import leaves the database unchanged', async () => {
  const dbName = uniqueDBName('blocked-merge');
  // Another connection that does not close on versionchange (e.g. another tab)
  const other = await createTestDB(dbName, 1, [{ name: 'users', keyPath: 'id' }]);

  const backup = {
    backupVersion: 1, databaseName: dbName, databaseVersion: 1, exportedAt: new Date().toISOString(),
    schema: {
      users: { keyPath: 'id', autoIncrement: false, indexes: [] },
      tags: { keyPath: 'id', autoIncrement: false, indexes: [] }, // new store -> version bump
    },
    stores: { users: [], tags: [{ key: 'a', value: { id: 'a' } }] },
  } as ExportFormat;

  await expect(importDB({ dbName, backupData: backup, strategy: 'merge' })).rejects.toThrow(/upgrade blocked/);

  other.close(); // do what the error message says
  await new Promise((r) => setTimeout(r, 50));

  const db = await openDB(dbName);
  const state = { version: db.version, stores: Array.from(db.objectStoreNames) };
  db.close();

  const later = await new Promise<string>((resolve) => {
    const req = indexedDB.open(dbName, 10); // e.g. the app's next schema upgrade
    req.onblocked = () => resolve('blocked');
    req.onsuccess = () => { req.result.close(); resolve('ok'); };
  });

  expect(state).toEqual({ version: 1, stores: ['users'] }); // actual: { version: 2, stores: ['tags', 'users'] }
  expect(later).toBe('ok');                                  // actual: 'blocked'
});
```

**Expected:** a rejected import leaves the database as it was, and no connection stays open.

**Actual:** the version is bumped to 2 and an empty `tags` store exists. A later upgrade is blocked by a connection the caller has no handle to.

### Logs and Screenshots

```
state after close: { version: 2, stores: [ 'tags', 'users' ] }
later upgrade: blocked
AssertionError: expected { Object (version, stores) } to deeply equal { version: 1, stores: [ 'users' ] }
```

The same steps in headless Chrome 155, using the bundled library on a page served over `http://127.0.0.1`:

```
first import: REJECTED Database "A-…" upgrade blocked. Close all other connections and try again.
after app.close(): version 2 stores ["notes","tags"] tags rows 0
app open(name, 3): BLOCKED
plain open(name) while the other connection is still open: (no event within 1.5 s)
```

In Chrome the stray connection is closed once it is garbage-collected: after a forced `gc()`, `open(name, 3)` succeeded. So how long it blocks depends on GC timing. Under `fake-indexeddb` it never goes away.

### Environment Details

- `main` @ 4b90eca (package 1.1.0)
- Node v24.22.1, Vitest 5.0.1, `fake-indexeddb` 6.2.5, `happy-dom`
- Also reproduced in Google Chrome 155 (headless, Linux). Firefox and Safari not tested.
- It only happens when the merge needs new stores (version bump) and another connection without an `onversionchange` → `close()` handler is open. Connections that close on `versionchange` don't trigger it.

### Impact

Medium - Feature works but has issues

### Proposed fix

In the merge upgrade branch of `openDatabaseForImport` (`src/core/importer.ts:235-252`), use the same cancel-on-blocked pattern that PR #130 adds for the overwrite path:

```ts
let blocked = false;
upgradeRequest.onblocked = () => { blocked = true; reject(new Error(/* same message */)); };
upgradeRequest.onupgradeneeded = () => {
  if (blocked) { upgradeRequest.transaction?.abort(); return; } // cancel the late upgrade
  createStoresFromSchema(upgradeRequest.result, schema, strategy);
};
upgradeRequest.onsuccess = () => {
  if (blocked) { upgradeRequest.result.close(); return; }
  resolve(upgradeRequest.result);
};
upgradeRequest.onerror = (event) => {
  if (blocked) { event.preventDefault(); return; } // expected AbortError
  reject(new Error(/* same message */));
};
```

I tried this locally. After the other connection closes, the database stays at version 1 with only `users`. A later `open(name, 10)` and an overwrite import both succeed, a retried merge imports the records, and the existing importer, exporter and round-trip tests still pass. The `createRequest.onblocked` handler at `:272` could get the same guard for consistency. It is hard to reach because the database didn't exist a moment earlier.

A regression test like the one above can go in `tests/importer.test.ts`. A retry made while the other connection is still open will still wait in the queue until that connection closes, because IndexedDB can't cancel a queued open. A short note in the README about closing connections on `versionchange` would help.

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
