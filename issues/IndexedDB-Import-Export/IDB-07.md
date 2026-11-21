# [BUG]: Error objects become `{}` in a backup, and Float16Array and boxed primitives are not restored

### Bug Description

IndexedDB's structured clone stores `Error` objects, boxed primitives (`new Number(5)`, `new String('a')`, `new Boolean(false)`, `Object(10n)`) and `Float16Array` without trouble. A backup made with `exportDB` and restored with `importDB` doesn't bring them back:

| Stored value | In the backup file | After `importDB` | Notes |
| --- | --- | --- | --- |
| `new TypeError('bad input')` (any `Error`) | `{}` | `{}` | **Silent loss.** `name`, `message`, `stack` and `cause` are all dropped, with no warning |
| `new Number(5)` / `new String('ab')` / `new Boolean(false)` | `5` / `"ab"` / `false` | primitive | The value survives but the wrapper is gone (`typeof` goes from `object` to `number`) |
| `Object(10n)` (BigInt wrapper) | n/a | n/a | `exportDB` resolves, then `JSON.stringify(backup)` / `downloadJSON(backup)` throws `TypeError: Do not know how to serialize a BigInt`, so the backup can't be saved |
| `new Float16Array([1.5, 2])` | `{"__type":"typed_array","value":{"type":"Float16Array","data":"AD4AQA=="}}` (complete) | plain `ArrayBuffer` | Export is fine. Import logs `Unknown typed array type "Float16Array" — returning an ArrayBuffer.` |

Causes in `src/serialization/index.ts`:
- `serialize()` has no case for these classes. Any non-plain object that it doesn't recognise falls through to the final `return value` (line 221), and `JSON.stringify` then turns it into `{}`, a primitive, or a throw.
- The `TYPED_ARRAY_CONSTRUCTORS` allow-list used by `deserialize()` (lines 76-89) has no `Float16Array`. The README promises a "Lossless round-trip for … typed arrays". `Float16Array` is a standard typed array (Chrome 135+, Firefox 129+, Safari 18.2+, Node 24).

#131 covers Blob/File values, which hit the same fall-through.

### Steps to Reproduce

Serialization only (run against the built package, Node 24):

```js
import { serialize, deserialize } from '@aossie-org/idb-backup';

const record = {
  err: new TypeError('bad input'),
  n: new Number(5),
  s: new String('ab'),
  half: new Float16Array([1.5, 2]),
};
const json = JSON.stringify(serialize(record));
console.log(json);
console.log(deserialize(JSON.parse(json)));

JSON.stringify(serialize({ v: Object(10n) })); // throws
```

End to end, using the test suite's `fake-indexeddb` setup and the `createTestDB` / `readAllFromStore` helpers from `tests/helpers/idb-helpers.ts`:

```ts
const db = await createTestDB('src', 1, [{ name: 'logs', keyPath: 'id', records: [
  { value: { id: 1, err: new RangeError('boom'), n: new Number(5), half: new Float16Array([1.5, 2]) } },
] }]);
db.close();

const backup = JSON.parse(JSON.stringify(await exportDB({ dbName: 'src' }))); // as a saved file
await importDB({ dbName: 'dst', backupData: backup, strategy: 'overwrite' });
console.log((await readAllFromStore('dst', 'logs'))[0].value);
```

**Expected:** the restored record matches the stored one (`err` is a `RangeError` with message `boom`, `n` is a `Number` object, `half` is a `Float16Array`).

**Actual:** `{ id: 1, err: {}, n: 5, half: ArrayBuffer <00 3e 00 40> }`, plus a console warning for the `Float16Array`.

### Logs and Screenshots

```
{"err":{},"n":5,"s":"ab","half":{"__type":"typed_array","value":{"type":"Float16Array","data":"AD4AQA=="}}}
[idb-backup] Unknown typed array type "Float16Array" — returning an ArrayBuffer.
[Object: null prototype] {
  err: [Object: null prototype] {},
  n: 5,
  s: 'ab',
  half: ArrayBuffer { [Uint8Contents]: <00 3e 00 40>, [byteLength]: 4 }
}
TypeError: Do not know how to serialize a BigInt
```

### Environment Details

- `@aossie-org/idb-backup` 1.1.0 (`main` @ 4b90eca)
- Node v24.22.1, Vitest 5 + `fake-indexeddb` + `happy-dom` (the repo's test setup)
- The loss comes from `serialize()` + `JSON.stringify`, so it happens the same way in every browser. The `Float16Array` part only applies where `Float16Array` exists.

### Impact

Low - Minor inconvenience. These value types are uncommon in IndexedDB data. When an `Error` is stored, though (for example in an error or telemetry log store), its contents are lost from the backup without any warning.

### Proposed fix

All changes are in `src/serialization/index.ts` and `src/types/index.ts`, with no new dependencies:

1. **Float16Array (one line):** add it to `TYPED_ARRAY_CONSTRUCTORS` when it exists, e.g. `...(typeof (globalThis as any).Float16Array === 'function' ? { Float16Array: (globalThis as any).Float16Array } : {})`. `tsconfig` targets `lib: ES2020`, so it has to be read off `globalThis`. Backups that already contain Float16Array data then restore correctly, because the bytes are already in the file.
2. **Error:** add an `error` tag `{ name, message, stack?, cause? }` (with `cause` serialized recursively). On import, rebuild only an allow-listed set of constructors (`Error`, `EvalError`, `RangeError`, `ReferenceError`, `SyntaxError`, `TypeError`, `URIError`), falling back to `Error`. This mirrors what structured clone keeps.
3. **Boxed primitives:** add a `boxed` tag `{ kind: 'number' | 'string' | 'boolean' | 'bigint', value }`. At minimum, unwrap `Object(10n)` to the existing `bigint` tag so the backup can be stringified.
4. Add tests in `tests/serialization.test.ts` and `tests/round-trip.test.ts`, and update the type table in the README.

Related: #131 (Blob/File, same fall-through), #81 / PR #82 (Set/Map/ArrayBuffer/RegExp/typed arrays), #124 and #132 (other values lost in the JSON step).

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
