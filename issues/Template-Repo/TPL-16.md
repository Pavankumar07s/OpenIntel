# [FEATURE]: Good First Issue form links to the non-existent `AOSSIE-Org/TODO` repo in every derived repo; add a placeholder check and a post-creation setup checklist

## Feature and its Use Cases

**Problem 1: a placeholder that every derived repo misses.** `.github/ISSUE_TEMPLATE/good_first_issue.yml` pre-fills the "Resources" field with:

```yaml
# line 39
- [Contribution Guide - Start Here!](https://github.com/AOSSIE-Org/TODO/blob/main/CONTRIBUTING.md)
```

That URL returns 404. Line 59 of the same form links to `AOSSIE-Org/Template-Repo/blob/main/CONTRIBUTING.md`. In a derived repo, that sends contributors to the template's guide instead of the project's own guide.

The README and CONTRIBUTING placeholders are obvious, and derived repos do replace them. This one is buried in YAML, so they don't. All three repos I checked still ship the 404 link: IndexedDB-Import-Export, ThruBox-Server and ThruBox-Client (`gh api repos/AOSSIE-Org/<repo>/contents/.github/ISSUE_TEMPLATE/good_first_issue.yml`, line 39). ThruBox-Server also still has `project_name: template-repo` and `name: template-repo` in `.goreleaser.yaml`.

Other placeholders a CI check could catch: `AOSSIE-Org/{repo}` (README.md:46-47), `AOSSIE-Org%2Frepo%2F` (README.md:50), `AOSSIE-Org/TODO` (README.md:131-133, 206, 283 and CONTRIBUTING.md:97), and `template-repo` (`.goreleaser.yaml:8, 175`).

**Problem 2: nothing lists the one-time steps after "Use this template".** Some repo settings and workflows only take effect if a maintainer acts after creating the repo, and nothing in the template lists them:

- **Create Initial Issues** (`create-initial-issues.yml`) is `workflow_dispatch`-only. It has never been run in any of the three derived repos (`gh run list -R AOSSIE-Org/<repo> --workflow create-initial-issues.yml` returns `[]`). None of the 31 issue titles from `.github/initial-issues.json` appear in their issue lists, and the 34 KB JSON file and the workflow are still in all three repos.
- **Private vulnerability reporting** is enabled on Template-Repo but disabled on all three derived repos (`gh api repos/AOSSIE-Org/<repo>/private-vulnerability-reporting`). The setting does not carry over from a template, and the org `SECURITY.md` calls GitHub Security Advisories the "preferred" reporting channel.
- **Template Sync** opens its PR with `GITHUB_TOKEN`. GitHub only allows that when "Allow GitHub Actions to create and approve pull requests" is enabled at the repo or org level. None of the three derived repos has ever received a `template_sync` PR.
- **Optional secrets.** `EXTERNAL_LABELLER_TOKEN` falls back to `GITHUB_TOKEN`, and `SCORECARD_TOKEN` appears only in a commented-out line. A short note would say when each one is worth adding.

**Problem 3: Create Initial Issues is not safe to re-run.** If any `issues.create` call fails, the script throws after creating the others (lines 65-76). The cleanup step is skipped, and the next manual run creates every issue again, including duplicates.

**Use cases:**
- A maintainer creating a new repo gets one list of what to replace, enable and run.
- CI flags leftover placeholders such as the 404 link above before contributors hit them.

## Additional Context

- #102 proposes an interactive init script that replaces placeholders and "ensures no unresolved TODO placeholders remain", with CI integration listed as a future step. Its PR #103 was closed for inactivity. The CI check below is the non-interactive half of that idea and could live under #102 if maintainers prefer.
- The `TODO:` text in README/PRIVACY/MAINTAINERS is meant to be filled in by hand. The check should only fail on placeholders that end up in links or config, such as `AOSSIE-Org/TODO`, `{repo}`, `AOSSIE-Org%2Frepo` and `template-repo`.

## Proposed fix

1. In `good_first_issue.yml`, replace both CONTRIBUTING links (lines 39 and 59) with one that resolves to the current repo, or make the line-39 placeholder obvious (for example `AOSSIE-Org/<REPO_NAME>`) and list it in the checklist from step 2.
2. Add a short "After creating a repo from this template" checklist (a `TEMPLATE_SETUP.md`, or a section at the top of README that gets deleted after setup). It should cover: placeholders to replace (including the issue form and `.goreleaser.yaml`), enabling private vulnerability reporting, allowing Actions to create PRs, optional secrets, and running "Create Initial Issues" (or deleting it).
3. Add a small `placeholder-check.yml` (skipped when `github.repository == 'AOSSIE-Org/Template-Repo'`) that greps tracked files for the link/config placeholders above and fails or warns.
4. In `create-initial-issues.yml`, list the existing issue titles first and skip any title that already exists, so a re-run after a partial failure doesn't create duplicates.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
