# [BUG]: Dependency Review `allow-licenses` rejects common permissive licenses (0BSD, BlueOak-1.0.0, Unicode-3.0, Zlib) but accepts GPL-2.0-only

## Bug Description

`.github/workflows/dependency-review-action.yml` runs `actions/dependency-review-action@v5` with an `allow-licenses` list (lines 74-97), `warn-only: false` (line 124) and `fail-on-scopes: runtime` (line 68). For every runtime dependency a PR adds, the action checks the license with `spdx-satisfies(license, allowList)`. If the license does not match, the job fails with `Dependency review detected incompatible licenses.` (`src/licenses.ts`, `src/main.ts` `printLicensesBlock`).

The list has two problems.

**1. Several common OSI-approved permissive licenses are missing,** so PRs that add ordinary dependencies fail. These are the license strings GitHub's dependency graph assigns, and the action reads them as-is:

| License (as reported by GitHub) | Example packages |
|---|---|
| `0BSD` | npm `tslib`, a runtime dependency of `rxjs` 7, Angular packages and many TypeScript libraries |
| `BlueOak-1.0.0` | npm `glob` >= 11.1.0, `minimatch` >= 10.1.0, `lru-cache` >= 11.2.3, every `path-scurry` release, `jackspeak` >= 2 (`glob` 10.x already depends on `path-scurry` 1.x and `jackspeak` 3.x) |
| `(MIT OR Apache-2.0) AND Unicode-3.0` (older versions: `... AND Unicode-DFS-2016`) | crates.io `unicode-ident`, used by `proc-macro2`/`syn`, so it ends up in most Rust projects that use `serde_derive` |
| `MIT AND Zlib` | npm `pako` 1.x |
| `MIT-0`, `BSL-1.0` | less common, e.g. npm `@csstools/postcss-*` plugins (`MIT-0`), crate `xxhash-rust` (`BSL-1.0`) |

The dependency graph marks Cargo dependencies as `scope=runtime`, even build-time crates such as `proc-macro2`, so `fail-on-scopes: runtime` does not skip them.

**2. `GPL-2.0-only` is allowed (line 82),** but this repository and the repos created from it are licensed GPL-3.0 (`LICENSE`, `README.md` line 274). The FSF lists GPL-2.0-only as incompatible with GPL-3.0, so the gate passes the one GPL variant that cannot be combined with the project.

Removing only the `GPL-2.0-only` line does **not** fix this. `spdx-satisfies` treats `GPL-2.0-or-later` in an allow-list as also matching `GPL-2.0-only`. To reject `GPL-2.0-only`, both `GPL-2.0-*` entries have to go. Dependencies licensed `GPL-2.0-or-later` still pass after that, because `GPL-3.0-only` / `GPL-3.0-or-later` stay on the list.

The workflow file is byte-identical in ThruBox-Client, ThruBox-Server and IndexedDB-Import-Export. None of them has failed so far: their recent Dependency Review runs only changed dev dependencies or GitHub Actions. For example, ThruBox-Client PR #78 adds `minimatch@10.2.6` (`BlueOak-1.0.0`), but only as a dev dependency, so the check skipped it.

## Steps to Reproduce

1. Run the same license check the action runs. `spdx-satisfies@6.0.0` is the version `dependency-review-action@v5.0.1` locks:
   ```sh
   mkdir dr-licenses && cd dr-licenses && npm init -y >/dev/null
   npm i spdx-satisfies@6.0.0 spdx-license-ids@3.0.23
   cat > check.js <<'EOF'
   const satisfies = require('spdx-satisfies');
   // allow-licenses from .github/workflows/dependency-review-action.yml (lines 74-97)
   const allow = ['MIT','Apache-2.0','BSD-2-Clause','BSD-3-Clause','ISC','CC0-1.0','Unlicense',
     'GPL-2.0-only','GPL-2.0-or-later','GPL-3.0-only','GPL-3.0-or-later',
     'LGPL-2.0-only','LGPL-2.0-or-later','LGPL-2.1-only','LGPL-2.1-or-later',
     'LGPL-3.0-only','LGPL-3.0-or-later','AGPL-3.0-only','AGPL-3.0-or-later',
     'MPL-2.0','EUPL-1.2','Python-2.0','PSF-2.0'];
   const withoutGpl2Only = allow.filter(l => l !== 'GPL-2.0-only');
   for (const l of ['0BSD', 'BlueOak-1.0.0', '(MIT OR Apache-2.0) AND Unicode-3.0',
                    'MIT AND Zlib', 'MIT-0', 'BSL-1.0', 'GPL-2.0-only'])
     console.log(l.padEnd(37), satisfies(l, allow) ? 'allowed ' : 'REJECTED',
                 '| list minus GPL-2.0-only:', satisfies(l, withoutGpl2Only) ? 'allowed' : 'REJECTED');
   EOF
   node check.js
   ```
