# [BUG]: CryptoKey, FileSystemHandle and RTCCertificate values are exported as `{}` with no warning, and restoring in place replaces the stored key

### Bug Description

IndexedDB is the usual place to keep a non-extractable WebCrypto `CryptoKey` or a File System Access handle, since structured clone can store them. `exportDB()` can't represent these values, but it doesn't say so: it writes `{}` and returns as if the backup were complete.

- `serialize()` (`src/serialization/index.ts`) has no case for these types, and they fail `isPlainObject` (lines 46-56), so they fall through to the final `return value` (line 221).
- `readAllRecords()` (`src/core/exporter.ts:49-52`) puts the raw object into the backup. `JSON.stringify` then writes `{}`, because `CryptoKey`, `FileSystemFileHandle` and `RTCCertificate` have no own enumerable properties.
- Nothing is logged.

The worst case is an in-place restore. If a backup of a database that holds a key is later restored into that same database, the key is replaced with `{}`. This happens with `strategy: 'overwrite'` and also with `strategy: 'merge'`, because merge upserts with `put()`. A non-extractable key can't be recreated, so any data encrypted with it can no longer be decrypted.

Other serializable browser types are kept only partly, again with no warning:
- `ImageData`: only its own `data` field is written, as an index-keyed object. `width`, `height` and `colorSpace` are lost.
- `DOMRect`, `DOMPoint` and `DOMMatrix`: their `toJSON()` fields are written, and they come back as plain objects.

