# [BUG]: README overstates SSR safety and has stale/incorrect details; exportDB/importDB throw a raw ReferenceError outside the browser

### Bug Description

Several statements in `README.md` (and one JSDoc) don't match the code on `main` (4b90eca):

1. **"SSR-safe: Graceful no-op guards"** (`README.md:62`; line 53 also says "includes SSR safety guards"). Nothing in the library no-ops:
   - `downloadJSON` / `readFileAsJSON` call `assertBrowser()`, which **throws** a clear `Error`. That behaviour is fine, but it is not a no-op.
   - `exportDB` / `importDB` have **no guard at all**. Outside a browser they fail with `ReferenceError: indexedDB is not defined`.
   - The JSDoc in `src/utils/ssr.ts` says to use `assertBrowser` in front of "IndexedDB" APIs, but the core functions don't. `isBrowser()` also requires `window` and `document`, so it would be the wrong guard for IndexedDB, which also works in Web Workers.
2. **Stale note** (`README.md:206`): "The core API is under active development. The examples below show the intended usage once the feature PRs are merged." The API has been published on npm since 1.0.0. The current version is 1.1.0.
3. **Broken link** (`README.md:75`): happy-dom links to `github.com/nicedayfor/happy-dom`, which returns 404. The project is `github.com/capricorn86/happy-dom`.
4. **Outdated tooling** (`README.md:76`): says "ESLint 9", but `package.json` uses ESLint 10.
5. **`deserialize` JSDoc** (`src/serialization/index.ts`, above `deserialize`): says a malformed payload for a known tag throws a `TypeError` (or a `RangeError` for an invalid date). Other error types come out as well:
   - invalid base64 in a `u8` / `buffer` value → `DOMException: InvalidCharacterError`
   - a non-numeric `bigint` value → `SyntaxError: Cannot convert abc to a BigInt`

### Steps to Reproduce

```bash
npm run build
node --input-type=module -e "
import { exportDB, importDB, downloadJSON } from './dist/index.js';
try { await exportDB({ dbName: 'x' }); } catch (e) { console.log('exportDB ->', e.constructor.name + ': ' + e.message); }
try { await importDB({ dbName: 'x', strategy: 'merge', backupData: { backupVersion: 1, databaseName: 'x', databaseVersion: 1, exportedAt: '', schema: {}, stores: {} } }); } catch (e) { console.log('importDB ->', e.constructor.name + ': ' + e.message); }
try { downloadJSON({}, 'a.json'); } catch (e) { console.log('downloadJSON ->', e.constructor.name + ': ' + e.message); }
"
```

For item 5, in a Vitest test: `deserialize({ __type: 'u8', value: '!!!' })` and `deserialize({ __type: 'bigint', value: 'abc' })`.

### Logs and Screenshots

```
exportDB -> ReferenceError: indexedDB is not defined
importDB -> ReferenceError: indexedDB is not defined
downloadJSON -> Error: downloadJSON is browser-only and cannot be called in a non-browser environment (e.g. during SSR).
```

### Environment Details

- `main` @ 4b90eca (`@aossie-org/idb-backup` 1.1.0)
- Node 24.22.1 (plain Node, no DOM); Vitest 5.0.1 + happy-dom 20.14.5 for item 5

### Impact

Low - Minor inconvenience

### Proposed fix

- README Features: replace "Graceful no-op guards" with an accurate statement, for example "Safe to import during SSR. Calling the APIs outside a browser throws a descriptive error; use `isBrowser()` to gate calls." Fix the line-53 wording to match.
- Add a small guard at the top of `exportDB` and `importDB` that checks `typeof indexedDB === 'undefined'` (not `isBrowser()`, so Web Workers keep working) and throws `exportDB requires IndexedDB, which is not available in this environment`. Add a test for it.
- Remove the stale "under active development" note, fix the happy-dom link and the ESLint version.
- Either wrap `atob`/`BigInt` failures in `deserialize` in a `TypeError` that names the tag, or make the JSDoc list the actual error types.
- Optional: add a short "Limitations" section that links the open issues about value types that don't survive a backup yet (#95, #124, #131, #132).

The README/`AGENTS.md` wording about `idb` is left out on purpose. It depends on the still-open question from #99 (drop or adopt the unused `idb` dependency) and is handled together with that.

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
