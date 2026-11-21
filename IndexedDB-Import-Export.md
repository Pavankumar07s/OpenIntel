# IndexedDB-Import-Export (`@aossie-org/idb-backup`): contribution audit

- **Repo:** AOSSIE-Org/IndexedDB-Import-Export, `main` @ `4b90eca` (last commit 2026-11-14), package version 1.1.0
- **Audited:** 2026-11-19, Node v24.22.1 / npm 11.19.1
- **Method:** I read every file in `src/` and `tests/` plus the docs, CI workflows and audit reports. I ran the full toolchain and wrote Vitest repros, using the repo's own `fake-indexeddb` + `happy-dom` setup and importing straight from `src/`, for every suspected bug. I deduplicated each finding against all 17 issues and 116 PRs on GitHub (read-only).
- **Repo state:** the working tree is clean. Nothing was committed or pushed. The only things added to the ignored `node_modules/` were dev tools installed with `--no-save`.

---

## 1. Summary

### Toolchain results

| Check | Result |
| --- | --- |
| `npm ci` | OK (180 packages). esbuild postinstall is not on npm's allowScripts list, but the build still works |
| `npm run build` (tsup) | OK. ESM 18.65 KB, CJS 19.92 KB, `index.d.ts` + `index.d.cts` |
| `npm test` (Vitest 5.0.1) | **98/98 passed**, 6 files, about 2 s |
| `npm run lint` (ESLint 10) | OK, 0 problems |
| `npm run format:check` | **FAILS**: `src/utils/ssr.ts` is missing a trailing comma (line 22) |
| `npx tsc --noEmit` | OK. Note that `tsconfig.json` excludes `tests/`, so test files are never type-checked. They do pass when checked by hand |
| `npm audit` | 16 vulns (1 low, 11 moderate, 4 high). **All dev-only** (`npm audit --omit=dev` → 0). Sources are happy-dom, vite/vitest, minimatch (eslint / typescript-eslint), source-map-js, brace-expansion and esbuild. All are fixable, and most are already covered by open Dependabot PRs #118–#122 |
| Coverage (`@vitest/coverage-v8@5.0.1`, `--no-save`) | **Statements 91.66 % (286/312), Branches 90.62 % (116/128), Functions 77.92 % (60/77), Lines 91.66 % (275/300).** `exporter.ts` 83.67 % stmts / **60 % branches**; `importer.ts` 88.48 % / 84.09 %; `serialization/index.ts` 98.97 %; `utils/file.ts` 95.45 % |
| `npx knip` | Unused dependency **`idb`** (never imported in `src/`). Unused files: `src/core/migrations.ts` (empty TODO), `dangerfile.js` (false positive, used by CI) |
| `npx publint` | Warning: `exports["."].types` is read as ESM for `require`. Suggests `sideEffects: false` |
| `npx @arethetypeswrong/cli --pack` | **node16 (from CJS): "Masquerading as ESM"** (FalseESM). node10, node16-ESM and bundler are green |
| `npx madge --circular` | No circular dependencies |
| ESM / CJS runtime parity | Both builds export the same 9 symbols |
| CI on PRs | **None.** No workflow runs build, test, lint, format or typecheck on pull requests. Tests run only in `version-release.yml`, *after* the tag and GitHub release are created. Checks on PR #117 were only label-conflicts, danger, sync-labels, CodeRabbit and Socket |

### Findings overview

18 findings: 9 bugs, 3 features, 1 performance improvement, then CI, packaging, tests, docs and a cleanup item. **16 are VERIFIED with a repro.** IDB-10 and IDB-12 are feature requests (nothing to reproduce). IDB-18 is a cleanup grab-bag that mixes tool-confirmed items with one suspected nit.

