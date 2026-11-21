# [BUG]: `merge` import that creates a missing store bumps the DB version, so the app's own `indexedDB.open(name, version)` fails with `VersionError`

## Bug Description

When `importDB({ strategy: 'merge' })` finds a store in the backup that the target database doesn't have, it reopens the database at `currentVersion + 1` to create that store (`src/core/importer.ts:235`). This is deliberate. The test `merge with new stores triggers version bump` checks it, and the `importDB` JSDoc mentions it in passing under `storeNames`. But the README and the `strategy` docs never say that a merge can change the database version, or what that does to the app that owns the database.

Most apps open their database with a fixed schema version (`indexedDB.open(name, N)`, `openDB(name, N, { upgrade })` from `idb`). After the merge:

1. **The current app build can't open its database.** `open(name, N)` now fails with `VersionError` because the database is at `N + 1`. It keeps failing until site data is cleared or the app ships a higher version.
2. **The app's next schema version can collide with the bumped version.** When the app later ships schema `N + 1`, `onupgradeneeded` doesn't fire because the database is already at `N + 1`. That release's migration never runs, so any stores or indexes it creates are missing.

This happens whenever the backup has a store the live database doesn't have. Two common cases:
- A backup made by a newer build of the app (one that added a store) is restored on a device still running an older build.
- An old backup still contains a store that a later schema version removed.

Passing `storeNames` limited to stores that already exist avoids the bump. That works today, but users have to know about the bump to think of it.

## Steps to Reproduce

Vitest, using the repo's own test setup (`fake-indexeddb` + `happy-dom`). Save as `tests/merge-version.test.ts` and run `npx vitest run tests/merge-version.test.ts`. Both tests fail on `main`.

```ts
import { it, expect } from 'vitest';
import { importDB } from '../src/core/importer.js';
import { setupFakeIDB, uniqueDBName } from './helpers/idb-helpers.js';

setupFakeIDB();

// How an app normally opens its own database: fixed schema version + migration.
function appOpen(name: string, version: number, migrate: (req: IDBOpenDBRequest, oldVersion: number) => void) {
  return new Promise<{ db?: IDBDatabase; error?: string; upgraded: boolean }>((resolve) => {
    let upgraded = false;
    const req = indexedDB.open(name, version);
    req.onupgradeneeded = (e) => { upgraded = true; migrate(req, e.oldVersion); };
    req.onsuccess = () => resolve({ db: req.result, upgraded });
    req.onerror = (e) => { e.preventDefault(); resolve({ error: String(req.error), upgraded }); };
  });
}

// App schema: v1 = "notes"; v2 adds a "tags" store and a "by_title" index on "notes".
const migrate = (req: IDBOpenDBRequest, oldVersion: number) => {
  if (oldVersion < 1) req.result.createObjectStore('notes', { keyPath: 'id' });
  if (oldVersion < 2) {
    req.result.createObjectStore('tags', { keyPath: 'id' });
    req.transaction!.objectStore('notes').createIndex('by_title', 'title');
  }
};
const migrateV1 = (req: IDBOpenDBRequest, oldVersion: number) => {
  if (oldVersion < 1) req.result.createObjectStore('notes', { keyPath: 'id' });
};

// Backup produced by the v2 build of the app.
const backupFromV2 = (dbName: string) => ({
  backupVersion: 1,
  databaseName: dbName,
  databaseVersion: 2,
  exportedAt: '2026-01-01T00:00:00.000Z',
  schema: {
    notes: { keyPath: 'id', autoIncrement: false, indexes: [{ name: 'by_title', keyPath: 'title', unique: false, multiEntry: false }] },
    tags: { keyPath: 'id', autoIncrement: false, indexes: [] },
  },
  stores: { notes: [{ key: 1, value: { id: 1, title: 'hello' } }], tags: [{ key: 't1', value: { id: 't1' } }] },
});

it('merge bumps the version, so the v1 app can no longer open its database', async () => {
  const dbName = uniqueDBName('merge-version');
  (await appOpen(dbName, 1, migrateV1)).db!.close(); // device still runs the v1 build

  await importDB({ dbName, backupData: backupFromV2(dbName), strategy: 'merge' });

  const reopen = await appOpen(dbName, 1, migrateV1);
  expect(reopen.error).toBeUndefined();
});

it("the bumped version collides with the app's next schema version, so its migration never runs", async () => {
  const dbName = uniqueDBName('merge-collision');
  (await appOpen(dbName, 1, migrateV1)).db!.close();

  await importDB({ dbName, backupData: backupFromV2(dbName), strategy: 'merge' }); // DB is now at 1 + 1 = 2

  const v2 = await appOpen(dbName, 2, migrate); // user updates to the v2 build
  expect(v2.upgraded).toBe(true);
  expect(() => v2.db!.transaction('notes').objectStore('notes').index('by_title')).not.toThrow();
  v2.db?.close();
});
```

**Expected:** after a merge, the app can still open its database at the schema version it was built for, and its own migrations still run when it moves to a new schema version. If that can't be guaranteed, the docs should say so and there should be a way to opt out.

**Actual:** see the logs below.

## Logs and Screenshots

```
× merge bumps the version, so the v1 app can no longer open its database
  AssertionError: expected 'VersionError: An attempt was made to …' to be undefined
  Received: "VersionError: An attempt was made to open a database using a lower version than the existing version."

× the bumped version collides with the app's next schema version, so its migration never runs
  AssertionError: expected false to be true
  (and if that assertion is removed, index('by_title') throws
   "NotFoundError: The operation failed because the requested database object could not be found.")
```

Control: the same merge with `storeNames: ['notes']` leaves the database at version 1, and `open(name, 1)` succeeds.

## Environment Details

- `@aossie-org/idb-backup` 1.1.0, `main` @ `4b90eca`
- Node v24.22.1, Vitest 5.0.1, `fake-indexeddb` + `happy-dom` (the repo's test setup)
- Not tied to one engine: the IndexedDB spec says opening with a version lower than the current one fails with `VersionError`, and `upgradeneeded` fires only when the requested version is higher than the current one.

## Impact

Low - Minor inconvenience. The bump is deliberate and tested, and passing `storeNames` avoids it, but an app that hits it cannot open its own database until it ships a higher version.

## Proposed fix

Additive, no new dependencies, default behaviour unchanged:

1. **Document it.** In the README's import section and in the JSDoc for `ImportOptions.strategy`, say that `merge` creates stores missing from the target database by opening it at `currentVersion + 1`. Say what that does to apps that open with a fixed version (`VersionError`, and possible collision with their next schema version). Recommend passing `storeNames` limited to the app's current stores when the app owns the schema.
2. **Add an opt-out**, e.g. `createMissingStores?: boolean` (default `true`). With `false`, `merge` never changes the version: stores the database doesn't have are skipped, and either reported or rejected with a clear error before anything is written (maintainers' choice).
3. Add tests for both modes in `tests/importer.test.ts`.
