# AOSSIE-Org/Template-Repo: Contribution-Opportunity Audit

- **Repo:** `AOSSIE-Org/Template-Repo` @ `main` `89ae62d` (2026-11-11)
- **Audit date:** 2026-11-19
- **Derived repos compared:** `IndexedDB-Import-Export`, `ThruBox-Server`, `ThruBox-Client`
- **Mode:** read-only on GitHub. Nothing was filed, commented, forked or pushed, and all four working trees are clean. Scratch work lived under `/tmp/claude-1000/-home-pavan-Desktop-gsoc/67cea752-264c-491c-8bf0-bb277aad86e9/scratchpad/template/`.

Every repo created from this template inherits its `.github/`, `dangerfile.js` and docs. Template-sync deliberately skips `.github/workflows/**` (#76 was closed not_planned). So a workflow fix here only reaches **new** repos, and existing repos have to copy it by hand. Backports from derived repos are listed in their own section near the end.

---

## 1. Summary of tool results

| Tool | Version | Scope | Result |
|---|---|---|---|
| `actionlint` + shellcheck | 1.7.12 / 0.11.0 | 23 workflows | **14 findings.** 1 `action` error (`release-drafter@v7` has no `disable-autolabeler` input). 13 shellcheck: 12× SC2086 (info), 1× SC2129 (style). |
| `zizmor` (regular persona, online audits on) | 1.32.0 | workflows + dependabot + pre-commit | **82 reported** (166 total, 84 suppressed). By severity: 51 high, 15 medium, 9 low, 7 info. By rule: 41 `unpinned-uses`, 13 `dependabot-cooldown`, 12 `template-injection` (5 high, 7 info), 9 `artipacked`, 4 `dangerous-triggers` (`pull_request_target`), 2 `excessive-permissions`, 1 `cache-poisoning`. |
| `yamllint` (default rules) | 1.38.0 | 30 YAML files | **543** (444 error, 99 warning): 259 line-length, 173 trailing-spaces, 32 document-start, 44 comments, 23 truthy, 10 missing EOF newline, **2 real indentation errors** (`osv-scanner-release.yml:47`, `template-sync.yml:6`). |
| `markdownlint-cli2` | 0.24.1 | 8 `.md` files | **387.** Mostly style (MD013/MD032/MD036). **8× MD051 broken link fragments** (the whole CONTRIBUTING TOC) and 1× MD045 (README image without alt text). |
| `lychee` | 0.24.2 | docs, issue forms, PR template, workflows, `initial-issues.json` | Docs: **23 errors in 91 links.** 12 are expected `TODO` placeholders. The real ones are the 8 TOC anchors, `public/todo-project-logo.svg`, `docs/INSTALL_GUIDE.md`, the PR template's `./CONTRIBUTING.md`, `keylength.com/en/4/` (404) and the first criteria URL in `checklist-score.yml` (404). `initial-issues.json`: **5 errors in 50 links** (`README_TEMPLATE.md` ×4 issues, `KYA.md`, wagmi `useNetwork`, freecodecamp, honeycomb). |
| `check-jsonschema` | 0.39.0 | issue forms, issue config, dependabot, workflows, release-drafter v7 schema, CodeRabbit v2 schema | **All pass.** The schemas are fine; the bugs are semantic. |
| `node --check dangerfile.js` + mock-Danger harness | node 24 | dangerfile.js | Syntax OK. **The logic flags a 100%-filled PR template** (see TPL-06). |
| `pre-commit run --all-files` (the template's own config, on a scratch clone) | 4.7.0 | whole repo | **Fails.** trailing-whitespace and end-of-file-fixer rewrite **30 files**; `check-json` fails on `.vscode/extensions.json`. |
| `goreleaser check` and `goreleaser release --snapshot` | 2.20.2 | `.goreleaser.yaml` | `check` exits 1 (deprecated `archives.format_overrides.format`). Snapshot fails with **"does not contain a main function"**. With a stub Go main it then fails on **missing `CHANGELOG.md`**. |
| GitHub Actions run history (`gh run list` / job logs, read-only) | | Template-Repo (400 runs) plus the 3 derived repos (600 each) | Exposed several silently broken jobs (CodeQL skipped, AI moderator erroring, criteria sync no-op, gitleaks/Dependabot/template-sync/release-drafter failures). Details in the findings. |
| GitHub dedup | | 60 issues (17 open), 118 PRs (7 open) | Every finding is tagged NEW / PARTIALLY COVERED / ALREADY COVERED below. |

---

## 2. Findings table (ranked by value)

| ID | Title (ready to file) | Type | Sev | Status | Effort | Dedup |
|---|---|---|---|---|---|---|
| TPL-01 | CodeQL workflow never analyzes code: `set-codeql-language-matrix@v1` has no `matrix` output, so the Analyze job is always skipped | security/ci | **high** | VERIFIED | S | NEW |
| TPL-02 | Template ships a LaTeX `.gitignore`: `.env`, `node_modules/`, build output not ignored while README says to create `.env` | security | **high** | VERIFIED | S | NEW |
| TPL-03 | Issue AI Moderator is a silent no-op: invalid `github-token: \${{…}}` input and every prompt errors, yet it reports "no spam" | bug/ci | medium | VERIFIED (effect) / SUSPECTED (root cause of API error) | S | PARTIALLY (#168) |
| TPL-04 | OSV "release gate" can never fail: tag workflow reuses the PR-diff workflow (base = head), and version-release tags never trigger it | security/ci | medium | VERIFIED | S | NEW |
| TPL-05 | gitleaks workflow fails on every Dependabot push (no `GITLEAKS_LICENSE` for Dependabot runs) and has no `permissions:` | ci/security | medium | VERIFIED | S | NEW |
| TPL-06 | Danger PR-template check flags fully compliant PRs, nags bot PRs, and links to a 404 | bug | medium | VERIFIED | S | PARTIALLY (#112, #176) |
| TPL-07 | sync-pr-labels mislabels PRs: copies labels from unrelated `#N`, strips Dependabot's `dependencies`, tags bots as first-time contributors, never applies `org-member` | bug | medium | VERIFIED | M | NEW (related #11) |
| TPL-08 | `.templatesyncignore` too narrow: template-sync merges template content into VERSION, checklist score, CONTRIBUTING, MAINTAINERS… (and fails monthly in all derived repos) | bug | medium | VERIFIED (files included, monthly failures) / SUSPECTED (template content wins) | S | PARTIALLY (#76, #98 closed) |
| TPL-09 | Tag-triggered GoReleaser release fails in every new repo (no build config, missing CHANGELOG.md, no syft, release target hard-coded to `template-repo`) | bug/ci | medium | VERIFIED | M | NEW |
| TPL-10 | OpenSSF criteria-sync job has been a silent no-op every week (404 URL + regex matches 0 IDs) | bug/ci | medium | VERIFIED | S | NEW |
| TPL-11 | dependabot.yml: 13 always-on ecosystems cause weekly failed "Dependabot Updates" runs; `chore(deps)(deps)` double scope; no cooldown | ci | medium | VERIFIED | S | PARTIALLY (#72; #73/PR #172 don't fix it) |
| TPL-12 | Labels used by issue forms, Dependabot, stale, moderator are never provisioned (`triage-needed` silently dropped on every issue) | bug | medium | VERIFIED | S | PARTIALLY (#85, stale) |
| TPL-13 | Release Drafter v7 misconfig: `pull_request` trigger fails for bot/fork PRs, invalid `disable-autolabeler`, "Contributors" category duplicates every PR, no version-resolver | ci | low | VERIFIED | S | NEW |
| TPL-14 | Harden workflows per zizmor: script injection of inputs/`ref_name`, missing `permissions`, `persist-credentials`, mixed SHA/tag pinning | security | low | VERIFIED | M | PARTIALLY (#75) |
| TPL-15 | version-release.yml is not idempotent: tags before releasing, picks any draft, fails irrecoverably, can't trigger downstream release jobs | improvement | medium | VERIFIED | M | NEW (related #139) |
| TPL-16 | Add a placeholder check and post-creation setup guide (hidden `AOSSIE-Org/TODO` link survives in every derived repo; initial issues never created) | feature | medium | VERIFIED | M | PARTIALLY (#102) |
| TPL-17 | Best-practices scorer: score can exceed 100%, inline "N/A" boxes are ignored, bot commits on every run | bug | low | VERIFIED | S | PARTIALLY (#110) |
| TPL-18 | Template fails its own pre-commit hooks (30 files rewritten; `check-json` fails on JSONC) and CI never runs them | ci | low | VERIFIED | S | NEW |
| TPL-19 | Docs: broken CONTRIBUTING TOC, U+FFFD mojibake in headings, dead links, CONTRIBUTING's PR template contradicts the real one | docs | low | VERIFIED | S | NEW |
| TPL-20 | Cleanup bundle: Scorecard push branch `dev`, stack-specific CodeRabbit JS rules, stale comments, Stability-Nexus leftovers, yamllint indentation, etc. | improvement | low | VERIFIED (each item) | S | NEW (parts relate to #95/#96) |

---

## 3. Findings in detail

### TPL-01: CodeQL workflow never analyzes code: `set-codeql-language-matrix@v1` has no `matrix` output, so the Analyze job is always skipped

- **Type:** security/ci | **Severity:** high | **Status:** VERIFIED | **Effort:** S | **Dedup:** NEW
- **Evidence**
  - `.github/workflows/codeql.yml:29` reads `steps.set-matrix.outputs.matrix || '[]'`. `:35` uses `advanced-security/set-codeql-language-matrix@v1`. `:55` skips Analyze when the matrix is `'[]'`.
  - The floating tag `v1` → `9f9ba17a` = **v1.3.0** (latest is v1.6.0 = `038e12c9`). It's the only stale major tag among all 14 floating refs checked. At v1.3.0, `action.yml` declares **only** `outputs.languages`. The `matrix` output (with build-mode) first appears in later releases.
  - Run evidence: the create-matrix log prints `languages=['actions', 'go', 'javascript']` (ThruBox-Server run 43839125393), but the job list is `create-matrix=success, Analyze (${{ matrix.language }})=skipped`. The same happens on Template-Repo (run 43838698577). Every CodeQL run is green and analyzes nothing, including ThruBox-Server, which is a Go repo.
- **Proposed fix:** Pin `advanced-security/set-codeql-language-matrix@038e12c9… # v1.6.0`, which emits `matrix` with `build-mode`. Alternatively, keep `@v1`, read `outputs.languages`, and use `language: ${{ fromJSON(...) }}` with `build-mode: none`. Add a step that fails if the repo's languages API lists a CodeQL-supported language but the matrix is empty, so this can't regress silently. Also delete the commented `build-mode-manual-override` input at `:45`; it doesn't exist at v1.3.0.

### TPL-02: Template ships a LaTeX `.gitignore`: `.env`, `node_modules/`, build output not ignored while README says to create `.env`

- **Type:** security | **Severity:** high | **Status:** VERIFIED | **Effort:** S | **Dedup:** NEW
- **Evidence**
  - `.gitignore:1` begins `## Core latex/pdflatex auxiliary files`. All 326 lines are GitHub's TeX template.
  - `git check-ignore` on a scratch clone: `.env`, `node_modules/`, `__pycache__/`, `dist/`, `coverage/`, `venv/`, `.DS_Store`, `.vscode/settings.json` and `unremediated-security-reviews/` are all **NOT ignored**.
  - It also ignores legitimate source: `man/tool.1` (`*.[1-9]`, line 100), `src/init.el` (`*.el`, line 309), `*.table`, `*.loc`, `*.pre`, `*.end`, `*.ver`.
  - `README.md:220-228` tells contributors to "Create a `.env` file in the root directory" with `API_KEY=`. `.vscode/settings.example.json` invites copying to `settings.json`. The bundled `security-review` skill writes unfixed findings to `unremediated-security-reviews/` and expects it to be ignored.
  - All 3 derived repos had to replace it: ThruBox-Server `633fec2` "fix: replace LaTeX .gitignore with Go-specific patterns", ThruBox-Client `4c2e91f` "fix: replace LaTeX .gitignore with Node.js/TypeScript patterns", and IndexedDB `c8de460`. All three add `.env*`, `.vscode/settings.json` and `unremediated-security-reviews/`.
- **Proposed fix:** Replace the file with a language-neutral baseline: `.env`, `.env.*` with `!.env.example`, `node_modules/`, `dist/`, `build/`, `coverage/`, `__pycache__/`, `*.pyc`, `.venv/`, `venv/`, `.DS_Store`, `Thumbs.db`, `.idea/`, `.vscode/settings.json`, `*.log`, `unremediated-security-reviews/`. Add commented per-stack sections (Go, Flutter, Solidity: `out/`, `cache/`, `artifacts/`). Drop the TeX rules.

### TPL-03: Issue AI Moderator is a silent no-op: invalid `github-token: \${{…}}` input and every prompt errors, yet it reports "no spam"

- **Type:** bug/ci | **Severity:** medium | **Status:** VERIFIED (effect), SUSPECTED (root cause of the API error) | **Effort:** S | **Dedup:** PARTIALLY COVERED. Open Dependabot PR #168 only bumps `actions/github-script` 7→9 in this file.
- **Evidence**
  - `.github/workflows/issue-moderator.yml:18` reads `github-token: \${{ secrets.GITHUB_TOKEN }}`. The input name is wrong (`github/ai-moderator@v1` only accepts `token`) and the literal backslash is a copy-paste artifact.
  - Every run warns `Unexpected input(s) 'github-token', valid inputs are ['token', …]`, and the log shows `github-token: \***`.
  - All 3 runs on Template-Repo (42872094351, 42455503908, 42455170625) log `Error running prompt …/ai-detection.prompt.yml: TypeError: Cannot read properties of undefined (reading '0')`, and the same for link-spam and spam prompts. They then log **`No spam or AI-generated content detected ✅`**. The action's `dist/index.js:40781` does `response.choices[0]`, so the GitHub Models call returns no `choices`. My guess is that Models isn't enabled for the org or the call is rejected; that part is unverified.
  - Policy conflict: the follow-up step closes **and locks** any issue labeled `ai-slop` (`:47-62`). Meanwhile `CONTRIBUTING.md` "Coding with AI" says AI-assisted contributions are accepted, and a locked issue gives the reporter no way to appeal on-thread. Also `actions/github-script@v7` is a Node 20 action (the run log shows a deprecation warning).
- **Proposed fix:** Use `token: ${{ secrets.GITHUB_TOKEN }}` (or omit it). Fail the step when the moderator logs prompt errors, or open an upstream issue so errors aren't swallowed. Confirm GitHub Models is enabled for AOSSIE-Org. Reconsider auto-**lock** on `ai-slop`: label plus comment is enough. Keep close-and-lock for `spam` only. Bump github-script to v9 (merge #168). Add `concurrency` so `edited` events don't re-run after the issue is closed.

### TPL-04: OSV "release gate" can never fail: tag workflow reuses the PR-diff workflow (base = head), and version-release tags never trigger it

- **Type:** security/ci | **Severity:** medium | **Status:** VERIFIED | **Effort:** S | **Dedup:** NEW
- **Evidence**
  - `.github/workflows/osv-scanner-release.yml:19` uses `osv-scanner-reusable-pr.yml` on `push: tags: "*"`.
  - That reusable workflow (v2.7.0) runs `git checkout $GITHUB_BASE_REF`, scans, then `git checkout -f $GITHUB_SHA`, scans again, and reports only **new** vulnerabilities. On a tag push `GITHUB_BASE_REF` is empty, so both scans run on the same commit and the reporter can never find a difference.
  - Upstream docs (`google/osv-scanner/docs/github-action.md` §"Scan on release", line 135) use **`osv-scanner-reusable.yml`** (full scan).
  - The `tests`/`release` jobs (`:28-48`) are `echo` placeholders, and the workflow is misnamed "Go Release Process" (`:2`).
  - `version-release.yml:55` pushes the tag with `GITHUB_TOKEN`. Events created by `GITHUB_TOKEN` do not start new workflow runs, so neither this workflow nor `release-goreleaser.yml` ever fires from the official release path. ThruBox-Server documents exactly this in its version-release header ("A tag pushed with GITHUB_TOKEN does not trigger other workflows, so the build has to be dispatched").
  - SUSPECTED: `osv-scanner-pr.yml` also triggers on `merge_group`, where `GITHUB_BASE_REF` is likewise unset.
- **Proposed fix:** Switch to `osv-scanner-reusable.yml` with `scan-args: -r ./`. Make the release jobs real, or remove the file and call the OSV scan from version-release via `workflow_call`. In version-release, dispatch tag-triggered jobs explicitly (`gh workflow run … --ref "$TAG"`, as ThruBox-Server does) or push the tag with an App token. Rename the workflow.

### TPL-05: gitleaks workflow fails on every Dependabot push (no `GITLEAKS_LICENSE` for Dependabot runs) and has no `permissions:`

- **Type:** ci/security | **Severity:** medium | **Status:** VERIFIED | **Effort:** S | **Dedup:** NEW
- **Evidence**
  - `.github/workflows/gitleaks-scanning.yml:3` uses `push:` (all branches), `:18` reads `GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}`, and there is no `permissions:` block (zizmor `excessive-permissions`).
  - Template-Repo: **15/15 push runs on `dependabot-*` branches failed** and every push to `main` passed. Log of run 43838686929: `🛑 missing gitleaks license. … store it as a GitHub Secret named GITLEAKS_LICENSE`.
  - ThruBox-Server: 19 push failures, same message (run 43839110330). Dependabot-triggered runs only receive Dependabot secrets, not Actions secrets. Every Dependabot PR therefore shows a red check unrelated to the change.
- **Proposed fix:** Trigger on `push: branches: [main]` plus `pull_request` and `schedule`. Add `if: github.actor != 'dependabot[bot]'` on push, or add `GITLEAKS_LICENSE` as an org **Dependabot** secret. Add `permissions: contents: read` and `persist-credentials: false`. Alternatively, run the gitleaks CLI or container directly, which needs no license.

### TPL-06: Danger PR-template check flags fully compliant PRs, nags bot PRs, and links to a 404

- **Type:** bug | **Severity:** medium | **Status:** VERIFIED | **Effort:** S
- **Dedup:** PARTIALLY COVERED.
  - #112 (checklist text mismatch) is **ALREADY COVERED** for the core bug but has been assigned to kpj2006 since 2026-03-29 with no PR. Ask to take it over.
  - #176 covers re-runs on bot edits and missing concurrency.
  - The other bullets below are NEW.
- **Evidence**
  - `dangerfile.js:67` requires "My PR addresses a single issue", which no longer exists in `.github/PULL_REQUEST_TEMPLATE.md`.
  - Harness run: the PR template with an issue linked and **all boxes ticked** still yields `Some required checklist items are not completed: My PR addresses a single issue`. Live proof: PR #174 had every box `[x]` and Danger posted that exact message.
  - `CONTRIBUTING.md:197-218` "PR Description Template" (`## Related Issue / Closes #…`) fails Danger outright: missing `### Addressed Issues:` plus 3 unchecked items.
  - Danger also comments on every Dependabot PR (#168, #177, #178) about "missing required sections".
  - The link `[PR template](.github/PULL_REQUEST_TEMPLATE.md)` (`:44`) is left relative in `body_html` and resolves to `/AOSSIE-Org/Template-Repo/pull/new/.github%2FPULL_REQUEST_TEMPLATE.md` (verified with curl).
  - Nested items are emitted as `- ${item}` inside a `- ` list (`:79`, `:89`), so they render flattened (visible in the #172/#174 comments).
- **Proposed fix**
  - Derive the required items from the template, or match the current text ("My code follows the project's code style", "My changes generate no new warnings or errors").
  - Skip bots: `if: github.event.pull_request.user.type != 'Bot'` in `danger.yml`, or early-return in the dangerfile.
  - Build the link as `https://github.com/${owner}/${repo}/blob/${base}/.github/PULL_REQUEST_TEMPLATE.md`.
  - Indent nested items with two spaces.
  - Add `concurrency: {group: danger-${{ github.event.pull_request.number }}, cancel-in-progress: true}` (#176).
  - Align CONTRIBUTING's sample template (see TPL-19).

### TPL-07: sync-pr-labels mislabels PRs: copies labels from unrelated `#N`, strips Dependabot's `dependencies`, tags bots as first-time contributors, never applies `org-member`

- **Type:** bug | **Severity:** medium | **Status:** VERIFIED | **Effort:** M | **Dedup:** NEW. #11 ("security check for sync pr labels", planned, empty body) is related but different.
- **Evidence (all from `.github/workflows/sync-pr-labels.yml`)**
  1. **Fallback `/#(\d+)/g` (`:45`).** It grabs the first `#N` anywhere in the body. On Dependabot PR #178 that was `#145`, an *upstream* google/osv-scanner-action PR number in the release notes. The workflow fetched Template-Repo #145, an unrelated copyright PR, and copied its labels (`documentation`, `no-issue-linked`, `size/XS`, `repeat-contributor`, `needs-review`). The timeline shows `labeled documentation by github-actions[bot]` and similar events.
  2. **Substring matching (`:189` `file.includes(pattern)`).** `package-lock.json` and `checklist-status.json` match `.js` and get `javascript`. `src/latest/x.go` matches `test/` and gets `tests`. `dependencies` only matches `package.json`-style names, so the "remove stale managed labels" step strips Dependabot's own `dependencies` label: timeline `unlabeled dependencies by github-actions[bot]` on Template-Repo #178 and ThruBox-Client #78. Release-drafter's "⬆️ Dependencies" section therefore never gets Dependabot PRs.
  3. **Bots get contributor labels.** Dependabot is labeled `first-time-contributor` on **94/94** merged Dependabot PRs (Template-Repo 33, IndexedDB 34, ThruBox-Server 27). On #178 the label even flipped from repeat- to first-time between runs.
  4. **`org-member` is never applied** (0 of 100 recent PRs in ThruBox-Server/IndexedDB). There are two causes. `:365` checks `['admin','maintain'].includes(permission)`, but the REST `permission` field is legacy: "maintain role is mapped to write and the triage role is mapped to read" (GitHub docs), so maintainers are never matched. And `:366` pushes `'org-Member'` while setup-labels and release-drafter use `org-member`.
- **Proposed fix**
  - Only honor closing keywords for issue linking, and verify the target is an issue (`!issue.pull_request`).
  - Match by extension with `path.extname` and a basename list, and add lockfiles (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `go.sum`, `poetry.lock`, `Cargo.lock`) and workflow files to `dependencies`/`github-actions`.
  - Never remove labels the workflow didn't add.
  - Skip contributor labels when `user.type === 'Bot'`.
  - Use `role_name` (`admin`/`maintain`) or org membership, and a single lowercase `org-member` constant.

### TPL-08: `.templatesyncignore` too narrow: template-sync merges template content into VERSION, checklist score, CONTRIBUTING, MAINTAINERS… (and fails monthly in all derived repos)

- **Type:** bug | **Severity:** medium | **Status:** VERIFIED (scope of the sync commit and the monthly failures) / SUSPECTED (which side wins the merge) | **Effort:** S | **Dedup:** PARTIALLY COVERED. #76 (PAT for workflow sync, closed not_planned) and #98 (closed duplicate) cover only the workflow-permission part. The clobbering problem is NEW.
- **Evidence**
  - `.templatesyncignore:8-12` ignores only `README.md`, `LICENSE` and `.github/workflows/**`.
  - IndexedDB template-sync run 41257131894 (2026-11-01) log: `Auto-merging … BestPracticesChecklist.md, CONTRIBUTING.md, MAINTAINERS.md, PRIVACY.md, VERSION, checklist-status.json, .coderabbit.yaml, .gitignore …` and `29 files changed, 1265 insertions(+), 448 deletions(-)`. Only PRIVACY.md and README.md were held back, and only because IndexedDB added PRIVACY.md to its own ignore file. VERIFIED: per-repo files are part of the sync commit. SUSPECTED, not shown in the log (`GIT_REMOTE_PULL_PARAMS` printed empty): that template content wins. The histories are unrelated and git still reported "Automatic merge went well". If so, a merged sync PR would reset `VERSION` (which triggers version-release) and the repo's score.
  - All 3 derived repos fail **every month**: `refusing to allow a GitHub App to create or update workflow .github/workflows/anti-slop-pr.yaml without workflows permission` (IndexedDB 41257131894, ThruBox-Server 41256973180, ThruBox-Client 41255715708). They were created when their `.templatesyncignore` had `# .github/workflows/` commented out, and the ignore file itself is never synced.
- **Proposed fix:** Add per-repo files to the ignore list: `VERSION`, `checklist-status.json`, `BestPracticesChecklist.md`, `CONTRIBUTING.md`, `MAINTAINERS.md`, `PRIVACY.md`, `CHANGELOG.md`, `.github/initial-issues.json`, `public/**`. Consider `.coderabbit.yaml` and `.gitignore` too, or invert to an allow-list of truly shared files. Document in the template-sync header that pre-existing repos must update their own `.templatesyncignore` (or delete the workflow). The derived repos' failure is theirs to fix, but the template should say so.

### TPL-09: Tag-triggered GoReleaser release fails in every new repo (no build config, missing CHANGELOG.md, no syft, release target hard-coded to `template-repo`)

- **Type:** bug/ci | **Severity:** medium | **Status:** VERIFIED | **Effort:** M | **Dedup:** NEW
- **Evidence**
  - `goreleaser check` exits 1 ("uses deprecated properties": `archives.format_overrides.format`, `.goreleaser.yaml:92`).
  - `goreleaser release --snapshot --skip=publish` on a scratch clone fails with `build failed: build for template-repo does not contain a main function`. `builds:` is commented out (`:22`), so GoReleaser defaults to a Go build.
  - With a stub Go `main`, the next failure is `globbing failed for pattern CHANGELOG.md: … file does not exist` (`:86`; the template has no CHANGELOG.md).
  - `sboms:` (`:114`) needs `syft`, which `release-goreleaser.yml` never installs.
  - `release.github.name: template-repo` (`:175`) and `project_name: template-repo` (`:8`) would publish to the wrong repo.
  - `release-goreleaser.yml:30` uses `version: latest`, which is unpinned. zizmor flags `cache-poisoning` on `:21`.
  - ThruBox-Server made the workflow `workflow_dispatch`-only with the comment ".goreleaser.yaml is still the unconfigured template".
- **Proposed fix:** Make `release-goreleaser.yml` `workflow_dispatch`/`workflow_call`-only until it's configured, or have it skip when no `builds:` exist. Remove `CHANGELOG.md` from `archives.files`, or use a glob `CHANGELOG*` with `allow_different_binary_count`. Add `anchore/sbom-action/download-syft`. Drop the hard-coded `release.github` (GoReleaser infers it from the git remote). Use `formats: [zip]`. Pin `version: "~> v2"`.

### TPL-10: OpenSSF criteria-sync job has been a silent no-op every week (404 URL + regex matches 0 IDs)

- **Type:** bug/ci | **Severity:** medium | **Status:** VERIFIED | **Effort:** S | **Dedup:** NEW
- **Evidence**
  - `checklist-score.yml:174` URL `…/docs/criteria/criteria.md` returns 404. The fallback `…/docs/criteria.md` returns 200.
  - The upstream file marks criteria with `<a name="description_good">` (68 IDs), but the parser at `:200` looks for `[id](#id)` and finds **0**. It then hits `:203-205`: prints `⚠️ Warning: No criteria IDs found in upstream` and `sys.exit(0)`.
  - Template-Repo run 43837059874 (2026-11-16) log line 209 shows exactly that warning, and all 10 scheduled runs are "success". No "OpenSSF Criteria Update" PR has ever been opened.
  - A naive regex fix would open a PR **every week**: with `<a name=…>` parsing the diff is 19 IDs (`floss_license`, `maintained`, `static_analysis`, `test_continuous_integration`, …) that the checklist intentionally delegates to Scorecard (`BestPracticesChecklist.md:6-8`).
  - Branch name `chore/openssf-sync-<date>` collides on a same-day re-run.
- **Proposed fix:** Parse `<a name="([a-z0-9_]+)"`. Add an explicit `SCORECARD_COVERED = {…}` exclusion set. Make "0 IDs parsed" a **failure**, not success. Use a stable branch name and update the existing PR. Drop the 404 URL.

### TPL-11: dependabot.yml: 13 always-on ecosystems cause weekly failed "Dependabot Updates" runs; `chore(deps)(deps)` double scope; no cooldown

- **Type:** ci | **Severity:** medium | **Status:** VERIFIED | **Effort:** S
- **Dedup:** PARTIALLY COVERED. #72 (auto-discovery, planned, unassigned) would solve the failing ecosystems. #73/PR #172 (grouping, approved) does **not** address any of these three points.
- **Evidence**
  - All 13 ecosystems point at `/`. ThruBox-Client (a JS repo) has **30 failed** "Dependabot Updates" runs, e.g. 38338915157 `bundler in /`: `dependency_file_not_found … No files found in /`. ThruBox-Server has 18 failures with the same error.
  - `prefix: "chore(deps)"` + `include: "scope"` (`dependabot.yml:37-38`) produces titles like `chore(deps)(deps): bump …` on every Dependabot PR (#168, #170, #175, #177, #178; `(deps-dev)` on ThruBox-Client #78).
  - zizmor `dependabot-cooldown` ×13: no `cooldown:`, so freshly published (possibly compromised) releases are proposed immediately.
- **Proposed fix:** Keep only `github-actions` enabled and ship the other ecosystems commented out, or use `directories:` globs per #72. Use `prefix: "chore"` with `include: "scope"`, which yields `chore(deps):`. Add `cooldown: {default-days: 7}`. Coordinate with PR #172 to avoid conflicts.

### TPL-12: Labels used by issue forms, Dependabot, stale, moderator are never provisioned (`triage-needed` silently dropped on every issue)

- **Type:** bug | **Severity:** medium | **Status:** VERIFIED | **Effort:** S
- **Dedup:** PARTIALLY COVERED by #85. That issue is assigned to Okd2006, whose PR #89 was **closed by the stale bot** on 2026-03-27. Its list is also outdated: `pending-coderabbit-review` is no longer referenced anywhere since #146.
- **Evidence:** setup-labels creates 20 labels; 54 labels referenced by automation are never created. The ones that actually break behavior:
  - **`triage-needed`** (`bug_report.yml:4`, and the feature and good-first-issue forms). GitHub docs: "If a label does not already exist in the repository, it will not be automatically added to the issue." No `[BUG]`/`[FEATURE]` issue in any of the 4 repos carries it, and the label doesn't exist in any derived repo (checked with `gh label list`).
  - **Dependabot labels** `npm`, `go`, `ruby`, `rust`, `java`, `php`, `python`, `infrastructure`, `flutter`, `dart`, `dotnet`. ThruBox-Client #78 got the Dependabot comment "The following labels could not be found: `npm`. Please create it…".
  - **Stale exemptions** `Keep Open`, `Accepted`, `In Progress`, `Work In Progress`, `WIP` (`stale.yml:75,79`) don't exist, so there is no way to exempt anything.
  - `good-first-issue`/`help-wanted` (hyphenated, from initial-issues.json) duplicate the default `good first issue`, which is the only one stale exempts.
  - Also missing: `needs-review`, `PR has merge conflicts`, `Stale`, `ai-slop`, `template_sync`, `auto_pr`, `maintenance`, plus `org-Member` vs `org-member` (TPL-07).
  - Side effect: on Template-Repo alone, 7 non-Dependabot contributor PRs were auto-closed by the stale bot (#78, #82, #86, #87, #89, #97, #103). #89 and #97 were closed while awaiting maintainer review, and the `needs-review` label that every PR gets isn't exempt.
- **Proposed fix:** Generate setup-labels' list from one source of truth covering issue forms, dependabot, stale, the moderator, template-sync and release-drafter. Add a CI check that greps label references against it. Normalize `good first issue`. Consider exempting `needs-review` PRs from stale, or only staling PRs with changes requested.

### TPL-13: Release Drafter v7 misconfig: `pull_request` trigger fails for bot/fork PRs, invalid `disable-autolabeler`, "Contributors" category duplicates every PR, no version-resolver

- **Type:** ci | **Severity:** low | **Status:** VERIFIED | **Effort:** S | **Dedup:** NEW
- **Evidence**
  - actionlint: `input "disable-autolabeler" is not defined in action "release-drafter/release-drafter@v7"` (`release-drafter.yml:24`). v7 moved the autolabeler to `release-drafter/release-drafter/autolabeler@v7`.
  - `pull_request: types: [closed]` (`:8-9`) fails when Dependabot or a fork closes a PR:
    - Template-Repo 5 failures, e.g. 42592596251: `Repo load failed. Config file not found with error 404 (target: AOSSIE-Org/.github…)`
    - IndexedDB 36531039066: `Resource not accessible by integration … create-a-release`
    - The v7 README's example triggers on `push` only, with `pull-requests: read`.
  - `.github/release-drafter.yml:54-58` has a "👥 Contributors" category keyed on `first-time-contributor`/`repeat-contributor`/`org-member`. sync-pr-labels puts one of those labels on **every** PR, and v7 "by default, a pull request can match multiple categories", so every PR is listed twice.
  - There is no `version-resolver`, so `$RESOLVED_VERSION` is always a patch bump.
- **Proposed fix:** Drop the `pull_request` trigger and `disable-autolabeler`. Set `permissions: contents: write, pull-requests: read`. Delete the Contributors category (`$CONTRIBUTORS` is already in the template). Add the `version-resolver` block IndexedDB uses (`breaking`/`major` → major, `feature`/`enhancement` → minor, default patch). Optionally add a separate autolabeler job.

### TPL-14: Harden workflows per zizmor: script injection of inputs/`ref_name`, missing `permissions`, `persist-credentials`, mixed SHA/tag pinning

- **Type:** security | **Severity:** low | **Status:** VERIFIED | **Effort:** M
- **Dedup:** PARTIALLY COVERED by #75, which covers pinning only. It has been assigned to kpj2006 since 2026-02-21 with no PR. Ask before taking it.
- **Evidence (zizmor)**
  - `template-injection`, high:
    - `cache-reaper.yml:31-32` puts `'${{ github.event.inputs.stale_days }}'` into JavaScript source; a free-text input like `7'; …` breaks out.
    - `checklist-score.yml:151` and `create-initial-issues.yml:92` put `${{ github.ref_name }}` into shell.
    - All three need write access to trigger, hence low severity.
  - `excessive-permissions`: `gitleaks-scanning.yml` has no `permissions:`. `osv-scanner-release.yml` grants workflow-level `security-events: write`.
  - `artipacked` ×9: checkout without `persist-credentials: false`, including `danger.yml` under `pull_request_target`.
  - `dangerous-triggers` ×4 (`pull_request_target`). I reviewed them: none checks out PR code, and PR data is read via `context`/env, so they're acceptable but should be documented. That also answers #11.
  - Inconsistent pinning: 41 tag refs vs 9 SHA refs. `sync-pr-labels.yml:416,437` pins `actions/github-script@3a2844b7…`, which **is** `v9`, while the 12 other steps in the same file use `@v9`. `scorecard.yml:101` uses `upload-sarif@v4` next to SHA-pinned steps. TPL-01 is a concrete case of a stale floating tag causing harm.
- **Proposed fix:** Pass inputs and `ref_name` through `env:` and quote them. Add least-privilege `permissions:` everywhere. Set `persist-credentials: false` where no push follows. Pin all third-party actions to SHAs with `# vX.Y.Z` comments (Dependabot keeps them fresh). Run `zizmor` in CI (it has a GitHub Action and SARIF output).

### TPL-15: version-release.yml is not idempotent: tags before releasing, picks any draft, fails irrecoverably, can't trigger downstream release jobs

- **Type:** improvement | **Severity:** medium | **Status:** VERIFIED (logic + derived-repo fixes) | **Effort:** M
- **Dedup:** NEW. Related to #139 (npm publish), which needs these fixes first.
- **Evidence (`.github/workflows/version-release.yml`)**
  - The tag is pushed at `:55` **before** a release is found. If there's no draft, `:93` `setFailed` runs after the tag already exists, and a re-run dies at `:43` "Tag already exists". Recovery requires deleting the tag by hand.
  - `:73` publishes `releases.find(r => r.draft)`, the first draft whatever its version.
  - The "Release Summary" failure branch (`:113-121`) is dead code: the step has no `if: always()`, so it's skipped whenever the previous step fails.
  - There is no `concurrency` and no `workflow_dispatch` for re-runs. The tag is pushed with `GITHUB_TOKEN`, so tag-triggered release jobs never start (TPL-04).
  - All 3 derived repos rewrote this file (180-240 changed lines each). Their changes: concurrency, a `github.ref == 'refs/heads/main'` guard, a maintainer check, resume when the tag exists on the same commit, a preference for the draft named `vX.Y.Z`, a fallback `createRelease` with generated notes, `gh workflow run docker-publish.yml --ref "$TAG"` dispatch (ThruBox-Server), and npm publish with provenance (IndexedDB, ThruBox-Client).
- **Proposed fix:** Backport ThruBox-Client's version (generic parts) and ThruBox-Server's dispatch pattern. Then #139 can add an optional, commented `npm-publish` job.

### TPL-16: Add a placeholder check and post-creation setup guide (hidden `AOSSIE-Org/TODO` link survives in every derived repo; initial issues never created)

- **Type:** feature | **Severity:** medium | **Status:** VERIFIED | **Effort:** M
- **Dedup:** PARTIALLY COVERED by #102 (init script; unassigned, and its PR #103 was stale-closed). #102 lists "validation to ensure no TODO placeholders remain" only as a future idea.
- **Evidence**
  - `good_first_issue.yml:39` links `https://github.com/AOSSIE-Org/TODO/blob/main/CONTRIBUTING.md` (404). That link is **still present in all 3 derived repos**, while `:59` of the same form links to Template-Repo's CONTRIBUTING.
  - Other placeholders that are easy to miss:
    - README badges `AOSSIE-Org/{repo}` (`:46-47`) and `AOSSIE-Org%2Frepo%2F` (`:50`)
    - `.goreleaser.yaml` `template-repo` (still unchanged in ThruBox-Server)
    - MAINTAINERS sample row with "Context-First AI Infrastructure"
  - TODO counts: README 20, PRIVACY 19, CONTRIBUTING 8, MAINTAINERS 3.
  - **Create Initial Issues has never run** in any of the 3 derived repos (`gh run list --workflow create-initial-issues.yml` returns `[]`). The 34 KB `initial-issues.json` and its workflow are still sitting there.
  - Nothing tells a new repo's maintainers to run it, to set `GITLEAKS_LICENSE`, `EXTERNAL_LABELLER_TOKEN` or `SCORECARD_TOKEN`, to enable private vulnerability reporting (disabled on ThruBox-Server, though the org SECURITY.md calls it "preferred"), or to allow Actions to create PRs (template-sync/criteria-sync).
  - create-initial-issues isn't idempotent: if any `issues.create` fails it throws after creating the others, so a re-run duplicates them.
- **Proposed fix:** Add `TEMPLATE_SETUP.md` (or a "Setup" issue in initial-issues.json) with a checklist of secrets, settings and workflows to dispatch. Add a `placeholder-check.yml` (skipped in Template-Repo) that fails or warns on `AOSSIE-Org/TODO`, `{repo}`, `AOSSIE-Org%2Frepo`, `template-repo` and `TODO:` in tracked files. Make create-initial-issues skip titles that already exist.

### TPL-17: Best-practices scorer: score can exceed 100%, inline "N/A" boxes are ignored, bot commits on every run

- **Type:** bug | **Severity:** low | **Status:** VERIFIED | **Effort:** S
- **Dedup:** PARTIALLY COVERED by #110 (dynamic `CATEGORY_TOTALS`; assigned kpj2006, no PR). #110 does not address the category leak or the N/A handling.
- **Evidence:** I ran the workflow's own Python (`checklist-score.yml` step 2) on modified copies.
  - (a) Every item ticked plus 3 `- [x]` notes under "📎 Project-Specific Notes" → **`Score: 52/49 (106%)`**, analysis 10/7. `current_cat` (`:57`) never resets on non-category `##` headings, so the notes section counts toward Analysis.
  - (b) Ticking the inline "`[x]` N/A" boxes that 20+ items invite (e.g. `- *Note:* \`[ ]\` N/A`) → **0/49**. Only the leading `- [~]` counts (`:62`), which contradicts the per-item hints.
  - `"updated": str(date.today())` (`:83`) changes on every run, so every run commits. Each derived repo's 2nd commit is "chore: update best practices score [skip ci]".
  - SUSPECTED: `push: paths:` has no `branches:`, so it also runs on any branch, and per GitHub docs "Path filters are not evaluated for pushes of tags". A manual tag push would run update-score on a detached HEAD.
- **Proposed fix:** Reset `current_cat` on any `## ` line that isn't a category. Cap `met` at `total`. Count inline `[x] N/A` (or remove those hints). Only bump `updated` when counts change. Add `branches: [main]`. Do this together with #110.

### TPL-18: Template fails its own pre-commit hooks (30 files rewritten; `check-json` fails on JSONC) and CI never runs them

- **Type:** ci | **Severity:** low | **Status:** VERIFIED | **Effort:** S | **Dedup:** NEW. #104 (hook version bump) was closed not_planned and is a different issue.
- **Evidence:** `pre-commit run --all-files` on a scratch clone:
  - `trim trailing whitespace … Failed` and `fix end of files … Failed`, modifying 30 files (222 +/222 −). yamllint independently counts 173 trailing-space lines (90 in setup-labels.yml) and 10 missing EOF newlines.
  - `check json … Failed`: `.vscode/extensions.json: Failed to json decode (line 3 column 40)`, because the file uses `//` comments (JSONC).
  - A new contributor who runs `pre-commit install` gets unrelated rewrites as soon as they touch these files. Also, the same `pre-commit-hooks` repo is declared twice (`.pre-commit-config.yaml:18,31`).
- **Proposed fix:** Run the hooks once and commit the result. Exclude `.vscode/*.json` from `check-json` (or use `check-json5`). Merge the duplicate repo blocks. Add a `pre-commit` CI job (`pre-commit/action`) so the template stays clean.

### TPL-19: Docs: broken CONTRIBUTING TOC, U+FFFD mojibake in headings, dead links, CONTRIBUTING's PR template contradicts the real one

- **Type:** docs | **Severity:** low | **Status:** VERIFIED | **Effort:** S | **Dedup:** NEW (skills docs are partly addressed by open PR #167)
- **Evidence**
  - markdownlint MD051 ×8 and lychee "Cannot find fragment" ×8: every CONTRIBUTING TOC link (`#how-can-i-contribute` …) is broken. The emoji-prefixed headings produce `#-how-can-i-contribute`. Still broken in all 3 derived repos.
  - The U+FFFD replacement character appears in `CONTRIBUTING.md:7` (`## � IMPORTANT`), `:19` (`## �📋 Table of Contents`) and `README.md:189` (`## �🍀 Getting Started`). Also inherited by all derived repos.
  - `README.md:7` shows a missing image `public/todo-project-logo.svg` with no alt text (MD045). Closed PR #63 tried to remove it.
  - `README.md:244` links to a missing `docs/INSTALL_GUIDE.md`.
  - `PULL_REQUEST_TEMPLATE.md:20` `./CONTRIBUTING.md` renders as a relative href that resolves to `/pull/new/CONTRIBUTING.md` (verified with curl). `../blob/HEAD/CONTRIBUTING.md` works from a PR page.
  - `initial-issues.json` links to non-existent `Template-Repo/blob/main/README_TEMPLATE.md` in 4 issues, plus 4 other dead links. `BestPracticesChecklist.md:195` keylength.com is 404.
  - `CONTRIBUTING.md:197-218` "PR Description Template" differs from the real `.github/PULL_REQUEST_TEMPLATE.md` and fails Danger (TPL-06).
  - "Code of Conduct" (`CONTRIBUTING.md:5`, and the issue-form checkbox headers) is never linked, even though the org `.github` repo provides `CODE_OF_CONDUCT.md` and `SECURITY.md`.
  - `skills/` (security-review/remediation) is not mentioned in README or CONTRIBUTING, and a root-level `skills/` isn't auto-discovered by agents.
- **Proposed fix:** Drop the emoji from headings or fix the anchors. Remove the U+FFFD characters. Remove the logo `<img>` (or add a placeholder SVG plus alt text). Replace CONTRIBUTING's sample with a pointer to the real template. Link the org CoC and SECURITY. Fix or remove the dead links (`README_TEMPLATE.md` → `README.md`). Add a short "AI skills" section.

### TPL-20: Cleanup bundle

- **Type:** improvement | **Severity:** low | **Status:** VERIFIED per item | **Effort:** S | **Dedup:** NEW. CodeRabbit parts relate to #95 (Go instructions; assigned Okd2006, PR #97 stale-closed) and #96 (path_filters).
- **Items**
  - **Scorecard push trigger is `branches: ["dev"]`** (`scorecard.yml:17`) while the template default is `main`. The job `if:` requires `default_branch == ref_name`, so push scans never run: Template-Repo has 0 push runs, only schedule. IndexedDB fixed this (`["main"]` + `contents: read`). The `if:` also references `pull_request`, which isn't a trigger (`:32`).
  - **CodeRabbit JS/TS instructions are stack-specific** but apply to every `**/*.{ts,tsx,js,jsx}`: "Ensure that 'use client' is being used" (`.coderabbit.yaml:187`), "Use expo-secure-store" (`:204`), "@testing-library/react-native" (`:267`). **All 3 derived repos deleted them.** ThruBox-Server adds a concise Go block, which is a ready answer for #95. Also, `request_changes_workflow: true` is commented "(just leave comments)" (`:81-82`), which is contradictory.
  - `stale.yml:5` comment "Runs every hour at minute 35" vs cron `0 1 * * *`.
  - yamllint real errors: wrong indentation at `osv-scanner-release.yml:47` (7 spaces) and `template-sync.yml:6`.
  - Stability Nexus leftovers in an AOSSIE template: `README.md:62` `TODO.stability.nexus` vs `:15` `TODO.aossie.org`, "Stability Nexus's logo", "knights of The Stable Order". `public/stability.svg` is unused.
  - `.vscode/settings.example.json` `"source.fixAll.eslint": true` (VS Code now expects `"explicit"`).
  - `.gitattributes` has no `* text=auto eol=lf`, although `.editorconfig` and pre-commit enforce LF.
  - No `CODEOWNERS`, even though MAINTAINERS.md exists. A commented `.github/CODEOWNERS` stub would auto-request reviews.
  - `create-initial-issues.yml:35` comment typo "hecne".

---

## 4. Backport from derived repos

| Improvement (where) | Template gap | Maps to |
|---|---|---|
| Real `.gitignore` per stack with `.env*`, `.vscode/settings.json`, `unremediated-security-reviews/` (all 3: `633fec2`, `4c2e91f`, `c8de460`) | LaTeX `.gitignore` | TPL-02 |
| Scorecard `push: branches: ["main"]` and job-level `contents: read` (IndexedDB) | `dev` branch | TPL-20 |
| Release Drafter: removed `pull_request` trigger and "Contributors" category, added `version-resolver` (IndexedDB). Manual-only plus `concurrency` so it doesn't overwrite hand-written drafts (ThruBox-Client) | TPL-13 | TPL-13 |
| version-release: `concurrency`, main-only guard, maintainer-permission check, idempotent resume, version-named draft selection, fallback release with generated notes (ThruBox-Client, IndexedDB). Downstream dispatch because `GITHUB_TOKEN` tags don't trigger workflows (ThruBox-Server). npm publish with provenance (IndexedDB, ThruBox-Client) | TPL-15, TPL-04 | TPL-15, **#139** |
| GitHub-native release notes via `.github/release.yml` instead of release-drafter (ThruBox-Server) | n/a | optional alternative for TPL-13 |
| `release-goreleaser.yml` set to `workflow_dispatch` until configured, plus a real `CHANGELOG.md` (ThruBox-Server) | TPL-09 | TPL-09 |
| CodeRabbit: stack-neutral JS/TS block (all 3). Go `**/*.go`, `**/*_test.go` and `go.mod/go.sum` instructions (ThruBox-Server) | TPL-20 | TPL-20, **#95** |
| `.templatesyncignore` adds `PRIVACY.md` (IndexedDB) | TPL-08 | TPL-08 |
| `docker-publish.yml` dispatched from version-release (ThruBox-Server) | none | optional commented template |

**Template bugs the derived repos had to work around (independent confirmation):**
- LaTeX `.gitignore` (TPL-02)
- Scorecard `dev` (TPL-20)
- Release-drafter `pull_request` trigger (TPL-13)
- Unconfigured GoReleaser (TPL-09)
- Fragile version-release (TPL-15)
- Stack-specific CodeRabbit rules (TPL-20)

**Template bugs still present in all 3 derived repos:**
- Danger checklist (TPL-06)
- sync-pr-labels (TPL-07)
- Missing `triage-needed` (TPL-12)
- Dependabot ecosystems (TPL-11)
- `AOSSIE-Org/TODO` link (TPL-16)
- CONTRIBUTING TOC and mojibake (TPL-19)
- CodeQL never analyzing (TPL-01; seen on ThruBox-Server)

**Do NOT backport:** the derived repos' `# .github/workflows/` (commented-out) line in `.templatesyncignore`. It is why their template-sync fails monthly (TPL-08).

---

## 5. Open issues: pick-up-able (no assignee, no open linked PR)

| # | Title | Still valid? |
|---|---|---|
| #176 | PR Template Validation re-runs on every edit, incl. bot edits, no concurrency | **Valid.** `danger.yml` has `edited`/`synchronize` and no `concurrency`; `sync-pr-labels.yml` has the same problem. Reporter (Harshit214713) asked to be assigned 9 days ago. Pairs with TPL-06. |
| #139 | Add automated NPM publish workflow to Template Repo | **Valid.** Working implementations exist in IndexedDB and ThruBox-Client version-release (with provenance). Do TPL-15 first. |
| #102 | Template initialization script to replace placeholders | **Valid.** PR #103 was stale-closed. Combine with TPL-16 (placeholder CI check plus setup guide). |
| #96 | Add more `path_filters` to `.coderabbit.yaml` | Valid but low value. Maintainer asked for a minimal, use-case-driven set (the issue text was AI-generated). |
| #72 | Wildcard/auto-discovery for Dependabot | **Valid.** Would also fix the failing-ecosystem runs (TPL-11). Dependabot's `directories:` glob support makes this feasible (verify on a test repo first). |
| #66 | Optional role-based first-claim assignment | Valid feature, but needs maintainer design sign-off (see Zahnentferner's comment in #16). Several people asked; nobody is assigned. |
| #135 | GUAC supply-chain knowledge graph | Valid but research-sized and vague (`planned`). Not a quick PR. |
| #11 | Security check for sync-pr-labels | Vague (body is just "Follow-up of #6"). Can be closed with the analysis in TPL-14 (no PR checkout, data via `context`/env) plus the TPL-07 fixes. |
| #55 | Welcome bot | Not really free: kpj2006 said in Feb they are "personally working on this". |
| #16 | Template enhancement ideas | Idea-collection thread. It explicitly says "Do not open a pull request". |

Excluded because they have an open linked PR: **#73**, which has approved PR #172.

**Assigned but stalled (ask before taking over):**
- **#112** (Danger checklist; kpj2006 since 2026-03-29, no PR). The bug is live on every PR (see #174).
- **#110** (dynamic totals; kpj2006).
- **#75** (SHA pinning; kpj2006 since 2026-02).
- **#85** (missing labels; Okd2006, PR #89 stale-closed 2026-03-27; list outdated, see TPL-12).
- **#95** (Go CodeRabbit rules; Okd2006, PR #97 stale-closed; ThruBox-Server has a ready block).
- **#91** (multi-repo wording; kpj2006): **obsolete**. CodeRabbit's schema now allows `linked_repositories` `maxItems: 20`, so the single-repo premise is gone and the issue can be closed.

**Open PRs worth knowing about:**
- #167 (merges the skills framework into main and removes `sync-skills-branch.yml`). It fixes the failing "Sync skills branch" job: 5/5 runs fail with `Input required and not supplied: token`, and `skills` is 16 commits behind `main`.
- #168 / #170 / #175 / #177 / #178 are Dependabot bumps. #177 doesn't fix TPL-04; it only bumps the version.

---

## 6. Suggested split for a 3-person team

- **Person A (security/CI):** TPL-01, TPL-04, TPL-05, TPL-14 (coordinate with #75), TPL-11.
- **Person B (PR/issue automation):** TPL-06 (+#176, take over #112), TPL-07, TPL-12 (refresh #85), TPL-13, TPL-03.
- **Person C (release/template hygiene):** TPL-02, TPL-08, TPL-09, TPL-15 → #139, TPL-16/#102, TPL-10, TPL-17, TPL-18, TPL-19, TPL-20.

Quick wins (S effort, high signal): **TPL-01, TPL-02, TPL-03, TPL-05, TPL-10, TPL-11**.
