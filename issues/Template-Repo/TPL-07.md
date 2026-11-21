# [BUG]: sync-pr-labels mislabels PRs: copies labels from any `#N` in the body, strips Dependabot's `dependencies`, marks bots as first-time contributors, misses the maintain role

## Bug Description

`.github/workflows/sync-pr-labels.yml` (main @ `89ae62d`) applies wrong labels in four ways. Every repo created from this template inherits the workflow, and template-sync does not update workflows afterwards.

1. **Any `#N` counts as a linked issue (lines 43-46).** When no closing keyword is found, the fallback `/#(\d+)/g` takes the first `#N` anywhere in the body and copies that issue's (or PR's) labels onto the PR. Dependabot release notes link upstream PRs as `#N`, so a Dependabot PR picks up labels from an unrelated local issue or PR, and `no-issue-linked` gets removed. 0 of the 43 Dependabot PRs in this repo have `no-issue-linked`. The same thing happens for any incidental `#N`, such as "changed the color to #333". The target is never checked to be an issue rather than a PR.
2. **File patterns are substring matches (line 189, `file.includes(pattern)`).** `.js` matches every `.json` file (`package-lock.json`, `checklist-status.json` get `javascript`). `test/` matches `latest/`, and `.go` matches `.goreleaser.yaml` (gets `backend`). `dependencies` only matches a few manifest names (`package.json`, `go.mod`, ...). The "remove stale managed labels" step (lines 206-228) then removes the `dependencies` label that Dependabot adds through `dependabot.yml` from GitHub Actions, Docker and lockfile-only updates. In this repo 0 of 43 Dependabot PRs still have `dependencies`, so they never show up in release-drafter's "⬆️ Dependencies" section.
3. **Bots get contributor labels (lines 348-380).** `repos.listCommits({ author: 'dependabot[bot]' })` returns `[]` even though `main` has many commits by `dependabot[bot]`. So every Dependabot PR is labeled `first-time-contributor`: 43/43 here, and 80/80, 47/47 and 51/51 in IndexedDB-Import-Export, ThruBox-Server and ThruBox-Client. Release-drafter then also lists them under "👥 Contributors".
4. **The maintain role never gets `org-member` (line 365).** The code checks `['admin', 'maintain'].includes(data.permission)`, but `permission` is the legacy field. GitHub docs: "the maintain role is mapped to write and the triage role is mapped to read". Only admins match, so the `'maintain'` branch never runs. PR #53 meant `org-member` for maintainers. Admins do get the label (the same code applied `org-member` to an admin's PR in AOSSIE-Org/Gitcord-Test #6). Line 366 pushes `'org-Member'` while setup-labels and release-drafter use `org-member`. GitHub label names are case-insensitive, so this works today, but it should use the same constant.

## Steps to Reproduce

Observed on open Dependabot PR #178 (bump `google/osv-scanner-action` 2.7.0 → 2.7.1, one workflow file changed):

1. The body has no closing keyword. The first `#N` is `#145`, a link to an upstream google/osv-scanner-action PR in the release notes.
2. The workflow fetched this repo's #145 (an unrelated PR, "Update copyright year from 2025 to 2026") and copied its labels.
3. In the same run it removed Dependabot's `dependencies` label, and replaced the copied `repeat-contributor` with `first-time-contributor`.

Local check of the matching logic (step scripts copied from the workflow and run with Node against a mocked `github`):

```
"...osv-scanner-action/issues/145">#145</a> from renovate"  -> issue "145"
"Changed the button color to #333"                          -> issue "333"
["package-lock.json"]                     -> add javascript,configuration
["checklist-status.json"]                 -> add javascript,configuration
["src/latest/handler.go"]                 -> add backend,tests
[".goreleaser.yaml"]                      -> add backend,configuration
[".github/workflows/stale.yml"] (has dependencies) -> remove dependencies | add configuration,ci-cd
["Dockerfile"]                  (has dependencies) -> remove dependencies
```

Author filter check:

```
gh api "repos/AOSSIE-Org/Template-Repo/commits?author=dependabot%5Bbot%5D" --jq length   # 0
gh api "repos/AOSSIE-Org/Template-Repo/commits?author=49699333%2Bdependabot%5Bbot%5D@users.noreply.github.com" --jq length   # 30
```

## Logs and Screenshots

Job log of run 43838695066 (PR #178):

```
Applying issue-based labels: documentation, no-issue-linked, size/XS, repeat-contributor, needs-review
Removed "no-issue-linked" label
Removing stale file-based labels: documentation, dependencies
Applying file-based labels: configuration, ci-cd
Applying contributor-based labels: first-time-contributor
```

PR #178 timeline: `labeled dependencies by dependabot[bot]` (09:19:24), then `unlabeled dependencies by github-actions[bot]` (09:19:36).
Derived repo, AOSSIE-Org/ThruBox-Client #78 (eslint bump, only `package-lock.json` changed): `dependencies` removed, `javascript` and `first-time-contributor` added.

## Environment Details

- Workflow: `.github/workflows/sync-pr-labels.yml` on `main` @ `89ae62d`, `actions/github-script@v9`, trigger `pull_request_target`
- Seen in AOSSIE-Org/Template-Repo and the repos created from it (IndexedDB-Import-Export, ThruBox-Server, ThruBox-Client)

## Impact

Low - Minor inconvenience (wrong triage labels, and wrong release-drafter categories for Dependabot PRs. Nothing fails.)

## Proposed fix

- Treat only closing keywords (`fix|close|resolve` variants, optionally followed by `:`) as an issue link. Drop the bare `#N` fallback, or limit it to the PR template's "Addressed Issues" line. Skip targets where `issue.data.pull_request` is set.
- Match file labels by extension (`path.extname`), basename and directory prefix instead of `includes`. Add lockfiles (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `go.sum`, `poetry.lock`, `Cargo.lock`) to `dependencies`. Do not remove `dependencies` or `github-actions` from Dependabot PRs. Better still, only remove labels this workflow added itself.
- Skip contributor labels when `pull_request.user.type === 'Bot'`.
- Check `data.role_name` (`admin` / `maintain`) instead of `data.permission`, and use one lowercase `org-member` constant everywhere.

## Code of Conduct

- [ ] I have joined the Discord server and will post updates there
- [ ] I have searched existing issues to avoid duplicates
