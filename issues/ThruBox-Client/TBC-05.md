# [BUG]: CommonJS TypeScript projects get ESM type declarations from the `require` export (TS1479)

## Bug Description

The package ships a CJS build (`dist/index.cjs`) and tsup already emits CJS declarations (`dist/index.d.cts`), but `package.json` points the `require` path at the ESM declarations:

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

The single top-level `"types"` condition applies to both `import` and `require`. Because the package has `"type": "module"`, TypeScript treats `index.d.ts` as an ES module. So a CommonJS TypeScript file that imports the package resolves ESM types but loads the CJS runtime file. Nothing in the package references `dist/index.d.cts`.

TypeScript then rejects the import with TS1479 under `"module": "node16"` or `"node18"`, and under `"nodenext"` before TypeScript 5.8. The runtime is fine: `require('@aossie-org/thrubox-client')` loads `index.cjs` and works. Only the types are wrong.

These setups are not affected: `"module": "nodenext"` or `"node20"` on TypeScript 5.8 or newer (which allows `require()` of ESM types), `moduleResolution: "node10"`, `"bundler"`, and ESM consumers.

## Steps to Reproduce

1. Create a CommonJS TypeScript project:
   ```bash
   mkdir cjs-app && cd cjs-app
   echo '{ "name": "cjs-app", "private": true, "type": "commonjs" }' > package.json
   npm i @aossie-org/thrubox-client@1.1.0 typescript@5.9.3
   cat > app.ts <<'TS'
   import { RelayClient, RelayError } from '@aossie-org/thrubox-client';
   const c = new RelayClient('http://localhost:8080');
   console.log(typeof c.send, typeof RelayError);
   TS
   ```
2. Type-check it:
   ```bash
   npx tsc --noEmit --strict --module node16 --moduleResolution node16 app.ts
   ```
3. See error TS1479.

The packaging checkers report the same thing:
- `npx @arethetypeswrong/cli --pack .` (run in this repo) shows `node16 (from CJS): 👺 Masquerading as ESM`.
- `npx publint` warns that `pkg.exports["."].types` "is interpreted as ESM when resolving with the "require" condition".

## Logs and Screenshots

```
app.ts(1,41): error TS1479: The current file is a CommonJS module whose imports will produce 'require' calls; however, the referenced file is an ECMAScript module and cannot be imported with 'require'. Consider writing a dynamic 'import("@aossie-org/thrubox-client")' call instead.
```

arethetypeswrong on the packed tarball:

```
👺 Import resolved to an ESM type declaration file, but a CommonJS JavaScript file.

node10              🟢
node16 (from CJS)   👺 Masquerading as ESM
node16 (from ESM)   🟢 (ESM)
bundler             🟢
```

Results of `tsc --noEmit` on the CJS `app.ts` above, against the current package:

| `module` / `moduleResolution` | TypeScript 5.9.3 | TypeScript 5.6 |
|---|---|---|
| `node16` / `node16` | TS1479 | TS1479 |
| `node18` / `nodenext` | TS1479 | n/a |
| `nodenext` / `nodenext` | OK | TS1479 |
| `commonjs` / `node10` | OK | OK |
| `preserve` / `bundler` | OK | OK |

## Environment Details

- `@aossie-org/thrubox-client` 1.1.0. Both the npm release and `main` at `9afc707` have the same `exports` map.
- TypeScript 5.9.3 (also 5.6.1-rc for the `nodenext` row)
- Node.js v24.22.1, Linux

## Impact

Low - Minor inconvenience

## Proposed fix

Give each condition its own `types` entry, pointing at the declarations tsup already builds. This needs no change to `tsup.config.ts`:

```json
"exports": {
  ".": {
    "import": { "types": "./dist/index.d.ts", "default": "./dist/index.js" },
    "require": { "types": "./dist/index.d.cts", "default": "./dist/index.cjs" }
  }
}
```

I tested this locally against a packed tarball:
- every row in the table above type-checks;
- the compiled CJS output runs;
- arethetypeswrong reports "No problems found";
- publint shows no warnings.

Running `attw --pack` and `publint` in CI would stop this from coming back.
