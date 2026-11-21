# [BUG]: `merge` ignores schema differences on stores that already exist (missing indexes skipped, keyPath mismatch fails with an unwrapped `DataError`)

## Bug Description

With `strategy: 'merge'`, `importDB` never compares the backup's schema with a store that already exists in the target database. `createStoresFromSchema` (`src/core/importer.ts:114-119`) hits `continue` for every existing store, so that store's indexes are skipped as well. Neither the README nor the `importDB` JSDoc mentions this. They only say that merge "creates any missing stores" and upserts records.

This causes two visible problems:

1. **Indexes in the backup are not created on an existing store.** The import resolves, and the store is left without them. This also happens when merge *does* run a version change because the backup has a new store. In that case the new store gets its indexes and the existing store still does not. The code comment says this is necessary ("We cannot access the store for index creation outside a versionchange transaction…"). Inside `onupgradeneeded`, though, an existing store can be reached with `request.transaction.objectStore(name)` and given new indexes, so the comment is wrong for that path.
2. **A keyPath mismatch fails late, with a raw `DOMException`.** Example: the existing store has inline key `uid` and the backup's store uses out-of-line keys. Then `store.put(value)` in `insertRecords` throws a synchronous `DataError`. `insertRecords` only wraps *async* request errors (`Failed to insert record into store "…"`), so the caller gets a bare `DataError` that does not name the store. The thrown error also doesn't abort the readwrite transaction. Records that were already queued for other stores are committed, and the merge is left half-applied. The half-applied part has the same root cause as #114. The fix in PR #117 only moves `deserialize()` up front, so it does not cover errors thrown by `put()`.

This happens when a user merges a backup from a different schema version of the app, for example one with an extra index or a store whose keyPath changed.

## Steps to Reproduce

Vitest, using the repo's own helpers (can go into `tests/importer.test.ts`):

```ts
const env = { backupVersion: 1, exportedAt: '2026-01-01T00:00:00.000Z' } as const;

it('merge does not create a backup index on an existing store', async () => {
  const dbName = uniqueDBName('merge-index');
  (await createTestDB(dbName, 1, [{ name: 'users', keyPath: 'id' }])).close();

  await importDB({ dbName, strategy: 'merge', backupData: { ...env, databaseName: dbName, databaseVersion: 2,
    schema: { users: { keyPath: 'id', autoIncrement: false,
      indexes: [{ name: 'by_email', keyPath: 'email', unique: false, multiEntry: false }] } },
    stores: { users: [{ key: 1, value: { id: 1, email: 'a@x' } }] } } });

  const db = await new Promise<IDBDatabase>((res) => { const r = indexedDB.open(dbName); r.onsuccess = () => res(r.result); });
  console.log(db.version, Array.from(db.transaction('users').objectStore('users').indexNames)); // 1 []
  db.close();
});

it('keyPath mismatch: raw DataError, other store still written', async () => {
  const dbName = uniqueDBName('merge-keypath');
  (await createTestDB(dbName, 1, [{ name: 'notes', keyPath: 'id' }, { name: 'users', keyPath: 'uid' }])).close();

  const err = await importDB({ dbName, strategy: 'merge', backupData: { ...env, databaseName: dbName, databaseVersion: 1,
    schema: { notes: { keyPath: 'id', autoIncrement: false, indexes: [] },
              users: { keyPath: null, autoIncrement: false, indexes: [] } },
    stores: { notes: [{ key: 1, value: { id: 1 } }], users: [{ key: 'k1', value: { name: 'x' } }] } } })
    .catch((e) => e);
  console.log(String(err));                                  // DataError: ... (no store name)
  console.log(await readAllFromStore(dbName, 'notes'));     // [{ key: 1, value: { id: 1 } }]: committed despite the rejection
});
```

Also observed: if the backup adds a new store (`drafts`) next to the existing `users`, merge upgrades the database to version 2. `drafts` gets its index and `users` still has none.

## Logs and Screenshots

Vitest + fake-indexeddb:

```
index case:     version=1 indexNames=[]
new-store case: users {"names":[],"v":2}   drafts {"names":["by_title"],"v":2}
keyPath case:   DataError: Data provided to an operation does not meet requirements.
                notes after rejected merge: [{ key: 1, value: { id: 1 } }]
```

Real browser (built `dist/index.js`, headless Chrome 155):

```
A version=1 indexNames=[]
C err ctor=DOMException name=DataError msg=Failed to execute 'put' on 'IDBObjectStore': Evaluating the object store's key path did not yield a value.
C notes count after rejected merge=1
```

## Environment Details

- `@aossie-org/idb-backup` 1.1.0, `main` at `4b90eca`
- Node 24.22.1, Vitest 5.0.1, fake-indexeddb 6.2.5, happy-dom 20.14.5
- Google Chrome 155.0.8059.79 (headless) on Linux, loading the built ESM bundle

## Impact

Low - Minor inconvenience

## Proposed fix

- During the merge probe (which already opens the database to read `objectStoreNames`), read `keyPath`, `autoIncrement` and `indexNames` of each existing store that is being imported, and compare them with `backupData.schema`.
- If the keyPath differs, reject before writing anything with a descriptive error, for example `Store "users" keyPath mismatch: database has "uid", backup has null`.
- For missing indexes, let maintainers choose:
  - (a) When merge already runs a version change for new stores, create the missing indexes on existing stores via `upgradeRequest.transaction.objectStore(name)`, and correct the misleading comment.
  - (b) Document that merge never changes an existing store's keyPath or indexes.
  - Adding a version bump only to add indexes is not suggested. The library would then change a schema the app owns: the app's next `open(name, appVersion)` would fail, and its own later `createIndex` call with the same name would throw.
- In `insertRecords`, catch synchronous `put()`/`add()` throws, wrap them with the store name the same way async errors are wrapped, and abort the transaction so a failed merge is not half-applied (see #114).
- Add regression tests for all three cases.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
