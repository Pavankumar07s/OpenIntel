# [BUG]: Release Drafter fails on merged fork PRs and closed Dependabot PRs, lists PRs twice, and uses options removed or deprecated in v7

## Bug Description

The Release Drafter setup (`.github/workflows/release-drafter.yml` and `.github/release-drafter.yml`) has four problems. Every repo created from the template inherits them.

1. **The `pull_request: types: [closed]` trigger fails, and it isn't needed.** Merging a PR into `main` already fires the `push` trigger, which updates the draft with a write token. The extra `pull_request` run fails in two common cases:
   - **A PR from a fork is merged.** Most contributors open PRs from forks. On fork PRs the `GITHUB_TOKEN` is read-only, so the action can't see the existing draft (drafts are only visible with write access). It then tries to create a new release and fails with `Resource not accessible by integration - https://docs.github.com/rest/releases/releases#create-a-release`.
   - **A Dependabot PR is closed without merging** (for example, superseded by a newer one). The config lookup fails with `Repo load failed. Config file not found with error 404. (target: AOSSIE-Org/.github:.github/release-drafter.yml)`.

   The run also starts for every PR that is closed without merging, where it does nothing useful. The v7 README example triggers only on `push`, with `permissions: contents: write, pull-requests: read`.

2. **`disable-autolabeler` is not an input of `release-drafter/release-drafter@v7`.** v7 moved the autolabeler into a separate action (`release-drafter/release-drafter/autolabeler@v7`). Every run logs `Unexpected input(s) 'disable-autolabeler'` and ignores the input. The template has no `autolabeler:` rules, so the line does nothing.

3. **The `👥 Contributors` category lists PRs a second time.** The category matches `first-time-contributor` / `repeat-contributor` / `org-member`. `sync-pr-labels.yml` adds one of these labels to each PR it processes. Release Drafter lets a PR match several categories by default, so those PRs show up under their type category and again under "👥 Contributors". The template body already has a `## Contributors` section with `$CONTRIBUTORS`, so the category is redundant.

4. **The config uses deprecated v7 syntax and never resolves anything except a patch bump.**
   - Every run logs 14 deprecation warnings: one for each `categories[*].labels` and one for `exclude-labels`. Each warning says: "This field will be removed in a future release."
   - No category sets `semver-increment`, so `$RESOLVED_VERSION` is always a patch bump. The run log shows `Resolved version increment: patch` even when feature PRs were merged.
   - `version-release.yml` renames the draft's name and tag to the `VERSION` value when it publishes, but it leaves the body unchanged. The body's `**Full Changelog**` link uses `v$RESOLVED_VERSION`, so it can point to the wrong tag.

## Steps to Reproduce

