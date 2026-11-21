# [BUG]: exportDB silently drops an object store named `__proto__`

## Bug Description

`exportDB` builds the backup's `schema` and `stores` maps as plain `{}` literals and fills them with bracket assignment (`src/core/exporter.ts:144-151`):

```ts
const schema: Record<string, StoreSchema> = {};
const stores: Record<string, Array<{ key: unknown; value: unknown }>> = {};
// ...
schema[storeName] = extractStoreSchema(store);
stores[storeName] = await readAllRecords(store);
```

IndexedDB accepts any string as an object store name, `"__proto__"` included. For that name, `schema["__proto__"] = ...` does not create a key. It calls the inherited `Object.prototype.__proto__` setter, which replaces the object's prototype. So:

- the store is left out of the backup's own keys, so `JSON.stringify` (and `downloadJSON`) drops its schema and every record without any error;
- the in-memory return value is also changed: `backup.schema` now inherits from that store's schema object (`backup.schema.keyPath === 'id'`), and `backup.stores` inherits from the records array.

This only changes the returned object's prototype. `Object.prototype` itself is not polluted.

The import side already handles this name. `JSON.parse` and `Object.fromEntries` create a real own `"__proto__"` key, and `importDB` creates and fills the store correctly. Only export is affected, which means a database restored from a backup that has such a store can't be exported again. The same kind of issue came up for `recordCounts` in the importer during review of #87, and it was fixed there with a null-prototype object.

## Steps to Reproduce

Run this in a browser, or in Vitest with the repo's `fake-indexeddb` + `happy-dom` setup:

```ts
import { exportDB, importDB } from '@aossie-org/idb-backup';

// 1. Create a database with a store named "__proto__"
await new Promise<void>((resolve, reject) => {
  const req = indexedDB.open('proto-demo', 1);
  req.onupgradeneeded = () => {
    for (const name of ['__proto__', 'items']) {
      req.result.createObjectStore(name, { keyPath: 'id' }).put({ id: 1, from: name });
    }
  };
  req.onsuccess = () => { req.result.close(); resolve(); };
  req.onerror = () => reject(req.error);
});

// 2. Export it, then round-trip the backup through JSON the way a file would
const backup = await exportDB({ dbName: 'proto-demo' });
console.log(Object.keys(backup.schema));                     // ['items']
console.log((backup.schema as any).keyPath);                 // 'id' (inherited)
const parsed = JSON.parse(JSON.stringify(backup));
console.log(JSON.stringify(backup).includes('__proto__'));   // false

// 3. Restore into a fresh database
await importDB({ dbName: 'proto-restore', backupData: parsed, strategy: 'overwrite' });
// the restored DB has only ['items']; the "__proto__" store and its records are gone
```

`exportDB({ dbName: 'proto-demo', storeNames: ['__proto__'] })` returns `schema: {}` and `stores: {}`.

**Expected:** `Object.keys(backup.schema)` and `Object.keys(backup.stores)` contain `'__proto__'`, the JSON includes that store, and a restore brings it back.

**Actual:** the store is missing from the export, and there is no error or warning.

## Logs and Screenshots

Output on `main` (other store names that clash with `Object.prototype`, such as `constructor` and `toString`, export fine):

```
source stores: [ '__proto__', 'constructor', 'items', 'toString' ]
own schema keys: [ 'constructor', 'items', 'toString' ]
own stores keys: [ 'constructor', 'items', 'toString' ]
schema prototype is Object.prototype? false
stores prototype is an array? true
inherited backup.schema.keyPath = id
JSON contains "__proto__"? false
storeNames:["__proto__"] -> schema/stores: {} {}
restored stores: [ 'items' ]
```

When a backup JSON that already contains a `"__proto__"` store is imported, the store is created with its records. Exporting that database again then gives `schema keys: []`.

## Environment Details

- `@aossie-org/idb-backup` 1.1.0, `main` @ `4b90eca`
- Node v24.22.1, Vitest 5.0.1, `fake-indexeddb` + `happy-dom` (the repo's test setup)
- This doesn't depend on the engine: assigning to `obj["__proto__"]` on a `{}` literal calls the inherited setter in every JS engine, and the IndexedDB spec allows any string as a store name.

## Impact

Low - Minor inconvenience (a rare store name, but the data loss is silent when it does happen)

## Proposed fix

Build `schema` and `stores` with `Object.fromEntries`, which defines own data properties, so `"__proto__"` becomes an ordinary key and both objects keep `Object.prototype`:

```ts
const results = await Promise.all(
  targetStores.map(async (storeName) => {
    const store = transaction.objectStore(storeName);
    const storeSchema = extractStoreSchema(store);
    return { storeName, storeSchema, records: await readAllRecords(store) };
  }),
);
const schema = Object.fromEntries(results.map((r) => [r.storeName, r.storeSchema]));
const stores = Object.fromEntries(results.map((r) => [r.storeName, r.records]));
```

`Object.defineProperty` with an enumerable, writable, configurable data property would work too. I tried the change above locally: the existing suite still passes, and so do `tsc --noEmit` and ESLint. Add a regression test to `tests/exporter.test.ts` (or `tests/round-trip.test.ts`) that exports a database with a `__proto__` store and checks `Object.keys(backup.schema)` / `Object.keys(backup.stores)` after a JSON round trip.
