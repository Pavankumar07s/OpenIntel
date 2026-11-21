# [BUG]: Unused `idb` runtime dependency is installed for every consumer

### Bug Description

`package.json` on `main` (4b90eca) lists `"dependencies": { "idb": "^8.0.0" }`, so every project that installs `@aossie-org/idb-backup` also installs `idb`. Nothing in `src/` or `tests/` imports it (`grep -rn "from 'idb'" src tests` finds nothing), and the built `dist/index.js` / `dist/index.cjs` don't reference it. The core uses the native IndexedDB API directly through hand-written promise wrappers.

#99 raised exactly this and asked whether to adopt `idb` or drop it. Nobody answered on the issue. It was closed automatically when PR #100 was merged, because that PR's description said "Fixes #99", although PR #100 is about non-existent databases and doesn't touch the dependency. (#101 was a duplicate, closed by its author.) So the question is still open.

### Steps to Reproduce

```bash
npm ci
grep -rn "idb'" src tests            # no imports of idb
npm ls idb                           # idb@8.x installed as a production dependency
```

### Environment Details

- `main` @ 4b90eca, Node 24.22.1, npm 11.19.1

### Impact

Low - Minor inconvenience

### Proposed fix

- If maintainers agree to drop `idb` (the option that matches the current code): `npm uninstall idb`, so `package.json` and `package-lock.json` no longer list it. The README (Features "Lightweight" bullet, Tech Stack row) and `AGENTS.md` (project overview, and the "only runtime dependency is `idb`" convention) then need a small wording update to say the library has no runtime dependencies; that can follow in a separate docs PR once the direction is agreed.
- If maintainers would rather adopt `idb`, that is a larger refactor and needs its own issue.

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
