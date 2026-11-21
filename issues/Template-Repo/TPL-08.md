# [BUG]: Template Sync PRs reset per-project files (VERSION, CONTRIBUTING.md, MAINTAINERS.md, checklist score) because `.templatesyncignore` only excludes README/LICENSE/workflows

## Bug Description

`.github/workflows/template-sync.yml` uses `AndreasAugustin/actions-template-sync@v2` with its default pull parameters. The default is `--allow-unrelated-histories --squash --strategy=recursive -X theirs` (`src/sync_template.sh` line 61 at `v2`, and the action README). A repo created from a template has no history in common with it, and each sync is squashed, so every sync merges against an empty base. With `-X theirs`, **any file that differs from the template comes back as the template's version, even if the template never changed it**. Files the project deleted are added back too.

`.templatesyncignore` only excludes `README.md`, `LICENSE` and `.github/workflows/**`. Every file that each project is expected to fill in or own is therefore reverted in the monthly "upstream merge template repository" PR:

- `VERSION` goes back to `1.0.0`. Merging the PR pushes a `VERSION` change to `main`, which triggers `version-release.yml`.
- `checklist-status.json` / `BestPracticesChecklist.md` go back to the template's 0% score and unticked boxes.
- `MAINTAINERS.md` gets the `TODO` / `@username` rows back in place of the real maintainers.
- `CONTRIBUTING.md` and `PRIVACY.md` get their `TODO: Project Name` placeholders back.
- `.gitignore` is replaced by the template's (LaTeX) `.gitignore`, so the project's own entries are dropped.
- Files a project removed (`.github/initial-issues.json`, `BestPracticesChecklist.md`, `PRIVACY.md`, `skills/**`, `.goreleaser.yaml`, …) are added back.

You only see this in the PR diff. The sync log reports `Automatic merge went well` and no conflicts.

### Evidence from existing repos

- **AOSSIE-Org/MCP-Template** (created 2026-08-27, after `.github/workflows/**` was added to the ignore file) syncs successfully every month. Its open sync PR AOSSIE-Org/MCP-Template#27 would change `# Contributing to AOSSIE MCP Template` back to `# Contributing to TODO: Project Name`. It would also delete the repo's own `.gitignore` entries (`src/generated/runtime.ts`, `catalog/dist/`) and add back `BestPracticesChecklist.md`, `checklist-status.json` (0%), `PRIVACY.md`, `.github/initial-issues.json` and `skills/**` (+1946/−127). The two earlier sync PRs (#7, #19) were closed as stale without being merged. That repo has already added `.coderabbit.yaml` and `.github/dependabot.yml` to its own ignore file by hand.
- **AOSSIE-Org/IndexedDB-Import-Export** run 41257131894 (2026-11-01) built a 29-file sync commit (`1265 insertions(+), 448 deletions(-)`). It includes `VERSION`, `checklist-status.json`, `BestPracticesChecklist.md`, `MAINTAINERS.md`, `CONTRIBUTING.md` and `.gitignore`. When I replay that exact merge locally I get the same 29-file stat. In that result `VERSION` goes from `1.1.0` to `1.0.0`, the score goes from 65% to 0%, the Mentors/Maintainers tables become `TODO` rows, and `node_modules/`, `dist/` and `.env` are removed from `.gitignore`. That push was only rejected because of the separate `.github/workflows` permission error (see note below), so no PR was opened.

Note: IndexedDB-Import-Export, ThruBox-Server and ThruBox-Client fail every monthly sync with `refusing to allow a GitHub App to create or update workflow .github/workflows/anti-slop-pr.yaml without workflows permission`. They were created before `.github/workflows/**` was uncommented in `.templatesyncignore` (ca52ca1, 2026-08-13), and the action never syncs `.templatesyncignore` itself. Those repos need to update their own ignore file. Once they do, they will start getting the same reverting PRs as MCP-Template.

## Steps to Reproduce

These steps simulate the action's sequence from `sync_template.sh` at `v2`: pull, restore the ignore file, `git add .`, reset the ignored paths, clean, commit.