A non-extractable key can't be exported by any library, by design. So the fix is not to preserve these values. It's to stop reporting success when the backup is missing data. (dexie-export-import's README, for comparison, says it supports structured-cloneable types "except CryptoKeys (which by design cannot be exported)".)

### Steps to Reproduce

**1. In the repo's own test setup (Vitest + fake-indexeddb + happy-dom)**

This works in Node 20+, which has a structured-cloneable global `CryptoKey`. Save it as `tests/cryptokey-repro.test.ts` and run `npx vitest run tests/cryptokey-repro.test.ts`:

```ts
import { it, expect } from 'vitest';
import { exportDB } from '../src/core/exporter.js';
import { importDB } from '../src/core/importer.js';
import { setupFakeIDB, createTestDB, readAllFromStore } from './helpers/idb-helpers.js';

setupFakeIDB();

it('a CryptoKey survives export -> JSON -> in-place restore', async () => {
  const key = await crypto.subtle.generateKey({ name: 'AES-GCM', length: 256 }, false, ['encrypt', 'decrypt']);
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const ct = await crypto.subtle.encrypt({ name: 'AES-GCM', iv }, key, new Uint8Array([1, 2, 3]));

  const db = await createTestDB('keys', 1, [
    { name: 'keys', keyPath: 'id', records: [{ value: { id: 'main', key } }] },
  ]);
  db.close();

  const json = JSON.stringify(await exportDB({ dbName: 'keys' })); // as a saved backup file
  await importDB({ dbName: 'keys', backupData: JSON.parse(json), strategy: 'overwrite' });

  const [row] = await readAllFromStore('keys', 'keys');
  const restored = (row!.value as { key: unknown }).key;
  await expect(crypto.subtle.decrypt({ name: 'AES-GCM', iv }, restored as CryptoKey, ct)).resolves.toBeDefined();
});
```

**2. In a real browser (Chrome), for the browser-only types**

Using the built `dist/index.js`:

```js
import { exportDB, importDB } from './dist/index.js';

const values = [
  await crypto.subtle.generateKey({ name: 'AES-GCM', length: 256 }, false, ['encrypt', 'decrypt']),
  (await crypto.subtle.generateKey({ name: 'ECDSA', namedCurve: 'P-256' }, false, ['sign', 'verify'])).privateKey,
  await (await navigator.storage.getDirectory()).getFileHandle('note.txt', { create: true }),
  await RTCPeerConnection.generateCertificate({ name: 'ECDSA', namedCurve: 'P-256' }),
  new ImageData(2, 2),
  new DOMRect(1, 2, 3, 4),
  new DOMPoint(1, 2, 3, 1),
  new DOMMatrix([1, 2, 3, 4, 5, 6]),
];
// Put each one as { id: i + 1, v } into object store 's' (keyPath 'id') of database 'src', then:
const json = JSON.stringify(await exportDB({ dbName: 'src' }));
await importDB({ dbName: 'dst', backupData: JSON.parse(json), strategy: 'overwrite' });
// Read store 's' of 'dst' back and inspect each `v`.
```

**Expected:** `exportDB` fails with an error naming the value it can't represent, or at least reports which fields it skipped. It should not return a backup that looks complete.

**Actual:** the export succeeds silently, and the values are replaced as shown below.

### Logs and Screenshots

Repo test setup (case 1):

```
"stores":{"keys":[{"key":"main","value":{"id":"main","key":{}}}]}}
restored key: [object Object]
TypeError: Failed to execute 'decrypt' on 'SubtleCrypto': 2nd argument is not of type CryptoKey.
```

Chrome 155 (case 2):

| Stored value | In the backup file | After `importDB` |
| --- | --- | --- |
| `CryptoKey` AES-GCM secret (`extractable: false`, and also `true`) | `{}` | `{}` |
| `CryptoKey` ECDSA private (`extractable: false`) | `{}` | `{}` |
| `FileSystemFileHandle` (`note.txt`) | `{}` | `{}` |
| `RTCCertificate` | `{}` | `{}` |
| `ImageData` 2x2 | `{"data":{"0":0,…,"15":0}}` | plain object, no `width`/`height` |
| `DOMRect(1,2,3,4)` | `{"x":1,"y":2,"width":3,"height":4,"top":2,"right":4,"bottom":6,"left":1}` | plain object |
| `DOMPoint` / `DOMMatrix` | their `toJSON()` fields | plain object |

No console warning was printed. After restoring the backup into `src` itself (with either `overwrite` or `merge`), decrypting with the restored key fails:

```
TypeError: Failed to execute 'decrypt' on 'SubtleCrypto': parameter 2 is not of type 'CryptoKey'.
```

### Environment Details

- `@aossie-org/idb-backup` 1.1.0 (`main` @ 4b90eca)
- Case 1: Node v24.22.1, Vitest 5.0.1 with `fake-indexeddb` and `happy-dom` (the repo's test config)
- Case 2: Google Chrome 155.0.8059.79, headless, Linux, loading the built `dist/index.js` from a local server
- The silent `{}` comes from `serialize()` + `JSON.stringify`, so it happens the same way in every browser that can store these types

### Impact

Medium - Feature works but has issues

Ordinary data backs up fine. Apps that store keys or file handles get an incomplete backup with no warning, and an in-place restore permanently destroys their key.

### Proposed fix

1. In `serialize()`, after the known cases, treat any remaining non-plain object as unsupported. Throw a `TypeError` that names the type (`Object.prototype.toString.call(value)`), and have `exportDB` add the store name and record key. If partial backups are wanted, add an option such as `onUnsupportedValue: 'throw' | 'skip'` (default `'throw'`) that drops the field and returns the skipped paths.
2. This depends on #128. Today a throw inside the cursor `onsuccess` callback leaves `exportDB()` pending forever, so without #128 the throw would turn silent data loss into a hang.
3. Handle `ImageData`, `DOMRect`, `DOMPoint` and `DOMMatrix` explicitly, so the new throw doesn't break exports that work today. Either add tags for them (for example `{ width, height, colorSpace, data: <base64> }` for `ImageData`, guarded by `typeof ImageData !== 'undefined'`), or exempt objects that have a `toJSON` method and document that they come back as plain objects.
4. Tests: the CryptoKey test above runs in the existing Vitest + `fake-indexeddb` setup, so no real browser is needed. The catch-all can be unit-tested with `serialize()` and any non-plain object.
5. Add the unsupported types to the README.

Related: #131 (Blob/File hit the same fall-through, so whatever fail-loud default is chosen there should apply here), #128 (export hangs when `serialize()` throws), #127 (rewrites `serialize()`, so a fix needs to be coordinated with it).

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
