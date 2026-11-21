# [BUG]: CommonJS TypeScript consumers get the ESM type declarations, so `import` fails to type-check under `module: node16`

### Bug Description

`package.json` gives one `types` entry for both the `import` and the `require` condition:

```json
"type": "module",
"exports": {
  ".": {
    "types": "./dist/index.d.ts",
    "import": "./dist/index.js",
    "require": "./dist/index.cjs"
  }
}
```

The package is `"type": "module"`, so TypeScript treats `dist/index.d.ts` as an ES module declaration, even when it was resolved through `require`. A CommonJS consumer is therefore told the package is ESM-only, although a working CJS build (`dist/index.cjs`) ships right next to it. tsup already emits the matching CJS declaration file, `dist/index.d.cts`, but `package.json` never points to it.

This is the "Masquerading as ESM" problem that `@arethetypeswrong/cli` reports for `node16 (from CJS)`.

### Steps to Reproduce

1. `npm run build` on `main` (4b90eca), then put the package (`package.json` + `dist/`) into a consumer's `node_modules/@aossie-org/idb-backup`.
2. Consumer `package.json`: `{ "name": "c", "private": true }` (CommonJS, no `"type"`).
3. Consumer `tsconfig.json`: `{ "compilerOptions": { "module": "node16", "moduleResolution": "node16", "strict": true, "noEmit": true } }`
4. `index.ts`:
   ```ts
   import { exportDB } from '@aossie-org/idb-backup';
   export const f = exportDB;
   ```
5. Run `npx tsc -p .`

**Expected:** it type-checks. At runtime `require('@aossie-org/idb-backup')` works and returns all 9 exports.
**Actual:**

### Logs and Screenshots

```
index.ts(1,26): error TS1479: The current file is a CommonJS module whose imports will produce 'require' calls;
however, the referenced file is an ECMAScript module and cannot be imported with 'require'.
Consider writing a dynamic 'import("@aossie-org/idb-backup")' call instead.
```

`--traceResolution` shows that the module resolves to `dist/index.d.ts` instead of `dist/index.d.cts`.

### Environment Details

- `@aossie-org/idb-backup` 1.1.0 (`main` @ 4b90eca), built with the repo's tsup config
- TypeScript 5.9.3, Node 24.22.1, Linux
- Affects CJS consumers using `module`/`moduleResolution: node16`, and `nodenext` on TypeScript versions before 5.8. With TypeScript ≥ 5.8 and `nodenext`, the import is accepted because `require()` of ESM is allowed, but the types are still the ESM ones. ESM consumers and `moduleResolution: bundler` are unaffected.

### Impact

Low - Minor inconvenience

### Proposed fix

Give each condition its own types (checked locally: the TS1479 error goes away for CJS and resolves to `index.d.cts`, while ESM `node16` and `bundler` consumers still resolve to `index.d.ts` and type-check):

```json
"exports": {
  ".": {
    "import":  { "types": "./dist/index.d.ts",  "default": "./dist/index.js" },
    "require": { "types": "./dist/index.d.cts", "default": "./dist/index.cjs" }
  }
},
"sideEffects": false
```

`"sideEffects": false` is safe here because the entry point only re-exports functions and constants. It lets bundlers tree-shake unused exports. Optionally, add `npx publint` and `npx @arethetypeswrong/cli --pack` to CI so this doesn't regress.

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