1. Open the Release Drafter run history: `gh run list -R AOSSIE-Org/Template-Repo --workflow release-drafter.yml -L 200`. In the retained history, 15 of 89 `pull_request` runs failed. None of the 102 `push` runs failed.
2. Merged fork PR, for example run `39254742973` (PR #166, head repo `kpj2006/Template-Repo`). The job token is `Contents: read`, and the run fails with `Resource not accessible by integration … create-a-release`.
3. Dependabot PR closed without merging, for example run `42592596251` (PR #171, closed with `mergedAt: null`). It fails with `Config file not found with error 404`.
4. The same failures still happen on the current `v7` tag (v7.11.0, SHA `d1a19e6`) in repos created from this template:
   - `AOSSIE-Org/MCP-Template` run `44030107842` (fork PR): `Resource not accessible by integration`
   - `AOSSIE-Org/PythonScripts` run `43838957871` (Dependabot PR #27 closed unmerged): config 404
5. Open the log of any successful run, for example the `push` run `42983922776`. It shows the `Unexpected input(s) 'disable-autolabeler'` warning, 14 `Use of deprecated …` warnings and `Resolved version increment: patch`.
6. Look at release notes published from this config in derived repos:
   - `AOSSIE-Org/IndexedDB-Import-Export` v1.0.1: PRs #61, #60, #50, #49, #48, #47, #45 and #46 appear in a type category and again under `## 👥 Contributors`.
   - `AOSSIE-Org/ThruBox-Client` v1.0.0: same pattern. Its body also ends with `**Full Changelog**: …/compare/v1.0.0...v1.0.1` on the release tagged `v1.0.0`.

## Logs and Screenshots

```
# Template-Repo run 39254742973 (merged fork PR #166)
GITHUB_TOKEN Permissions: Contents: read, PullRequests: read
##[error]Resource not accessible by integration - https://docs.github.com/rest/releases/releases#create-a-release

# Template-Repo run 42592596251 (Dependabot PR #171, closed without merge)
Config not found in AOSSIE-Org/Template-Repo, falling back to AOSSIE-Org/.github
##[error]Repo load failed. Config file not found with error 404. (target: AOSSIE-Org/.github:.github/release-drafter.yml)

# Every run
##[warning]Unexpected input(s) 'disable-autolabeler', valid inputs are ['config-name', 'token', 'name', 'tag', 'version', 'from', 'publish', 'latest', 'prerelease', 'prerelease-identifier', 'include-pre-releases', 'commitish', 'header', 'footer', 'dry-run', 'filter-by-range']
##[warning]Use of deprecated 'categories[*].label' or 'categories[*].labels' field detected on category "🚀 Features". Please migrate. This field will be removed in a future release. ...   (x13 categories)
##[warning]Use of deprecated 'exclude-labels' or 'exclude-paths' field detected. Please migrate. ...
Resolved version increment: patch
```

IndexedDB-Import-Export v1.0.1 release notes (excerpt):

```
## 🔧 Configuration
- ci: migrate npm publish to Trusted Publishing (OIDC) @Atharva0506 (#61)
- Remove CodeRabbit approval workflows @rohans02 (#60)
...
## 👥 Contributors
- ci: migrate npm publish to Trusted Publishing (OIDC) @Atharva0506 (#61)
- Remove CodeRabbit approval workflows @rohans02 (#60)
...
```

## Environment Details

- `AOSSIE-Org/Template-Repo` `main` @ `89ae62d`
- `release-drafter/release-drafter@v7`: failures seen on SHA `978bb0b` (before v7.11.0) and on SHA `d1a19e6` (v7.11.0, current `v7`)
- Files: `.github/workflows/release-drafter.yml` (lines 8-9, 13 and 24) and `.github/release-drafter.yml` (lines 4-58 and 74-79)
- Precedent: `AOSSIE-Org/IndexedDB-Import-Export` PR #96 already removed the `pull_request` trigger and the Contributors category and added minor/major resolution. It used the top-level `version-resolver:` block, which v7.11 also flags as deprecated.

## Impact

Low - Minor inconvenience

## Proposed fix

`.github/workflows/release-drafter.yml`:
- Remove the `pull_request: types: [closed]` trigger and keep only `push` to `main`/`master`.
- Change `pull-requests: write` to `pull-requests: read`.
- Remove `disable-autolabeler: false`. `sync-pr-labels.yml` already labels PRs, so no autolabeler job is needed.

`.github/release-drafter.yml`, migrated to the v7 schema:
- Move each category's `labels:` under `when: { labels: [...] }`.
- Remove the `👥 Contributors` category. `$CONTRIBUTORS` stays in the template.
- Replace `exclude-labels:` with a `type: pre-exclude` category.
- Add `semver-increment: minor` to the Features category, plus a `type: version-resolver` category with `semver-increment: major` for `breaking`/`major` labels. Don't add the deprecated top-level `version-resolver:` block.

Example (validated against the `drafter/schema.json` shipped in v7):

```yaml
categories:
  - type: pre-exclude
    when:
      labels: ['skip-changelog', 'no-changelog', 'duplicate', 'invalid', 'wontfix']
  - type: version-resolver
    semver-increment: major
    when:
      labels: ['breaking', 'major']
  - title: '🚀 Features'
    semver-increment: minor
    when:
      labels: ['feature', 'enhancement', 'feat']
  - title: '🐛 Bug Fixes'
    when:
      labels: ['fix', 'bugfix', 'bug']
  # ...remaining categories in the same `when:` form, without "👥 Contributors"
```

Workflow fixes only reach newly created repos, because template-sync skips `.github/workflows/**`. Existing repos would need to copy the workflow change by hand.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
