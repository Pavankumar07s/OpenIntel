# [BUG]: Backups with more than ~380 MiB of binary data can't be saved or read in Chromium (downloadJSON throws RangeError: Invalid string length)

### Bug Description

`downloadJSON` turns the whole backup into one JavaScript string (`JSON.stringify(data, null, 2)`, `src/utils/file.ts:25`). `readFileAsJSON` reads the whole file into one string (`reader.readAsText(file)`, `src/utils/file.ts:76`) and then calls `JSON.parse`. V8 caps a string at 2^29 - 24 = 536,870,888 characters. Binary values are stored as base64, which takes 4 characters for every 3 bytes. So in Chrome, Edge and other Chromium browsers, a database with more than about 380 MiB of `Uint8Array`/`ArrayBuffer`/typed-array data can't be turned into a backup file:

- `exportDB` resolves (it builds the backup as an in-memory object in `src/core/exporter.ts:41-65` and `:143-163`), but `downloadJSON(backup, ...)` throws `RangeError: Invalid string length`. The library has no other API that writes the backup to a file.
- A backup file above the same size can't be read back either. `readFileAsJSON` rejects with a `NotReadableError` whose browser message talks about "permission problems", which doesn't help find the real cause.

Pretty-printing isn't the cause in this case. At 380 MiB, the pretty-printed and compact JSON differ by only about 11,000 characters, so `JSON.stringify(backup)` hits the same limit.

The exact ceiling depends on the engine. In Firefox 158, `'a'.repeat(2 ** 30 - 2)` succeeds, so the limit there is about twice as high. But a backup made in Firefox above ~380 MiB still can't be imported in Chrome.

### Steps to Reproduce

In Chrome, using the README's export/download flow:

```js
import { exportDB, downloadJSON, readFileAsJSON } from '@aossie-org/idb-backup';

// 96 records of 4 MiB each = 384 MiB of binary data
const db = await new Promise((resolve, reject) => {
  const req = indexedDB.open('big', 1);
  req.onupgradeneeded = () => req.result.createObjectStore('photos', { keyPath: 'id' });
  req.onsuccess = () => resolve(req.result);
  req.onerror = () => reject(req.error);
});
for (let i = 0; i < 96; i++) {
  await new Promise((resolve, reject) => {
    const tx = db.transaction('photos', 'readwrite');
    tx.objectStore('photos').put({ id: i, data: new Uint8Array(4 * 1024 * 1024) });
    tx.oncomplete = resolve;
    tx.onerror = () => reject(tx.error);
  });
}
db.close();

const backup = await exportDB({ dbName: 'big' }); // resolves
downloadJSON(backup, 'backup.json');              // throws RangeError: Invalid string length
```

Import side: build the same backup as a file in parts, so that no single string holds all of it, and read it back:

```js
const { stores, ...head } = backup;
const parts = [JSON.stringify({ ...head, stores: {} }).slice(0, -2), '"stores":{"photos":['];
stores.photos.forEach((r, i) => parts.push((i ? ',' : '') + JSON.stringify(r)));
parts.push(']}}');
const file = new File(parts, 'backup.json', { type: 'application/json' }); // 536,879,502 bytes
await readFileAsJSON(file); // rejects
```

### Logs and Screenshots

Results on headless Chrome 155 (4 MiB records):

| Stored | `exportDB` | `downloadJSON` | `JSON.stringify(backup)` |
| --- | --- | --- | --- |
| 95 records (380 MiB) | resolves, ~18 s | ok, Blob of 531,298,286 bytes | 531,286,999 chars |
| 96 records (384 MiB) | resolves, ~18.6 s | `RangeError: Invalid string length` | `RangeError: Invalid string length` |

```
readFileAsJSON(536,879,502-byte file):
Error: Failed to read file "backup.json": NotReadableError: The requested file could not be read, typically due to permission problems that have occurred after a reference to a file was acquired.
```

The engine limit itself, in the same browser:

```
'a'.repeat(2 ** 29 - 24)  // ok
'a'.repeat(2 ** 29 - 23)  // RangeError: Invalid string length
```

### Environment Details

- `@aossie-org/idb-backup` built from `main` at 4b90eca (1.1.0)
- Google Chrome 155.0.8059.79 (headless), Linux x86_64
- Firefox 158.0 (string-limit comparison only)

### Impact

Medium - Feature works but has issues

Backups work up to about 380 MiB of binary data. Above that, in Chromium browsers, the documented export/restore flow can't produce or read a backup file.

### Proposed fix

- **Export:** add `exportDBToBlob(options)`, or let `downloadJSON` accept a `Blob`. It would build the file from parts (the envelope first, then one `JSON.stringify(record)` per record) and pass them to `new Blob(parts)`, so no single string holds the whole backup. The output stays in the current v1 format.
- **Import:** write each record on its own line. The file stays valid v1 JSON. Then add a reader that goes through `file.stream().pipeThrough(new TextDecoderStream())` line by line and falls back to `JSON.parse` for files laid out differently. This needs no new runtime dependency and no format version bump.
- **Short term:** document the limit in the README. Catch the `RangeError` / `NotReadableError` and rethrow it with a message that says the backup is larger than the browser's maximum string length.
- **Tests:** a fixture of more than 512 MiB isn't practical under happy-dom. Test the chunked writer and reader with small inputs instead, and check that the output parses back to the same `ExportFormat`.

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
