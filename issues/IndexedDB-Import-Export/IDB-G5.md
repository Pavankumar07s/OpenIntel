# [FEATURE]: Add a conflict policy to `merge` imports (overwrite / skip existing / custom resolver)

## Feature and its Use Cases

**What is the feature?**

An optional `onConflict` setting for `importDB({ strategy: 'merge' })`. It would control what happens when a backup record has the same primary key as a record already in the database.

Today `merge` always writes with `store.put()` (`src/core/importer.ts:320-321`), so the backup copy always replaces the live record. This upsert is intended: it is documented ("add/update records from the backup") and covered by `tests/importer.test.ts:294`. But there is no way to keep the existing record instead.

**Scenario it addresses**

A user wants to bring back records they deleted after an old backup was taken. `merge` is the only strategy that keeps the current data, so they use it. The deleted records come back, but every record edited since the backup is also rolled back to the old version. Nothing reports that existing records were replaced.

Minimal repro on `main` (4b90eca), using the repo's own test setup (vitest + fake-indexeddb):

```ts
// DB 'notes-app', store 'notes' (keyPath 'id') holds:
//   { id: 1, text: 'groceries v1', updatedAt: 1 }
//   { id: 2, text: 'draft',        updatedAt: 1 }
const lastWeek = JSON.parse(JSON.stringify(await exportDB({ dbName: 'notes-app' })));

// Later the user edits note 1, adds note 3 and deletes note 2:
//   put    { id: 1, text: 'groceries v2 (edited today)', updatedAt: 2 }
//   put    { id: 3, text: 'new note', updatedAt: 2 }
//   delete 2

// To get note 2 back:
await importDB({ dbName: 'notes-app', backupData: lastWeek, strategy: 'merge' });
```

`getAll()` before the import:

```
[{"id":1,"text":"groceries v2 (edited today)","updatedAt":2},{"id":3,"text":"new note","updatedAt":2}]
```

`getAll()` after the import:

```
[{"id":1,"text":"groceries v1","updatedAt":1},{"id":2,"text":"draft","updatedAt":1},{"id":3,"text":"new note","updatedAt":2}]
```

Note 2 is restored, but note 1 has lost today's edit. Out-of-line-key stores behave the same way: a live value `'v2-local'` under key `'k'` becomes the backup's `'v1'`.

The docs also differ on what to expect. The feature list (`README.md:61`) and the JSDoc `@example` (`src/core/importer.ts:368`) call `merge` an "additive sync", which reads as "only adds". The usage section (`README.md:234`) and `ImportOptions.strategy` say "add/update".

**How users would benefit**

Users could restore deleted records, or combine data from another device, without rolling back newer local edits.

**Proposed API**

The default stays as it is today, so this is not a breaking change.

```ts
onConflict?:
  | 'overwrite'   // default: the current behaviour (put)
  | 'skip'        // keep the existing record; only insert keys that are absent
  | ((existing: unknown, incoming: unknown, ctx: { storeName: string; key: IDBValidKey }) =>
      unknown | undefined); // return the value to write, or undefined to keep `existing`
```

"Newer wins" would be a resolver, for example `(a, b) => (b.updatedAt > a.updatedAt ? b : undefined)`. It can't be a built-in mode because the library doesn't know which field holds the timestamp.

**Implementation notes**

- Every backup record already stores its primary key (`src/core/exporter.ts:50`, `cursor.primaryKey`). For both inline- and out-of-line-key stores, the importer can call `store.get(key)` and then `put` or skip, all inside the existing readwrite transaction.
- Do **not** implement `skip` as "`add()` and `preventDefault()` on `ConstraintError`". A unique-index violation raises the same `ConstraintError`. I tested this with fake-indexeddb: a record with a new `id` but a duplicate value in a unique `email` index was silently dropped, along with the real primary-key conflict. Check whether the primary key exists instead.
- The resolver has to be synchronous. If it awaits a non-IndexedDB promise, the transaction auto-commits and the next request throws `TransactionInactiveError`. Document this, or reject a resolver that returns a Promise.
- Change the "additive sync" wording in `README.md:61` and the `importer.ts` JSDoc example to match the actual add/update behaviour, and document `onConflict`.
- Add tests for each mode, covering inline and out-of-line keys and a unique-index conflict under `skip`.

## Additional Context

- **Environment:** `main` at 4b90eca, Node 24.22.1, vitest 5.0.1, fake-indexeddb 6.2.5 (the repo's test setup). The result follows from the IndexedDB `put()` semantics, so browsers behave the same way.
- **Current workaround:** before calling `importDB`, open the database yourself, read the existing keys and filter `backupData.stores[storeName]` down to the missing keys.
- **Prior art:** dexie-export-import does not overwrite existing rows by default. With `overwriteValues: false` it uses `bulkAdd`, which fails the import on an existing key rather than skipping it. `overwriteValues: true` uses `bulkPut`, and `clearTablesBeforeImport` empties the tables first. A `skip` mode would go one step further than that default.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
