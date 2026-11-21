# [FEATURE]: Add an atomic restore that clears and refills stores in one transaction instead of deleting the database

### Feature and its Use Cases

**Problem.** The only way to make a database match a backup today is `strategy: "overwrite"`, and it works by calling `indexedDB.deleteDatabase()` and then recreating the database at the backup's version (`src/core/importer.ts:164-192`). Deleting a database is not part of any transaction, so:

- **A write error after the delete loses the data.** If any record fails to write (a duplicate key, a unique-index violation), the import rejects but the old data has already been deleted. PR #117 handles records that fail to *deserialize*. It says itself that write-time errors are out of scope.
- **Any other open connection blocks it.** Another tab, or the app's own connection, makes the delete fire `blocked`. That is the cause of #126, which PR #130 is fixing.
- **The database is recreated at the backup's version,** which may not be the version the running app expects. That happens even when the only goal was to restore data.

Checked on `main` (4b90eca) with the repo's `fake-indexeddb` setup:

```
overwrite, backup has two records with key 3 -> rejects with ConstraintError
store contents afterwards                     -> []          (both original records gone)
overwrite while another connection is open    -> "deletion blocked" error
```

For comparison, here is the same thing done inside one `readwrite` transaction (`store.clear()` and then `add()` for each record), with a second connection still open:

```
transaction result     -> aborted: ConstraintError   (not blocked by the other connection)
store contents after   -> [{"id":1,"v":"old"},{"id":2,"v":"old"}]   (unchanged)
```

IndexedDB rolls back a `readwrite` transaction as a whole, so the restore either applies completely or leaves the stores as they were.

**Proposal.** Add a restore mode that, for stores that already exist in the target database:

1. Validates the backup and deserializes every record first, as PR #117 does.
2. Opens the database at its **current** version.
3. In a **single `readwrite` transaction** over the selected stores, calls `store.clear()` on each one and then `add()`s every record from the backup.
4. Calls `transaction.abort()` on the first failed request, so nothing is committed.

It could be a new value, `strategy: 'replace'`. Or `"overwrite"` could take this path when the target database already has every store in the backup with a matching `keyPath`/`autoIncrement`, and fall back to delete-and-recreate only when the schema really has to change. Maintainers, which do you prefer?

**Limits (worth stating in the docs):**
- If the backup contains stores the database doesn't have, or a different `keyPath` or index set, the schema can only change in a `versionchange` upgrade. That upgrade is blocked by other connections and bumps the version. Such cases should either fall back to the current behaviour or be rejected with a clear error. They should not be applied partly.
- Stores that are not in the backup (or not in `storeNames`) are left untouched. This is a change from `"overwrite"`, which removes them.

**Who benefits:** apps that restore their own backup into their own database ("put my data back"). That is the common case, and it is where data loss hurts most.

### Additional Context

- Related: #114 / PR #117 (deserialize-time errors), #126 / PR #130 (blocked delete). This proposal covers the write-time failures that neither of them can fix while the restore still deletes the database first.
- Tests can use the existing `fake-indexeddb` helpers: a duplicate key in the backup leaves the original rows intact; the restore succeeds while another connection is open; the database version is unchanged afterwards.

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