```bash
git clone https://github.com/AOSSIE-Org/Template-Repo tpl
mkdir derived && (cd tpl && git archive HEAD) | tar -x -C derived
cd derived && git init -q -b main && git add -A && git commit -qm "Initial commit"   # = "Use this template"

# normal per-project edits
echo "1.4.2" > VERSION
sed -i '1s/.*/# MyProject Maintainers/' MAINTAINERS.md
printf '\n## Project-specific rule\nRun npm test before opening a PR.\n' >> CONTRIBUTING.md
git rm -q PRIVACY.md && git commit -qam "project customisations"

# any later commit in the template (touching none of the files above)
(cd ../tpl && echo "# tweak" >> socket.yml && git commit -qam "template tweak")

# what actions-template-sync@v2 does
git config pull.rebase false
git checkout -q -b chore/template_sync
git pull ../tpl --allow-unrelated-histories --squash --strategy=recursive -X theirs main
git reset -q .templatesyncignore; git checkout -- .templatesyncignore
git add .
sed '/^[[:blank:]]*#/d;s/#.*//' .templatesyncignore | awk NF | xargs -r git reset -q --
git clean -dfq; git checkout -- .
git commit -q -m "chore(template): merge template changes :up:"
git diff --stat main HEAD; echo "VERSION=$(cat VERSION)"; head -1 MAINTAINERS.md
```

**Expected:** the sync PR contains only the template's change (`socket.yml`).

**Actual:**

```
Auto-merging CONTRIBUTING.md
Auto-merging MAINTAINERS.md
Auto-merging VERSION
Auto-merging socket.yml
Automatic merge went well; stopped before committing as requested
 CONTRIBUTING.md |   3 ---
 MAINTAINERS.md  |   2 +-
 PRIVACY.md      | 115 ++++++++++++
 VERSION         |   2 +-
 socket.yml      |   1 +
 5 files changed, 118 insertions(+), 5 deletions(-)
VERSION=1.0.0
# Maintainers, Mentors and Ideators
```

## Logs and Screenshots

- AOSSIE-Org/MCP-Template#27 (open sync PR), plus closed sync PRs #7 and #19
- IndexedDB-Import-Export job log, run 41257131894: `Auto-merging … BestPracticesChecklist.md, CONTRIBUTING.md, MAINTAINERS.md, PRIVACY.md, README.md, VERSION, checklist-status.json` → `Automatic merge went well` → `29 files changed, 1265 insertions(+), 448 deletions(-)`
- Action default: https://github.com/AndreasAugustin/actions-template-sync/blob/v2/src/sync_template.sh (line 61, `GIT_REMOTE_PULL_PARAMS` default)

## Environment Details

- Template-Repo `main` @ 89ae62d (`.templatesyncignore` lines 8–12, `.github/workflows/template-sync.yml`)
- `AndreasAugustin/actions-template-sync@v2` (v2.6.0, commit 6e1d008), with `git_remote_pull_params` not set
- git 2.56.0 on the runner, git 2.51.0 locally (same result)

## Impact

Medium - Feature works but has issues. The sync runs, but every PR it opens would undo per-project work. Maintainers can't safely merge these PRs, so the sync can't deliver the shared config updates it exists for.

## Proposed fix

Exclude the files that each project owns in `.templatesyncignore`:

```
VERSION
CHANGELOG.md
CONTRIBUTING.md
MAINTAINERS.md
PRIVACY.md
BestPracticesChecklist.md
checklist-status.json
.github/initial-issues.json
public/**
```

I tested this with the reproduction above: the sync commit then contains only the template's own change.

An alternative is an allow-list: `*` followed by `:!<file>` lines for the files that really are meant to be shared, such as `.coderabbit.yaml` (the syntax is already described in the file's comments). This also protects `.gitignore`, `dangerfile.js`, `.editorconfig` and any other file a project changes. Ignoring `.coderabbit.yaml` is not proposed here, because propagating it is what the sync was added for (#74, #76).

Also add a comment to `template-sync.yml` / `.templatesyncignore` saying that `.templatesyncignore` is never synced. Repos created earlier have to copy the new ignore list (including `.github/workflows/**`) by hand.