2. Confirm that GitHub's dependency graph reports these exact license strings, since those are what the action receives:
   ```sh
   # tslib -> 0BSD, lru-cache 11 / jackspeak / path-scurry -> BlueOak-1.0.0, pako -> MIT AND Zlib
   gh api repos/ReactiveX/rxjs/dependency-graph/sbom \
     --jq '.sbom.packages[] | select(.name|test("^(tslib|lru-cache|jackspeak|path-scurry|pako)$")) | "\(.name) \(.versionInfo) \(.licenseConcluded)"'
   # unicode-ident -> (MIT OR Apache-2.0) AND Unicode-3.0
   gh api repos/BurntSushi/ripgrep/dependency-graph/sbom \
     --jq '.sbom.packages[] | select(.name=="unicode-ident") | "\(.name) \(.versionInfo) \(.licenseConcluded)"'
   # The compare API the action calls, for ThruBox-Client PR #78:
   gh api repos/AOSSIE-Org/ThruBox-Client/dependency-graph/compare/9afc7070e497c7c679427a4fbda194b2c8ae7a15...6a5264da585460a037c28c48825c2184ac92c9dd \
     --jq '.[] | "\(.change_type) \(.name)@\(.version) scope=\(.scope) license=\(.license)"'
   ```
3. Given the above, a PR in a repo using this workflow that adds a runtime dependency pulling in any of these packages (for example `rxjs`) will fail the "Dependency & License Review" job. A PR that adds a `GPL-2.0-only` runtime dependency will pass.

## Logs and Screenshots

Output of `node check.js` (Node 24):
```
0BSD                                  REJECTED | list minus GPL-2.0-only: REJECTED
BlueOak-1.0.0                         REJECTED | list minus GPL-2.0-only: REJECTED
(MIT OR Apache-2.0) AND Unicode-3.0   REJECTED | list minus GPL-2.0-only: REJECTED
MIT AND Zlib                          REJECTED | list minus GPL-2.0-only: REJECTED
MIT-0                                 REJECTED | list minus GPL-2.0-only: REJECTED
BSL-1.0                               REJECTED | list minus GPL-2.0-only: REJECTED
GPL-2.0-only                          allowed  | list minus GPL-2.0-only: allowed
```

Dependency graph data (excerpt):
```
tslib 2.8.1 0BSD
lru-cache 11.5.3 BlueOak-1.0.0
jackspeak 3.4.3 BlueOak-1.0.0
path-scurry 1.11.1 BlueOak-1.0.0
pako 1.0.11 MIT AND Zlib
unicode-ident 1.0.24 (MIT OR Apache-2.0) AND Unicode-3.0
added minimatch@10.2.6 scope=development license=BlueOak-1.0.0   <- ThruBox-Client PR #78
```

When a rejected license is in runtime scope, the action logs `The following dependencies have incompatible licenses:` followed by `Dependency review detected incompatible licenses.` and fails the job (`warn-only: false`).

## Environment Details

- Workflow: `.github/workflows/dependency-review-action.yml` on `main` (`89ae62d`). The same file is in ThruBox-Client, ThruBox-Server and IndexedDB-Import-Export.
- Action: `actions/dependency-review-action@v5` (resolves to v5.0.1, `bbc44d1`), which depends on `spdx-satisfies` 6.0.0 and `spdx-license-ids` 3.0.23
- Project license: GPL-3.0 (template and all three derived repos)
- `main` has no branch protection or rulesets, so the failing check shows a red X on the PR but does not block the merge. The file's header comment says it "Blocks merges", which is only true if the check is made required.

## Impact

Low - Minor inconvenience

## Proposed fix

In `.github/workflows/dependency-review-action.yml`:

1. Add the missing permissive licenses to `allow-licenses`: `0BSD`, `BlueOak-1.0.0`, `Unicode-3.0`, `Unicode-DFS-2016`, `Zlib`, `MIT-0`, `BSL-1.0`.
2. Remove **both** `GPL-2.0-only` and `GPL-2.0-or-later`. `GPL-2.0-or-later` dependencies still pass through the `GPL-3.0-*` entries. With the current `spdx-satisfies`, keeping `GPL-2.0-or-later` still lets `GPL-2.0-only` through.
3. Add a comment above the list saying it must be kept compatible with the repository's `LICENSE`, so projects that switch to MIT or Apache-2.0 know to remove the copyleft entries.

`.templatesyncignore` excludes `.github/workflows/**`, so the three existing repos would need the same change applied by hand.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
