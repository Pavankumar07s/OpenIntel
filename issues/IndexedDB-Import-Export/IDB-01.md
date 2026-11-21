# [BUG]: Overwrite import deletes the database before checking the backup's structure, so a malformed backup wipes existing data

### Bug Description

With `strategy: "overwrite"`, `importDB` deletes the existing database (`src/core/importer.ts:166`) before it checks that the backup's `schema` and `stores` are usable. The only check before that is on `databaseVersion` (line 396). If a backup has a valid `databaseVersion` but its structure is wrong, the import fails only after the delete. It rejects with a `TypeError` or an opaque `AbortError`, and the user's data is already gone.

Cases verified on `main` (4b90eca):

| Backup problem | `importDB` result | Existing data afterwards |
| --- | --- | --- |
| `stores` missing | `TypeError: Cannot convert undefined or null to object` | store recreated empty |
| `schema` missing | `Failed to create database "…": AbortError` | database deleted |
| a schema entry has no `indexes` array | `AbortError` | database deleted |
| a store or index `keyPath` is not a valid key path | `AbortError` | database deleted |
| `schema: {}` (or `[]`) while `stores` has records | resolves successfully | all stores gone, records ignored |

`onBeforeImport` does not protect against this. When it is set, a missing `schema` or `stores` happens to throw inside `buildImportSummary()` before the delete. But a schema entry without `indexes`, or with an invalid `keyPath`, still deletes the database after the hook returns `true`. That is because `createStoresFromSchema()` reads those fields inside `onupgradeneeded` (lines 102–134), and that runs after the delete.

To be clear about scope: a random JSON file, or a JSON array, is already rejected safely by the `databaseVersion` check. Only files that look like a backup (they have a positive integer `databaseVersion`) but have a broken structure reach the delete. Examples are a hand-edited backup, a file written by another tool, or a future format version. `backupVersion` is never checked either (`src/core/migrations.ts` is an empty TODO), so the library cannot refuse a format it doesn't understand.

Related: under `"merge"`, a backup without `schema` makes `importDB` never settle. A `TypeError` is thrown inside the probe request's `onsuccess` handler, so the promise is never resolved or rejected. No data is lost in that case.

### Steps to Reproduce

Add to `tests/importer.test.ts`, which already uses the repo's `fake-indexeddb` + `happy-dom` setup:

```ts
it('a schema entry without "indexes" deletes the database, even with onBeforeImport', async () => {
  const dbName = uniqueDBName('malformed');
  (await createTestDB(dbName, 1, [
    { name: 'notes', keyPath: 'id', records: [{ value: { id: 1 } }, { value: { id: 2 } }] },
  ])).close();
  const backup = await exportDB({ dbName });
  const bad = { ...backup, schema: { notes: { keyPath: 'id', autoIncrement: false } } };

  await expect(
    importDB({ dbName, backupData: bad as never, strategy: 'overwrite', onBeforeImport: () => true }),
  ).rejects.toThrow('AbortError');
  const names = (await indexedDB.databases()).map((d) => d.name);
  expect(names).toContain(dbName); // fails on main: the database is gone
});

it('a backup without "schema" deletes the database', async () => {
  const dbName = uniqueDBName('noschema');
  (await createTestDB(dbName, 1, [
    { name: 'notes', keyPath: 'id', records: [{ value: { id: 1 } }, { value: { id: 2 } }] },
  ])).close();
  const { schema: _omit, ...noSchema } = await exportDB({ dbName });

  await expect(importDB({ dbName, backupData: noSchema as never, strategy: 'overwrite' })).rejects.toThrow();
  const names = (await indexedDB.databases()).map((d) => d.name);
  expect(names).toContain(dbName); // fails on main
});
```

**Expected:** a structurally invalid backup is rejected with a clear error *before* anything is deleted or written, and the existing database is left unchanged.

**Actual:** the import rejects only after the existing database has been deleted (or emptied).

### Logs and Screenshots

```
AssertionError: expected [] to include 'malformed-1795112249525-0'
AssertionError: expected [] to include 'noschema-1795112249558-1'
AssertionError: expected [] to have a length of 2 but got +0     (backup without "stores")
```

### Environment Details

- `@aossie-org/idb-backup` 1.1.0, `main` at `4b90eca`
- Node v24.22.1, Vitest 5.0.1, fake-indexeddb 6.2.5, happy-dom 20.14.5 (the repo's own test setup)
- Linux

Overlap: open PR #117 (for #114) deserializes `stores` before the database is opened. That incidentally fixes the "`stores` missing" case. I applied its diff locally and the other cases above still delete the database. #114 is about invalid *records*, not the envelope or schema.

### Proposed fix

Add a small hand-written structural check with no new dependency, for example `assertValidBackup(data: unknown)`. Call it at the top of `importDB`, before `onBeforeImport` and before `openDatabaseForImport`. It should check:

- `backupData` is a plain object, and `backupVersion` is a number no greater than `BACKUP_VERSION`
- `schema` and `stores` are plain objects
- every schema entry has `keyPath` (string, string array, or null), a boolean `autoIncrement`, and an `indexes` array of `{ name: string, keyPath: string | string[], unique: boolean, multiEntry: boolean }`
- every `stores` value is an array of `{ key, value }`
- `strategy` is `"overwrite"` or `"merge"` (for plain-JS callers)

The check should throw one descriptive error that names the bad path, e.g. `Invalid backup: schema.notes.indexes must be an array`. Add a test for each case in the table above. Checking that key paths are syntactically valid could also be done up front. At minimum, the existing data must stay intact when the check fails.

### Impact

Medium - Feature works but has issues
