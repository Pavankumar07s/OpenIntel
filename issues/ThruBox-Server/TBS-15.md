# [BUG]: Stale workflow auto-closes unreviewed Dependabot PRs (incl. golang/alpine base-image bumps), and Dependabot then skips those versions

## Bug Description

`.github/workflows/stale.yml` treats Dependabot PRs like contributor PRs. It marks them `Stale` after 14 days without activity and closes them 7 days later. The only exemptions are `exempt-pr-labels: 'Keep Open,Work In Progress,WIP'`, and none of those labels exists in the repo.

Dependabot does add a `dependencies` label, but `sync-pr-labels.yml` removes it again. `dependencies` is one of its "managed" file-based labels, and it only matches `package.json`, `go.mod`, `requirements.txt` and similar files. So a PR that only touches the `Dockerfile` or `.github/workflows/*` loses the label a few seconds after it is opened. The same step also labels Dependabot `first-time-contributor`.

When the stale bot closes a Dependabot PR, Dependabot treats it as a manual close and replies *"OK, I won't notify you again about this release, but will get in touch when a new version is available."* That version is then skipped for good.

Dependabot PRs closed this way by `github-actions[bot]`:

| PR | Update | Closed |
|----|--------|--------|
| #16 | alpine 3.20 → 3.24 | 2026-08-30 |
| #19 | ossf/scorecard-action 2.4.3 → 2.4.4 | 2026-08-19 |
| #23 | actions/stale 10 → 11 | 2026-08-25 |
| #47 | golang 1.23.12-alpine → 1.27.1-alpine | 2026-09-29 |
| #71 | golang 1.23.12-alpine → 1.27.3-alpine | 2026-11-17 |

Since #16 closed, Dependabot has not opened another Alpine PR. The `Dockerfile` on `main` is still `golang:1.23.12-alpine` / `alpine:3.20`.

PRs #77 (go-sqlite3), #78 (harden-runner) and #79 (docker/login-action) have had no activity since 2026-11-09. They will be marked stale on about 2026-11-23 and closed about a week later.

## Steps to Reproduce

1. Open the timeline of #71 (or #16, #47) and look at the label and close events:
   ```
   gh api "repos/AOSSIE-Org/ThruBox-Server/issues/71/timeline?per_page=100" \
     --jq '.[] | select(.event|test("labeled|unlabeled|closed")) | "\(.created_at) \(.event) \(.actor.login) \(.label.name // "")"'
   ```
2. You will see these events, in this order:
   - `dependabot[bot]` labels the PR `dependencies` and `docker`.
   - About 8 s later, `github-actions[bot]` removes `dependencies` and adds `first-time-contributor`.
   - `github-actions[bot]` adds `Stale` 14 days later.
   - `github-actions[bot]` closes the PR 7 days after that.
3. Compare the labels each PR had when it was closed with `exempt-pr-labels` in `stale.yml`. None of them is exempt:
   ```
   #16  labels=Stale,docker,first-time-contributor,needs-review,no-issue-linked,size/XS
   #19  labels=Stale,ci-cd,configuration,first-time-contributor,github-actions,needs-review,size/XS
   #47  labels=Stale,docker,first-time-contributor,needs-review,no-issue-linked,size/XS
   #71  labels=Stale,docker,first-time-contributor,needs-review,no-issue-linked,size/XS
   #78  labels=ci-cd,configuration,first-time-contributor,github-actions,needs-review,size/XS   (open)
   ```

**Expected:** dependency-update PRs stay open until a maintainer merges or closes them. Dependabot already limits open PRs to 5 per ecosystem and closes superseded PRs on its own.

**Actual:** they get the human-contributor "please push an update" message and are auto-closed. Dependabot then stops proposing that version.

## Logs and Screenshots

Comment on #16 after the stale bot closed it (2026-08-30T01:27:12Z):
```
dependabot[bot]: OK, I won't notify you again about this release, but will get in touch when a new version is available.
```
Dependabot posted the same comment on #19, #23 and #47.

## Environment Details

- Repo `main` at f026f22
- `.github/workflows/stale.yml` (`actions/stale@v10`, `days-before-pr-stale: 14`, `days-before-pr-close: 7`)
- `.github/workflows/sync-pr-labels.yml` (file-based label step, `labelMappings.dependencies`)
- `.github/dependabot.yml` (labels `dependencies` + `docker` / `github-actions` / `go`)

## Impact

Low - Minor inconvenience. Maintainers can still merge these PRs while they are open. But unreviewed base-image and action updates are dropped silently, and Dependabot does not offer that version again.

## Proposed fix

1. `sync-pr-labels.yml`: stop the file-based step from removing `dependencies`, at least when `pull_request.user.type === 'Bot'` or the author is `dependabot[bot]`. Also skip the contributor-label step for bots so they are not tagged `first-time-contributor`.
2. `stale.yml`: add `dependencies` to `exempt-pr-labels`. Or, if auto-closing PRs is not wanted at all, set `days-before-pr-stale: -1`.
3. Optionally reopen #16, or bump `alpine` by hand, so the skipped Alpine update is not lost.

The same `stale.yml` settings come from `AOSSIE-Org/Template-Repo`. Workflows are not overwritten by template sync (see `.templatesyncignore`), so a local change here will stay.