The 6 open issues (#95, #114, #124, #126, #131, #132) **all still reproduce on `main`**; I re-ran each one. They are **not** repeated as findings. See §4.

---

## 2. Findings table (ranked by value)

| ID | Title (ready to file) | Type | Sev | Status | GitHub dedup | Effort |
| --- | --- | --- | --- | --- | --- | --- |
| IDB-01 | `importDB` deletes the existing database before checking that the backup is well-formed (missing/invalid `schema`, `stores`, `indexes`, unknown `backupVersion`/`strategy`) | bug | **high** | VERIFIED | NEW. PR #117 incidentally covers only the "missing `stores`" case | M |
| IDB-02 | A blocked `merge` upgrade rejects, but the upgrade still runs later and leaks a connection that blocks every future version change | bug | **high** | VERIFIED | PARTIALLY COVERED: #126 / PR #130 fix only the overwrite *delete* path | M |
| IDB-03 | Overwrite import wipes existing data when a record fails to *write* (duplicate key, unique-index violation) | bug | **high** | VERIFIED | PARTIALLY COVERED: described in #114, but PR #117 explicitly leaves it out ("happy to open a follow-up issue") | M |
| IDB-04 | `exportDB` never settles and leaks its connection when a record can't be serialized (e.g. an Invalid Date, which IndexedDB stores fine) | bug | **high** | VERIFIED | PARTIALLY COVERED: draft PR #128 fixes the hang (no issue filed). Encoding Invalid Date is NEW | S |
| IDB-05 | `merge` that adds a store bumps the DB version, so the app's next `indexedDB.open(name, appVersion)` fails with `VersionError` | bug | medium | VERIFIED | NEW | S–M |
| IDB-06 | `merge` silently ignores schema differences on stores that already exist (missing indexes not created, keyPath mismatch → opaque `DataError`) | bug | medium | VERIFIED | NEW | M |
| IDB-07 | `Error`, boxed primitives, `Float16Array` and BigInt wrapper objects are silently corrupted on export (or break `JSON.stringify` after a "successful" export) | bug | medium | VERIFIED | PARTIALLY COVERED: #131 covers only Blob/File | M |
| IDB-08 | `deserialize()` and `ImportSummary.recordCounts` return null-prototype objects (`hasOwnProperty` missing, `String(x)` throws) | bug | low–med | VERIFIED | NEW | S |
| IDB-09 | An object store named `__proto__` is silently dropped by `exportDB` | bug | low | VERIFIED | NEW | S |
| IDB-10 | Add an atomic, non-destructive "replace" strategy (clear and refill selected stores in one readwrite transaction) | feature | medium | n/a (feature) | NEW | M |
| IDB-11 | Make `importDB` return an `ImportResult` so callers can tell an aborted import from a completed one | feature | low–med | VERIFIED (behaviour) | NEW | S |
| IDB-12 | Add `onProgress` and batched reads/writes for large databases | feature | low–med | n/a (feature) | NEW | M–L |
| IDB-13 | Speed up base64 encoding of binary values (about 11× faster on 16 MiB) | improvement | low | VERIFIED (benchmark) | NEW | S |
| IDB-14 | Add a PR CI workflow (build, test, lint, format, typecheck, coverage) and run tests *before* tagging a release | ci | medium | VERIFIED | NEW (BestPracticesChecklist lists it as a gap; no issue or PR) | S |
| IDB-15 | CJS consumers get ESM type declarations ("Masquerading as ESM"). Split `types` per condition and add `sideEffects: false` | bug (packaging) | low–med | VERIFIED | NEW | S |
| IDB-16 | Close the test gaps: `exportDB` on a missing DB, blocked/error branches, prototype-sensitive assertions, type-check tests | test | low | VERIFIED (coverage) | NEW | S–M |
| IDB-17 | README/docs claims don't match behaviour (SSR "no-op", `idb` wrapper, stale "under development" note, broken link, no limitations section) and `exportDB`/`importDB` throw a raw `ReferenceError` outside the browser | docs | low | VERIFIED | NEW (the `idb` part was raised in #99/#101, both closed) | S |
| IDB-18 | Cleanup: Prettier failure on `main`, unused `idb` dependency, empty `migrations.ts`, mojibake in CONTRIBUTING, immediate `revokeObjectURL` | cleanup | low | mixed | `idb`: #99/#101 closed by maintainers. Rest NEW | S |

---

## 3. Findings in detail

### Repro harness

All repros below were run like this:

```bash
S=/tmp/.../scratchpad/idb            # any empty dir
ln -s <repo>/node_modules $S/node_modules
echo '{"type":"module","private":true}' > $S/package.json
# vitest.config.ts: environment 'happy-dom', include ['*.test.ts']
$S/node_modules/.bin/vitest run --root $S --reporter=verbose
```

`h.ts` re-exports `exportDB`, `importDB`, `serialize` and `deserialize` from `<repo>/src/...`, and `setupFakeIDB`, `uniqueDBName`, `createTestDB` and `readAllFromStore` from `<repo>/tests/helpers/idb-helpers.ts`. It also adds three small helpers:

```ts
export const settleWithin = <T>(p: Promise<T>, ms: number) =>
  Promise.race([p, new Promise<'TIMEOUT'>((r) => setTimeout(() => r('TIMEOUT'), ms))]);
export const listDatabases = async () => (await indexedDB.databases()).map((d) => d.name!);
export const openDB = (name: string, v?: number) => new Promise<IDBDatabase>((res, rej) => {
  const r = v === undefined ? indexedDB.open(name) : indexedDB.open(name, v);
  r.onsuccess = () => res(r.result); r.onerror = () => rej(r.error);
});
```

When a PR is opened, each repro can go almost unchanged into `tests/importer.test.ts`, `tests/exporter.test.ts` or `tests/serialization.test.ts`.

---

### IDB-01: `importDB` deletes the existing database before checking that the backup is well-formed

- **Type:** bug · **Severity:** high (silent, unrecoverable data loss from a bad file) · **Status:** VERIFIED · **Effort:** M
- **Evidence**
  - `src/core/importer.ts:396`: the only validation is `Number.isSafeInteger(backupData.databaseVersion)`.
  - `src/core/importer.ts:166`: under `overwrite`, `deleteDatabase()` runs before anything else touches `schema` or `stores`.
  - `src/core/importer.ts:97-136`: `createStoresFromSchema` iterates `schema` and `storeSchema.indexes` *inside* `onupgradeneeded`, which runs after the delete. A `TypeError` there aborts the upgrade and is reported only as an opaque `AbortError`.
  - `src/core/importer.ts:428`: records for a store missing from `schema` are dropped silently.
  - `backupVersion` is never checked, and `src/core/migrations.ts` is an empty TODO. `strategy` is never checked at runtime: anything other than `'overwrite'` takes the merge path but inserts with `add()` (line 320).
  - `readFileAsJSON()` returns `unknown`, and callers cast it to `ExportFormat`. So any JSON file a user picks reaches this code unchecked.
- **Repro** (all verified on `main`):

  ```ts
  setupFakeIDB();
  async function seed(dbName: string) {
    (await createTestDB(dbName, 1, [{ name: 'items', keyPath: 'id',
      records: [{ value: { id: 1 } }, { value: { id: 2 } }] }])).close();
  }
  const env = { backupVersion: 1, databaseName: 'x', databaseVersion: 1, exportedAt: '2026-01-01T00:00:00.000Z' };

  it('missing "stores" wipes data', async () => {
    const dbName = uniqueDBName('no-stores'); await seed(dbName);
    const bad = { ...env, schema: { items: { keyPath: 'id', autoIncrement: false, indexes: [] } } } as any;
    await importDB({ dbName, backupData: bad, strategy: 'overwrite' }).catch(() => {});
    expect(await readAllFromStore(dbName, 'items')).toHaveLength(2); // ACTUAL: 0
  });
  it('missing "schema" deletes the DB', async () => {
    const dbName = uniqueDBName('no-schema'); await seed(dbName);
    await importDB({ dbName, backupData: { ...env, stores: { items: [] } } as any, strategy: 'overwrite' }).catch(() => {});
    expect(await listDatabases()).toContain(dbName); // ACTUAL: database gone
  });
  it('schema entry without "indexes" deletes the DB', async () => {
    const dbName = uniqueDBName('noidx'); await seed(dbName);
    await importDB({ dbName, strategy: 'overwrite', backupData: { ...env,
      schema: { items: { keyPath: 'id', autoIncrement: false } as any }, stores: { items: [] } } }).catch(() => {});
    expect(await listDatabases()).toContain(dbName); // ACTUAL: gone; error is "AbortError: A request was aborted"
  });
  ```

  Observed output:

  ```
  [IDB-C1] importDB error: TypeError: Cannot convert undefined or null to object   | rows left in items: 0
  [IDB-C2] importDB error: Failed to create database "...": AbortError ...          | database still listed: false
  [IDB-C3] (backupData = JSON array `[]`)  AbortError ...                           | database still listed: false
  [IDB-C4] backupVersion 999 accepted silently; strategy "replace" -> ConstraintError from add()
  [IDB-C5] schema: {}, stores: { orphan: [2 records] } -> resolves, 0 stores created, no warning
  [IDB-C6] schema entry without "indexes" -> AbortError                             | db still exists: false
  ```
- **Proposed fix**
  1. Add `validateBackup(data: unknown): asserts data is ExportFormat` in `src/core/migrations.ts`, or a new `validate.ts`, and export it as public API.
  2. It should check that the envelope is a plain object, that `backupVersion` is a known version (≤ `BACKUP_VERSION`, with a hook for migrations), that `schema` and `stores` are objects, and that every schema entry has a valid `keyPath` type, a boolean `autoIncrement` and an `indexes` array of `{name, keyPath, unique, multiEntry}`.
  3. Every `stores` key must have a matching `schema` entry or an existing store, and each record list must be an array of `{key, value}`.
  4. It should validate `strategy ∈ {'overwrite','merge'}` and `storeNames` (array of strings).
  5. Call it at the top of `importDB`, before `onBeforeImport` and before any destructive step.
  6. Throw one descriptive error type (e.g. `BackupValidationError`) with a JSON path such as `schema.items.indexes`.
  7. Add tests for each case above.

  This pairs naturally with PR #117 (pre-deserialize records) but is a separate concern. #117 doesn't touch the schema, version or strategy.
- **Dedup:** NEW. #114 is about invalid *records*. PR #117 pre-deserializes `stores`, which would incidentally make a missing `stores` throw early, but not a missing or invalid `schema`, `indexes`, version or strategy.

---

### IDB-02: A blocked `merge` upgrade rejects, but the upgrade still runs later and leaks a connection that blocks every future version change

- **Type:** bug · **Severity:** high · **Status:** VERIFIED · **Effort:** M
- **Evidence**
  - `src/core/importer.ts:247`: `upgradeRequest.onblocked = () => reject(...)`. A blocked `open()` is **not cancelled** by rejecting. Once the other connection closes, the upgrade runs, `onupgradeneeded` creates the stores, and `onsuccess` resolves an already-rejected promise. The resulting `IDBDatabase` is never closed.
  - The same pattern appears at `:185` (overwrite re-create) and `:272` (merge create).
  - None of the library's connections set `db.onversionchange = () => db.close()`. A long export or import therefore also blocks the app's own upgrades.
- **Repro:**

  ```ts
  it('blocked merge upgrade', async () => {
    const dbName = uniqueDBName('blocked-merge');
    const appConn = await createTestDB(dbName, 1, [{ name: 'items', keyPath: 'id', records: [{ value: { id: 1 } }] }]);
    const backup = { backupVersion: 1, databaseName: dbName, databaseVersion: 1, exportedAt: '2026-01-01T00:00:00.000Z',
      schema: { items: { keyPath: 'id', autoIncrement: false, indexes: [] }, extra: { keyPath: 'id', autoIncrement: false, indexes: [] } },
      stores: { items: [], extra: [{ key: 1, value: { id: 1 } }] } };
    await expect(importDB({ dbName, backupData: backup, strategy: 'merge' })).rejects.toThrow(/blocked/);
    appConn.close();                                  // user follows the error message
    await new Promise((r) => setTimeout(r, 100));
    const db = await openDB(dbName);
    console.log(db.version, Array.from(db.objectStoreNames)); db.close();   // 2 ['extra','items']  <- applied anyway
    const outcome = await settleWithin(new Promise<string>((res) => {
      const req = indexedDB.open(dbName, 10);
      req.onblocked = () => res('BLOCKED'); req.onsuccess = () => { req.result.close(); res('ok'); };
    }), 1000);
    expect(outcome).toBe('ok');                       // ACTUAL: 'BLOCKED' (leaked connection)
  });
  ```

  Output:

  ```
  [IDB-E] importDB rejected with: Database "..." upgrade blocked. Close all other connections and try again.
  [IDB-E] after close: version 2 stores [ 'extra', 'items' ]
  [IDB-E] later upgrade by the app: BLOCKED by a connection nobody can close
  ```

  The leak was bad enough that the test file's `afterEach` `deleteDatabase` cleanup timed out for every later test.
- **Proposed fix**
  - In each `onblocked` handler, set a `cancelled` flag and reject. In `onupgradeneeded`, if `cancelled`, call `request.transaction.abort()`. In `onsuccess`, if `cancelled`, call `request.result.close()`.
  - Alternatively, keep waiting on `blocked` and make it observable through an `onBlocked` callback option.
  - Also set `db.onversionchange = () => db.close()` on every connection the library opens.
  - Factor all six open/delete call sites into one helper so the fix lives in one place. PR #130 already adds a similar `ensureNoOtherConnections`.
- **Dedup:** PARTIALLY COVERED. #126 / PR #130 (changes requested) handle only the overwrite `deleteDatabase` path. The merge upgrade, merge create and `versionchange` handling are untouched. Best filed as a new issue that cross-references #126, or raised on PR #130.

---

### IDB-03: Overwrite import wipes existing data when a record fails to *write*

- **Type:** bug · **Severity:** high · **Status:** VERIFIED · **Effort:** M
- **Evidence:** `src/core/importer.ts:166` deletes the DB, then `:297-340` inserts in a separate readwrite transaction. A `ConstraintError` (duplicate primary key, `unique` index violation) or a `DataError` (invalid key) aborts that transaction, but the original database is already gone. PR #117 moves `deserialize` before the delete, which covers malformed tags but **not** write-time errors. Its description says: *"A write error in overwrite mode (for example a constraint failure on add()) can still leave stores partly cleared … happy to open a follow-up issue."* No follow-up issue exists.
- **Repro:**

  ```ts
  it('duplicate key wipes data', async () => {
    const dbName = uniqueDBName('dup'); await seed(dbName);          // items: id 1, 2
    const backup = { ...env, schema: { items: { keyPath: 'id', autoIncrement: false, indexes: [] } },
      stores: { items: [{ key: 9, value: { id: 9 } }, { key: 9, value: { id: 9 } }] } };
    await importDB({ dbName, backupData: backup, strategy: 'overwrite' }).catch(() => {});
    expect((await readAllFromStore(dbName, 'items')).map((r) => r.key)).toEqual([1, 2]); // ACTUAL: []
  });
  // Same with two records sharing a value on a { unique: true } index -> rows left: 0
  ```

  Output: `ConstraintError ... rows left: 0` for both the duplicate-key and the unique-index cases. I also confirmed the #114 merge case: keys after a failed merge are `[1, 2, 11]`, a half-applied import.
- **Proposed fix:**
  - **(a) Cheap pre-flight**, before deleting: per store, check that keys are valid IDB keys and unique, and that values are unique on each `unique` index. For inline keys and indexes this means evaluating the key path on the deserialized value. That catches about 99 % of real failures.
  - **(b) Structural fix:** when `backupVersion > currentVersion`, do everything inside a single `versionchange` transaction (delete stores, recreate, insert). If anything throws, abort, and IndexedDB rolls back to the old version and contents.
  - **(c)** IDB-10's "replace" strategy avoids the delete entirely.
  - Whatever the approach, call `transaction.abort()` on any error so merge never commits half a batch.
- **Dedup:** PARTIALLY COVERED by #114 (the issue text mentions duplicates). The fix in PR #117 doesn't cover it. File as the follow-up that PR #117 offered, referencing #114 and #117.

---

### IDB-04: `exportDB` never settles and leaks its connection when a record can't be serialized

- **Type:** bug · **Severity:** high (hang plus connection leak). Invalid Date support itself is medium · **Status:** VERIFIED · **Effort:** S
- **Evidence:**
  - `src/core/exporter.ts:46-51`: `serialize()` runs inside `request.onsuccess`. A throw there escapes the event handler, not the Promise executor, so `readAllRecords` never resolves or rejects. `exportDB` awaits forever and `finally { db.close() }` never runs.
  - The trigger is realistic. `serialize(new Date(NaN))` calls `toISOString()`, which throws `RangeError` (`src/serialization/index.ts:186`). IndexedDB happily stores Invalid Date values, because structured clone supports them.
- **Repro:**

  ```ts
  it('Invalid Date hangs exportDB', async () => {
    const dbName = uniqueDBName('invalid-date');
    (await createTestDB(dbName, 1, [{ name: 'items', keyPath: 'id', records: [{ value: { id: 1, at: new Date(NaN) } }] }])).close();
    const r = await settleWithin(exportDB({ dbName }).then(() => 'resolved', (e) => `rejected: ${e}`), 1500);
    expect(r).not.toBe('TIMEOUT');   // ACTUAL: 'TIMEOUT'
  });
  ```

  Output: `[IDB-B] exportDB outcome after 1.5s: TIMEOUT`. Separately, IndexedDB stored `{ id: 1, d: Invalid Date }` without complaint.
- **Proposed fix:**
  1. Wrap the cursor callback body in `try/catch → reject(...)`, and ideally abort the transaction. That is what draft PR #128 does.
  2. Include the store name and primary key in the error.
  3. Make Invalid Date exportable: encode it as `{ __type: 'date', value: null }` (or `"Invalid Date"`) and decode it back to `new Date(NaN)`, instead of making the whole database unexportable. Today the test `serialization.test.ts:47` asserts the throw, so it would need updating.
- **Dedup:** PARTIALLY COVERED. Draft PR #128 (by aakashkumarrr08-art) fixes the hang but says "no tracking issue exists". Invalid Date encoding is NEW. The team could file one issue, "exportDB hangs on unserializable records + support Invalid Date", and reference #128 for the first half.

---

### IDB-05: `merge` that adds a store bumps the DB version, so the app's next `open(name, appVersion)` fails with `VersionError`

- **Type:** bug · **Severity:** medium · **Status:** VERIFIED · **Effort:** S–M
- **Evidence:** `src/core/importer.ts:235`: `indexedDB.open(dbName, currentVersion + 1)` whenever the backup has a store the DB lacks. Apps open with a fixed schema version, so after the merge their next `indexedDB.open('db', 3)` hits `VersionError` and the app can't open its own database until the user clears site data. Neither the README nor the JSDoc mention the version bump. Typical trigger: restoring a backup taken with a newer or older app build that had an extra store.
- **Repro:**

  ```ts
  it('merge bumps version and breaks app open', async () => {
    const dbName = uniqueDBName('appver');
    (await createTestDB(dbName, 3, [{ name: 'notes', keyPath: 'id' }])).close();
    await importDB({ dbName, strategy: 'merge', backupData: { backupVersion: 1, databaseName: dbName, databaseVersion: 3,
      exportedAt: '2026-01-01T00:00:00.000Z',
      schema: { notes: { keyPath: 'id', autoIncrement: false, indexes: [] }, drafts: { keyPath: 'id', autoIncrement: false, indexes: [] } },
      stores: { notes: [{ key: 1, value: { id: 1 } }], drafts: [] } } });
    const outcome = await new Promise<string>((res) => { const r = indexedDB.open(dbName, 3);
      r.onsuccess = () => { r.result.close(); res('ok'); }; r.onerror = () => res(String(r.error)); });
    expect(outcome).toBe('ok');  // ACTUAL: "VersionError: An attempt was made to open a database using a lower version than the existing version."
  });
  ```
- **Proposed fix:**
  - Add an option such as `onUnknownStore: 'create' | 'skip' | 'error'`, or `allowSchemaChanges: boolean`.
  - For library users who own the schema, default to `'skip'` or `'error'` (a breaking change; discuss first), or keep `'create'` and document it loudly.
  - Optionally accept a `targetVersion` so the caller can pick the version.
  - Document that `storeNames` can be used to avoid the bump.
- **Dedup:** NEW.

---

### IDB-06: `merge` silently ignores schema differences on stores that already exist

- **Type:** bug · **Severity:** medium · **Status:** VERIFIED · **Effort:** M
- **Evidence:** `src/core/importer.ts:113-119`: under `merge`, an existing store is skipped with `continue`, along with all of its indexes. The comment admits it. Index creation is in fact possible inside the version-change transaction that merge already opens when it needs new stores. When it doesn't, the library could just bump the version. A keyPath mismatch (existing inline `uid` vs backup out-of-line) surfaces as a bare `DataError`.
- **Repro:**

  ```ts
  it('merge does not create missing index', async () => {
    const dbName = uniqueDBName('merge-index');
    (await createTestDB(dbName, 1, [{ name: 'users', keyPath: 'id' }])).close();
    await importDB({ dbName, strategy: 'merge', backupData: { ...env, databaseVersion: 2,
      schema: { users: { keyPath: 'id', autoIncrement: false, indexes: [{ name: 'by_email', keyPath: 'email', unique: true, multiEntry: false }] } },
      stores: { users: [{ key: 1, value: { id: 1, email: 'a@x' } }] } } });
    const db = await openDB(dbName);
    expect(Array.from(db.transaction('users').objectStore('users').indexNames)).toContain('by_email'); // ACTUAL: []
  });
  it('keyPath mismatch', async () => {
    const dbName = uniqueDBName('merge-keypath');
    (await createTestDB(dbName, 1, [{ name: 'users', keyPath: 'uid' }])).close();
    const err = await importDB({ dbName, strategy: 'merge', backupData: { ...env,
      schema: { users: { keyPath: null, autoIncrement: false, indexes: [] } },
      stores: { users: [{ key: 'k1', value: { name: 'x' } }] } } }).catch((e) => e);
    // ACTUAL: "DataError: Data provided to an operation does not meet requirements." (no store/record context)
  });
  ```
- **Proposed fix:**
  - In the probe step, compare the backup schema with the existing stores (keyPath, autoIncrement, index set).
  - Create missing indexes in the upgrade transaction. Be careful with `unique` indexes over existing data, which can abort the upgrade.
  - On keyPath or autoIncrement mismatch, throw a descriptive `SchemaMismatchError` before writing anything.
  - Document the behaviour, and possibly add a `schemaMismatch: 'error' | 'ignore'` option.
- **Dedup:** NEW.

---

### IDB-07: `Error`, boxed primitives, `Float16Array` and BigInt wrappers are silently corrupted on export

- **Type:** bug · **Severity:** medium · **Status:** VERIFIED (end to end through IndexedDB) · **Effort:** M
- **Evidence:** `src/serialization/index.ts:221`. Any non-plain object that is not one of the handled classes falls through `return value`, and `JSON.stringify` then decides its fate. `TYPED_ARRAY_CONSTRUCTORS` (`:76-89`) has no `Float16Array`, which is now shipped in Chrome 135+, Firefox 129+, Safari 18.2+ and Node 24. All of these are structured-cloneable, so IndexedDB stores them fine.
- **Repro** (stored in IndexedDB → `exportDB` → `JSON` → `importDB`):

  ```ts
  const db = await createTestDB(src, 1, [{ name: 's', keyPath: 'id', records: [
    { value: { id: 1, e: new RangeError('boom'), n: new Number(5), h: new Float16Array([1.5, 2]) } }] }]);
  // stored:   { id: 1, e: RangeError: boom, n: [Number: 5], h: Float16Array(2) [1.5, 2] }
  // restored: { id: 1, e: {}, n: 5, h: ArrayBuffer <00 3e 00 40> }   (+ console.warn for Float16Array)

  // { v: Object(10n) }: exportDB() resolves, then JSON.stringify(backup) / downloadJSON() throws
  // "TypeError: Do not know how to serialize a BigInt"
  ```
- **Proposed fix:**
  - Add tags: `error` (name, message, stack?, cause; rebuild only allow-listed Error subclasses), `boxed` (`{kind:'number'|'string'|'boolean'|'bigint', value}`), and `Float16Array` in the typed-array allow-list (guarded by `typeof Float16Array !== 'undefined'`).
  - Add a final catch-all: any non-plain, unhandled object throws an `UnsupportedValueError` that includes the store, key and property path, instead of being emitted as `{}`.
  - Coordinate with #131 (Blob/File), which needs the same fail-loud default.
- **Dedup:** PARTIALLY COVERED. #131 covers Blob and File only, and explicitly names #81 / PR #82 as the earlier work. Error, boxed primitives, Float16Array and the BigInt wrapper are NEW. Comment on #131 or file a sibling issue.

---

### IDB-08: `deserialize()` and `ImportSummary.recordCounts` return null-prototype objects

- **Type:** bug · **Severity:** low–medium (public API surprise; records written through `importDB` are fine because structured clone re-prototypes them) · **Status:** VERIFIED · **Effort:** S
- **Evidence:** `src/serialization/index.ts:228-237`: `mapFields` builds `Object.create(null)` as a prototype-pollution defense, and `deserialize` (exported from `src/index.ts:6`) hands those objects to users. `exportDB()` record values come from the same function, and `buildImportSummary` uses `Object.create(null)` for `recordCounts` (`src/core/importer.ts:26`). The existing tests use `toEqual`, which ignores prototypes, so this has gone unnoticed.
- **Repro:**

  ```ts
  const restored = deserialize({ a: 1, nested: { b: 2 } }) as any;
  Object.getPrototypeOf(restored) === null;   // true
  typeof restored.hasOwnProperty;              // 'undefined'
  String(restored);                            // TypeError: Cannot convert object to primitive value
  restored instanceof Object;                  // false
  expect(restored).toStrictEqual({ a: 1, nested: { b: 2 } });   // FAILS
  ```
- **Proposed fix:** keep the defense but use ordinary objects. Build `{}` and assign with `Object.defineProperty(result, key, { value, enumerable: true, writable: true, configurable: true })`. `CreateDataProperty` semantics make `"__proto__"` an own data property without touching the prototype. Do the same for `recordCounts`. Add a `__proto__` key test and switch the round-trip assertions to `toStrictEqual`.
- **Dedup:** NEW. The 2026-09-19 security review praises the null-prototype choice but doesn't consider its API side effects.

---

### IDB-09: An object store named `__proto__` is silently dropped by `exportDB`

- **Type:** bug · **Severity:** low (rare name, but silent data loss) · **Status:** VERIFIED · **Effort:** S
- **Evidence:** `src/core/exporter.ts:144-151`: `schema` and `stores` are `{}` literals, and `schema[storeName] = …` with `storeName === '__proto__'` calls the `Object.prototype.__proto__` setter instead of creating a key. Store names can be any string. The import side survives because it gets its data from `JSON.parse` and `Object.fromEntries`.
- **Repro:**

  ```ts
  const dbName = uniqueDBName('proto-store');
  (await createTestDB(dbName, 1, [
    { name: '__proto__', keyPath: 'id', records: [{ value: { id: 1, secret: 'data' } }] },
    { name: 'normal', keyPath: 'id', records: [{ value: { id: 1 } }] }])).close();
  const b = JSON.parse(JSON.stringify(await exportDB({ dbName })));
  expect(Object.keys(b.schema)).toContain('__proto__');   // ACTUAL: ['normal'] (stores too)
  ```
- **Proposed fix:** build `schema` and `stores` with the same own-data-property helper as IDB-08 (or `Object.defineProperty`). Add the test above.
- **Dedup:** NEW.

---

### IDB-10: Add an atomic, non-destructive "replace" strategy

- **Type:** feature · **Severity:** medium · **Status:** n/a (feature) · **Effort:** M
- **Rationale:** `overwrite` = `deleteDatabase` + recreate. That path causes IDB-01, IDB-02 (#126) and IDB-03 (#114): it is non-atomic, it needs every other connection closed, and it resets the version. Most apps restoring their own backup just want "make these stores equal to the backup". A `'replace'` strategy (or `'clear'`) would:
  1. validate first (IDB-01)
  2. open the DB at its current version (add stores only if needed)
  3. in a **single readwrite transaction**, `store.clear()` each selected store, then `add()` every record
  4. call `transaction.abort()` on any error

  IndexedDB then guarantees all-or-nothing, open connections don't block, and the app's version stays valid (avoids IDB-05).
- **Proposed API:** `strategy: 'overwrite' | 'merge' | 'replace'`, with README guidance that `replace` is the safe default for "restore my app's data".
- **Dedup:** NEW. No issue or PR proposes it, and it complements PR #117 and PR #130.

---

### IDB-11: Make `importDB` return an `ImportResult`

- **Type:** feature · **Severity:** low–medium · **Status:** VERIFIED (current behaviour) · **Effort:** S
- **Evidence:** `src/core/importer.ts:393,416`: `Promise<void>`, and `if (!proceed) return;`. An aborted import and a completed import both resolve `undefined`. In plain JS (no TypeScript), a hook that forgets to `return true` silently aborts the restore.

  ```
  [IDB-G] importDB returned undefined ; db exists: false     <- hook returned undefined -> aborted, no signal
  [IDB-G] successful importDB also returned undefined
  ```
- **Proposed fix:**
  - Return `{ status: 'imported' | 'aborted', recordCounts, storesCreated, version }`. This is non-breaking for callers that ignore the value.
  - Optionally treat a non-boolean hook result as an error (`TypeError: onBeforeImport must return a boolean`).
  - A symmetrical `exportDB` metadata summary is optional.
- **Dedup:** NEW. #68 / PR #87 added the hook but not a result.

---

### IDB-12: Add `onProgress` and batched reads/writes for large databases

- **Type:** feature · **Severity:** low–medium · **Status:** n/a (feature). Baseline measured: 20 k small records export 394 ms / import 986 ms on fake-indexeddb · **Effort:** M–L
- **Evidence:** `exportDB` walks one cursor per record in one transaction and keeps the whole serialized DB in memory (`src/core/exporter.ts:41-65`). `insertRecords` deserializes and queues every record of every store in one transaction (`src/core/importer.ts:297-340`). There is no progress signal, so a UI can't show a progress bar for 100 MB+ wallets or offline caches.
- **Proposed fix:**
  - Use `getAll()` / `getAllKeys()` with key-range pagination (or `getAllRecords()` where available) to read in pages.
  - Add `onProgress({ store, done, total })` to both APIs, using `count()` for the total.
  - Optionally make `batchSize` configurable for merge, with a documented atomicity trade-off.
  - Optionally add `exportDBToBlob()` so big backups never need one giant JS string.
- **Dedup:** NEW.

---

### IDB-13: Speed up base64 encoding of binary values (about 11× faster)

- **Type:** improvement · **Severity:** low · **Status:** VERIFIED (benchmark) · **Effort:** S
- **Evidence:** `src/serialization/index.ts:16-22` concatenates one `String.fromCharCode` per byte.
- **Benchmark** (Node 24, happy-dom env, 16 MiB `Uint8Array`, byte-for-byte identical output):

  ```
  serialize (current): 2680 ms   chunked String.fromCharCode.apply(0x8000): 248 ms   deserialize: 84 ms
  ```
- **Proposed fix:** encode in 32 KiB chunks:

  ```ts
  let bin = '';
  for (let i = 0; i < bytes.length; i += 0x8000)
    bin += String.fromCharCode.apply(null, bytes.subarray(i, i + 0x8000) as unknown as number[]);
  return btoa(bin);
  ```

  Feature-detect `Uint8Array.prototype.toBase64` / `Uint8Array.fromBase64` (TC39 stage 4, shipping in Firefox/Safari/Chrome 2025+) and use them when present. Add a large-buffer round-trip test.
- **Dedup:** NEW.

---

### IDB-14: Add a PR CI workflow and run tests before tagging a release

- **Type:** ci · **Severity:** medium · **Status:** VERIFIED · **Effort:** S
- **Evidence:**
  - No workflow in `.github/workflows/` runs `npm test`, `lint`, `format:check`, `tsc` or `build` on `pull_request`. PR #117's checks were only label-conflicts, danger, sync-labels, CodeRabbit and Socket.
  - That is how `format:check` came to fail on `main` today (`src/utils/ssr.ts`). PR #117's author reported four failing files earlier.
  - `version-release.yml` creates the tag and GitHub release in job `release` and only runs `npm test` in the later `build` job. A red test leaves a published release with no npm package; the recently added "resume" logic exists because of this.
  - `BestPracticesChecklist.md` lists "no CI step running lint/test" and "no coverage measurement" as gaps (`warnings_fixed`, `test_most`).
  - The coverage config exists in `vitest.config.ts`, but `@vitest/coverage-v8` is not a devDependency and there is no script.
- **Proposed fix:**
  - Add `.github/workflows/ci.yml` on `pull_request` and `push: main`, with Node 20/22/24, `npm ci`, `format:check`, `lint`, `tsc --noEmit` (plus a test tsconfig), `build`, `test -- --coverage`, and `npx publint` / `attw --pack` (see IDB-15).
  - Pin actions by SHA, as the other workflows do, and use `harden-runner`.
  - Add `@vitest/coverage-v8` and a `test:coverage` script.
  - Move test and build ahead of tag creation in `version-release.yml`, or make the release job depend on CI.
  - Fix `ssr.ts` formatting in the same PR.
- **Dedup:** NEW. No issue or PR.

---

### IDB-15: CJS consumers get ESM type declarations ("Masquerading as ESM")

- **Type:** bug (packaging) · **Severity:** low–medium · **Status:** VERIFIED · **Effort:** S
- **Evidence:** `package.json:8-15`: a single `"types": "./dist/index.d.ts"` serves both `import` and `require`. Because the package is `"type": "module"`, TypeScript reads `index.d.ts` as ESM, even though tsup already emits `dist/index.d.cts`.
  - `attw --pack`: `node16 (from CJS) 👺 Masquerading as ESM`
  - `publint`: `pkg.exports["."].types types is interpreted as ESM when resolving with the "require" condition`
  - `publint` also suggests adding `"sideEffects": false`.
- **Proposed fix:**

  ```json
  "exports": { ".": {
    "import":  { "types": "./dist/index.d.ts",  "default": "./dist/index.js" },
    "require": { "types": "./dist/index.d.cts", "default": "./dist/index.cjs" } } },
  "sideEffects": false
  ```

  Add `attw --pack` and `publint --strict` to CI (IDB-14).
- **Dedup:** NEW.

---

### IDB-16: Close the test gaps

- **Type:** test · **Severity:** low · **Status:** VERIFIED (coverage report) · **Effort:** S–M
- **Evidence (uncovered lines, v8):**
  - `exporter.ts` 60, 79-81, 90-94. The **"database does not exist" path added by PR #100 has no exporter test.** I checked it by hand: it rejects with `Database "…" does not exist` and leaves no DB behind.
  - `importer.ts` 56, 60 (delete error / blocked), 109-110 (overwrite branch of `createStoresFromSchema`, effectively dead code), 182/186/224/226/244/248/269/277 (open/upgrade/create error and blocked branches), 334 (insert `onerror`).
  - `file.ts` 73 (`FileReader.onerror`).
  - Function coverage is 77.92 %.
  - Round-trip tests assert with `toEqual`, which hides prototype bugs (IDB-08).
  - `tsconfig.json:30` excludes `tests/`, so test types are never checked.
- **Proposed fix:**
  - Add tests: export of a missing DB, blocked delete/upgrade (with IDB-02), insert error, `FileReader` error, and a store named `__proto__` (IDB-09).
  - Switch to `toStrictEqual` where prototypes matter.
  - Add `tsconfig.test.json` (or `vitest --typecheck`) and wire it into CI.
  - Remove the dead overwrite branch at `importer.ts:105-112` or cover it.
- **Dedup:** NEW.

---

### IDB-17: README/docs claims don't match behaviour, and the APIs throw a raw `ReferenceError` outside the browser

- **Type:** docs (plus a small code guard) · **Severity:** low · **Status:** VERIFIED · **Effort:** S
- **Evidence:**
  - `README.md:62` says "SSR-safe: Graceful no-op guards". In reality `exportDB` / `importDB` have no guard: under plain Node, `exportDB({dbName:'x'})` throws **`ReferenceError: indexedDB is not defined`** (verified against `dist/index.js`). `downloadJSON` / `readFileAsJSON` *throw* rather than no-op.
  - `src/utils/ssr.ts:7` `isBrowser()` requires `window` and `document`. That makes it unsuitable as a guard for the IndexedDB APIs, which also work in Web Workers. Use `typeof indexedDB !== 'undefined'` and throw a clear error.
  - `README.md:64,73` and `AGENTS.md:7` say the library uses or wraps `idb`. It doesn't (knip; see IDB-18).
  - `README.md:206` still says "The core API is under active development … once the feature PRs are merged", although 1.1.0 is published.
  - `README.md:75` links happy-dom to `github.com/nicedayfor/happy-dom`, which is a 404. The correct repo is `capricorn86/happy-dom`.
  - There is no "Limitations" section: cycles (#95), Blob/File (#131), `undefined` (#132), NaN/±Infinity/-0 (#124), Error and boxed values (IDB-07), merge version bump (IDB-05), non-atomic overwrite (IDB-03).
  - The `deserialize` JSDoc (`src/serialization/index.ts:242-244`) promises `TypeError`/`RangeError` for malformed payloads, but invalid base64 throws `DOMException: InvalidCharacterError` (verified).
  - `BestPracticesChecklist.md` says "no SECURITY.md". An org-level `AOSSIE-Org/.github/SECURITY.md` does exist and applies, but private vulnerability reporting is **disabled** on this repo (`GET /private-vulnerability-reporting` → `enabled:false`). The org policy names GitHub Security Advisories as the preferred channel, so that should be enabled or the checklist corrected.
- **Proposed fix:**
  - Rewrite the Features bullets and add a "Limitations / unsupported values" section.
  - Remove the stale note and fix the link.
  - Add `assertIndexedDB('exportDB')` / `assertIndexedDB('importDB')` with a clear message.
  - Wrap `atob` failures in a `TypeError` that names the tag.
  - Update the checklist evidence.
- **Dedup:** NEW. The `idb` wording was part of #99/#101, both closed as "completed" without a code or docs change.

---

### IDB-18: Cleanup (one PR)

- **Type:** cleanup · **Severity:** low · **Effort:** S
- **Status:** VERIFIED by tools, except the `revokeObjectURL` item, which is SUSPECTED.

1. **Prettier fails on `main`:** `src/utils/ssr.ts:22` is missing a trailing comma (`npm run format:check`). VERIFIED.
2. **Unused runtime dependency `idb`** (`package.json:55`, knip). Every consumer installs it for nothing. VERIFIED. **Caution:** the maintainer closed #99 and #101 as "completed" without removing it. Ask on Discord which way they want to go (drop it, or actually adopt `idb` to replace the hand-rolled promise wrappers) before opening a PR.
3. **`src/core/migrations.ts`** is an empty TODO file that isn't exported (knip). Either implement it as part of IDB-01 (backupVersion validation and upgrade path) or delete it. VERIFIED.
4. **Mojibake in `CONTRIBUTING.md`:** lines 7 and 18 contain U+FFFD replacement characters (`## � IMPORTANT`, `## �📋 Table of Contents`). VERIFIED.
5. **`downloadJSON` revokes the object URL synchronously** right after `click()` (`src/utils/file.ts:35`). Some browsers, historically Safari and older Firefox, can cancel the download. Defer it with `setTimeout(() => URL.revokeObjectURL(url), 0–40_000)`, as FileSaver.js does. **SUSPECTED:** I could not reproduce it under happy-dom.
6. **Dev-dependency advisories:** 4 high and 11 moderate (happy-dom, vite, source-map-js, brace-expansion, minimatch). Merging the open Dependabot PRs #118 (vitest), #119 (eslint), #120 (fake-indexeddb), #121 (typescript-eslint) and #122 (happy-dom 21, a major bump) plus one `npm audit fix` clears them. No runtime exposure. VERIFIED.

- **Dedup:** item 2 is ALREADY REPORTED (#99/#101, closed). Items 1 and 3–6 are NEW.

---

## 4. Existing open issues: status on `main`

I re-checked every open issue against `main` @ `4b90eca`. All 6 still reproduce, so none should be refiled.

| Issue | Re-verified on main | Assignee | Linked PR | Notes |
| --- | --- | --- | --- | --- |
| #131 Blob/File silently lost | yes. Under happy-dom it shows up as `{"type":""}`; in browsers `{}` | none | **none** | **Pick-up-able** (see §5) |
| #132 `undefined` / holes lost | yes. `{"u":undefined,"arr":[1,,undefined]}` → `{"arr":[1,null,null]}` | none | PR #133 (by issue author, mergeable, no review yet) | |
| #126 blocked overwrite still deletes | (code unchanged) | none | PR #130 (CHANGES_REQUESTED by CodeRabbit) | Merge path not covered → IDB-02 |
| #124 NaN/Infinity/-0 lost | yes. `{n:NaN,i:Infinity,z:-0}` → `{"n":null,"i":null,"z":0}` | none | PR #125 | |
| #114 invalid record loses data | yes. Merge leaves keys `[1,2,11]`; overwrite leaves 0 rows | none | PR #117 (APPROVED) | Write-time errors not fixed → IDB-03 |
| #95 cycles / shared refs | yes. `RangeError: Maximum call stack size exceeded` | none | PR #127 (earlier PR #102 closed) | |

Open PRs (12): #133, #130, #129 (dependabot harden-runner), #128 (draft, no issue → IDB-04), #127, #125, #122, #121, #120, #119, #118 (dependabot), #117.

---

## 5. Pick-up-able existing issues (no assignee and no linked PR)

- **#131: "[BUG]: Blob and File values are silently lost in exportDB, restored as empty objects".** No assignee and no PR. The author asked about the approach and hasn't got an answer yet. Notes for whoever picks it up:
  - The design choice is async export: read `blob.arrayBuffer()` *after* the cursor transaction finishes, because awaiting inside the transaction would auto-commit it. Collect the raw records first, then serialize; `exportDB` is already async.
  - Alternatively, fail loudly, which pairs with the catch-all in IDB-07.
  - Import is synchronous (`new Blob([bytes], {type})`, `new File(...)`).
  - Confirm the approach on Discord first, as CONTRIBUTING requires.

Every other open issue already has an open PR, and none has an assignee. Partially open follow-ups the team can claim as **new** issues:

- The #114 write-time-error follow-up that PR #117 explicitly offered but never filed (IDB-03).
- The merge-path counterpart of #126 (IDB-02).
- A tracking issue for draft PR #128 plus the Invalid Date encoding (IDB-04).

### Suggested split for a 3-person team

- **Person A (importer safety):** IDB-01 → IDB-03 → IDB-10.
- **Person B (connections/schema):** IDB-02 → IDB-05 → IDB-06 → IDB-11.
- **Person C (serialization + infra):** IDB-14 (land CI first, so everyone else's PRs get checks) → IDB-15 → IDB-08/IDB-09 → IDB-07 / #131 → IDB-13 → IDB-17/IDB-18.

Per CONTRIBUTING, open the issue, get assigned, and post in the project's Discord channel before sending the PR.
