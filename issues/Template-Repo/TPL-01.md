# [BUG]: CodeQL workflow never analyzes code: `set-codeql-language-matrix@v1` has no `matrix` output, so the Analyze job is always skipped

## Bug Description

`.github/workflows/codeql.yml` builds the Analyze matrix from `steps.set-matrix.outputs.matrix` (line 29), which comes from `advanced-security/set-codeql-language-matrix@v1` (line 35).

The upstream `v1` tag still points to **v1.3.0** (`9f9ba17a`). At that version, `action.yml` declares only a `languages` output, and `main.py` only writes `languages=...`. The `matrix` output (with `build-mode`) arrived in v1.4.0, released 2026-08-05, and `v1` was never moved to it. Upstream issue advanced-security/set-codeql-language-matrix#20 ("The Current release doesn't support matrix") describes the same gap.

Here is what happens on every run:

1. `steps.set-matrix.outputs.matrix` is empty, so the job output falls back to `'[]'` (line 29).
2. The `analyze` condition `needs.create-matrix.outputs.matrix != '[]'` (line 55) is false.
3. `Analyze` is **skipped**, and the workflow still reports **success**.

So this workflow has never run CodeQL, either here or in any repo created from the template. In this repo, 0 of 182 "CodeQL Security Scan" runs executed the Analyze job. Runs from 2026-03-08 to 2026-05-26 failed before Analyze started. #121 ("Codeql fix") then added the `|| '[]'` fallback, which turned those failures into green skips.

## Steps to Reproduce

1. Open any recent "CodeQL Security Scan" run, for example the scheduled run https://github.com/AOSSIE-Org/Template-Repo/actions/runs/43792320555
2. The jobs are `create-matrix`: success and `Analyze (${{ matrix.language }})`: skipped. The job name is never expanded because the matrix never existed.
3. The `create-matrix` log shows that languages were detected but no `matrix` output was produced (see Logs below).
4. Check which version `@v1` resolves to and which outputs it has:
   ```bash
   git ls-remote https://github.com/advanced-security/set-codeql-language-matrix 'refs/tags/v1^{}' 'refs/tags/v1.3.0'
   # 9f9ba17a61d34c97baaf17e96276389a0cd73392  refs/tags/v1.3.0
   # 9f9ba17a61d34c97baaf17e96276389a0cd73392  refs/tags/v1^{}

   gh api "repos/advanced-security/set-codeql-language-matrix/contents/action.yml?ref=v1" --jq .content | base64 -d | sed -n '/^outputs:/,/^runs:/p'
   # outputs:
   #   languages:
   #     description: 'List of languages that will set the job matrix'
   # runs:
   ```

## Logs and Screenshots

Template-Repo, run 43792320555, job `create-matrix`:
```
Download action repository 'advanced-security/set-codeql-language-matrix@v1' (SHA:9f9ba17a61d34c97baaf17e96276389a0cd73392)
After mapping: ['actions', 'javascript']
Intersection: ['actions', 'javascript']
languages=['actions', 'javascript']
Evaluate and set job outputs
Set output 'matrix'
```
Then: `Analyze (${{ matrix.language }})` was skipped.

Derived repos behave the same way (scheduled runs on 2026-11-16):
- ThruBox-Server, run 43792378937: `languages=['go', 'javascript', 'actions']`, Analyze skipped. This is a Go repo that gets no CodeQL scan.
- ThruBox-Client, run 43792386100: `languages=['javascript', 'actions']`, Analyze skipped
- IndexedDB-Import-Export, run 43792369522: `languages=['javascript', 'actions']`, Analyze skipped

## Environment Details

- `main` at `89ae62d`
- `advanced-security/set-codeql-language-matrix@v1` resolves to `9f9ba17a` (v1.3.0). The latest release is v1.6.0 (`038e12c9`).
- GitHub-hosted `ubuntu-latest` runners

## Impact

High - Major feature is broken. CodeQL code scanning is silently disabled in this repo and in every repo generated from it, while the check shows green.

## Proposed fix

Pin a release that has the `matrix` output. Upstream's README now recommends pinning a full, immutable release tag. Then adapt the workflow to what that release emits:

```yaml
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}   # drop the `|| '[]'` fallback so a missing output fails visibly
    steps:
      - id: set-matrix
        uses: advanced-security/set-codeql-language-matrix@038e12c9474b0c253f122180eb943bee2e10a24a # v1.6.0
        with:
          access-token: ${{ secrets.GITHUB_TOKEN }}
          endpoint: ${{ github.event.repository.languages_url }}
          standard-language-names: 'true'
```

Pinning the new version alone is not enough. These follow-up changes are also needed:
- **`standard-language-names: 'true'`**: by default v1.6.0 emits `javascript` and `java`. The workflow's Setup Node, Setup Java, Gradle and Maven steps check for `javascript-typescript` and `java-kotlin`, so without this input they would never run.
- **Go build mode**: v1.6.0 sets `build-mode: manual` for Go (and Swift and Kotlin). The workflow has no Go or Swift build step, so a Go repo such as ThruBox-Server would go from "skipped" to a failing Analyze job. Use autobuild for those languages, for example `build-mode: ${{ (matrix.language == 'go' || matrix.language == 'swift') && 'autobuild' || matrix.build-mode }}`, or add a build step.
- **Empty-language guard**: when no language is detected, v1.6.0 emits `{"include": []}`, not `[]`. The `!= '[]'` check on line 55 therefore no longer matches. Use something like `fromJSON(needs.create-matrix.outputs.matrix).include[0] != null`.
- Remove the stale comment for the `build-mode-manual-override` input at line 45, or keep it. It only works once the action is on v1.4.0 or later.

A smaller alternative that stays on `@v1`: read `steps.set-matrix.outputs.languages`, use `matrix: { language: ${{ fromJSON(...) }} }`, and omit `build-mode` so that CodeQL picks its defaults. This output is marked deprecated upstream, though.

Note: template-sync skips `.github/workflows/**`, so repos that were already generated (ThruBox-Server, ThruBox-Client, IndexedDB-Import-Export, …) need to apply the same change themselves.
