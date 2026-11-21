# [BUG]: `deserialize()` and `serialize()` return null-prototype objects, unlike the records IndexedDB itself returns

## Bug Description

`deserialize` and `serialize` are part of the public API (`src/index.ts:6`). Every plain object they return, at every nesting level, has a `null` prototype. The cause is `mapFields` (`src/serialization/index.ts:228-237`), which builds each result with `Object.create(null)`. `exportDB()` record values come from `serialize`, so the in-memory `ExportFormat` it returns has null-prototype record values too.

This is different from what callers get from IndexedDB for the same record. A record restored through `importDB` and read back with `getAll()` is an ordinary object, because structured clone re-prototypes it. The same record passed through `deserialize()` directly is not. As a result:

- `obj.hasOwnProperty(...)`, `obj.toString()` and the other `Object.prototype` methods are `undefined`
- `String(obj)`, `` `${obj}` `` and `'x' + obj` throw `TypeError: Cannot convert object to primitive value`
- `obj instanceof Object` is `false`
- `expect(deserialize(x)).toStrictEqual({...})` fails, although `toEqual` passes (the existing tests use `toEqual`, so they don't catch this)

The null prototype is a deliberate defense against keys like `"__proto__"`, and the defense should stay. The same safety is available with ordinary objects (see Proposed fix), so callers don't have to pay for it.

`ImportSummary.recordCounts` (`src/core/importer.ts:26`) is also built with `Object.create(null)`, so `summary.recordCounts.hasOwnProperty('x')` throws there as well. That one is a string-keyed lookup table, and a null prototype has a real benefit: looking up a missing name such as `constructor` returns `undefined`, which the README's `summary.recordCounts.messages ?? 0` pattern depends on. So `recordCounts` should stay as it is. This issue is only about the record values from `deserialize`/`serialize`.

Normal `exportDB` → `downloadJSON` → `readFileAsJSON` → `importDB` use is **not** affected: `JSON.stringify` handles null-prototype objects, and IndexedDB stores ordinary objects. Only code that uses the returned objects directly is affected.

## Steps to Reproduce

Vitest, with the repo's own setup (`happy-dom` + `fake-indexeddb/auto`), importing from `src/index`:

```ts
import { it, expect } from 'vitest';
import 'fake-indexeddb/auto';
import { deserialize, serialize } from '../src/index';

it('deserialize returns ordinary objects', () => {
  const parsed = JSON.parse('{"a":1,"nested":{"b":2}}'); // ordinary objects in
  const out = deserialize(parsed) as any;

  expect(Object.getPrototypeOf(out)).toBe(Object.prototype);        // fails: null
  expect(Object.getPrototypeOf(out.nested)).toBe(Object.prototype); // fails: null
  expect(() => String(out)).not.toThrow();                          // fails: TypeError
  expect(out).toStrictEqual({ a: 1, nested: { b: 2 } });            // fails
  expect(Object.getPrototypeOf(serialize({ a: 1 }))).toBe(Object.prototype); // fails: null
});
```

For comparison, after `importDB({ ..., strategy: 'overwrite' })` with the same backup, `getAll()` on the restored store returns records whose prototype is `Object.prototype`.

## Logs and Screenshots

Observed on `main` (4b90eca):

```
Object.getPrototypeOf(deserialize(parsed)) === null        -> true  (input prototype was Object.prototype)
Object.getPrototypeOf(deserialize(parsed).nested) === null -> true
typeof out.hasOwnProperty                                   -> 'undefined'
out instanceof Object                                       -> false
String(out)        -> TypeError: Cannot convert object to primitive value
`${out.nested}`    -> TypeError: Cannot convert object to primitive value
expect(out).toStrictEqual({...}) -> AssertionError: expected { a: 1, nested: { b: 2 }, …(1) } to strictly equal { a: 1, nested: { b: 2 }, …(1) }
expect(out).toEqual({...})       -> pass
serialize({ a: 1 }) prototype            -> null
exportDB() stores.users[0].value prototype -> null
summary.recordCounts.hasOwnProperty('users') -> TypeError: rc.hasOwnProperty is not a function
record read back from IndexedDB after importDB -> prototype is Object.prototype
```

## Environment Details

- `@aossie-org/idb-backup` 1.1.0, `main` @ 4b90eca
- Node v24.22.1, Vitest 5.0.1, happy-dom 20.14.5, fake-indexeddb 6.2.5
- The behaviour is plain JavaScript semantics, so it is the same in any browser

## Impact

Low - Minor inconvenience

## Proposed fix

Keep the `"__proto__"` defense, but build ordinary objects in `mapFields`. Define each key as an own data property instead of assigning it:

```ts
const result: Record<string, unknown> = {};
for (const key of Object.keys(value)) {
  Object.defineProperty(result, key, {
    value: fn(value[key]),
    enumerable: true,
    writable: true,
    configurable: true,
  });
}
```

`Object.defineProperty` makes `"__proto__"` an own data property and does not call the `Object.prototype.__proto__` setter. This is the same `CreateDataProperty` behaviour `JSON.parse` already relies on. I checked that for the input `{"__proto__":{"polluted":1},"a":1}`, the result has own keys `['__proto__', 'a']`, its prototype is `Object.prototype`, and neither the result nor `Object.prototype` gets a `polluted` property.

Tests:
- add a `deserialize`/`serialize` test with a `"__proto__"` key that asserts no prototype change and no pollution
- assert `Object.getPrototypeOf(...) === Object.prototype` on round-trip results
- switch the round-trip assertions in `tests/serialization.test.ts` to `toStrictEqual`

`isPlainObject` already accepts both prototypes, so nothing else needs to change. Open PR #127 also edits `mapFields`/`deserialize` and keeps `Object.create(null)`, so whichever lands second will need a small rebase.
