# [BUG]: `triage-needed` and Dependabot ecosystem labels are never created, so GitHub silently drops them

## Bug Description

Some labels in the template are applied by GitHub features that **do not create a label if it is missing**. `setup-labels.yml` never creates these labels, so they are never applied. This affects this repo and every repo created from the template.

1. **`triage-needed` (all three issue forms).** `bug_report.yml`, `feature_request.yml` and `good_first_issue.yml` set `labels: [..., "triage-needed"]` (added in #51). The GitHub docs for issue forms say: *"If a label does not already exist in the repository, it will not be automatically added to the issue."* `triage-needed` does not exist in Template-Repo, IndexedDB-Import-Export, ThruBox-Server or ThruBox-Client. So no issue opened from a form has ever had it, and there is no triage queue to filter on.
2. **Dependabot ecosystem labels.** `dependabot.yml` uses `npm`, `go`, `ruby`, `rust`, `java`, `php`, `infrastructure`, `flutter`, `dart` and `dotnet`. `setup-labels.yml` doesn't create any of them; it only creates `dependencies`, `github-actions`, `docker` and `python`. Dependabot does not create labels either. It posts an error comment on the PR and leaves the label off.
3. **Stale exemption labels (minor).** `stale.yml` exempts `Keep Open`, `Accepted`, `In Progress` (issues) and `Keep Open`, `Work In Progress`, `WIP` (PRs). None of these exist in a new repo, so a maintainer has to create one by hand before it can be used. (`help wanted` and `good first issue` are GitHub defaults and already work.)

Labels that workflows add through the REST API (`needs-review`, `Stale`, `PR has merge conflicts`, `no-issue-linked`, `ai-slop`) and labels that actions create themselves (`template_sync`/`auto_pr` from actions-template-sync, `maintenance` from `checklist-score.yml`) are **not** affected. Those get created the first time they are used.

## Steps to Reproduce

1. Open a new issue with the **Bug Report** form in this repo or any derived repo.
2. The issue gets `bug` but not `triage-needed`.
3. List the labels: `gh label list -R AOSSIE-Org/Template-Repo -L 200 | grep triage` returns nothing. The same is true for IndexedDB-Import-Export, ThruBox-Server and ThruBox-Client.
4. Look at a Dependabot PR in a derived repo, for example AOSSIE-Org/ThruBox-Client#78 (npm), AOSSIE-Org/ThruBox-Server#77 (gomod) or AOSSIE-Org/IndexedDB-Import-Export#118–#122 (npm).

## Logs and Screenshots

Dependabot comment on AOSSIE-Org/ThruBox-Client#78:

```
### Labels

The following labels could not be found: `npm`. Please create it before Dependabot can add it to a pull request.

Please fix the above issues or remove invalid values from `dependabot.yml`.
```

Issues opened from a form (titles starting with `[BUG]`, `[FEATURE]` or `[GOOD FIRST ISSUE]`):

| Repo | Form issues | Have `bug`/`enhancement` | Have `triage-needed` |
|---|---|---|---|
| Template-Repo | 11 | 11 | 0 |
| IndexedDB-Import-Export | 12 | 12 | 0 |
| ThruBox-Server | 2 | 2 | 0 |
| ThruBox-Client | 3 | 3 | 0 |

## Environment Details

- `main` at `89ae62d`
- Files: `.github/ISSUE_TEMPLATE/{bug_report,feature_request,good_first_issue}.yml` (line 4), `.github/dependabot.yml`, `.github/workflows/stale.yml` (lines 75, 79), `.github/workflows/setup-labels.yml` (list of 20 labels)

## Impact

Low - Minor inconvenience

## Proposed fix

- Add `triage-needed` to the `requiredLabels` array in `setup-labels.yml`. Also add the Dependabot ecosystem labels (`npm`, `go`, `ruby`, `rust`, `java`, `php`, `infrastructure`, `flutter`, `dart`, `dotnet`) and the stale exemption labels (`Keep Open`, `Accepted`, `In Progress`, `Work In Progress`, `WIP`).
- Another option for Dependabot: remove the per-ecosystem labels from `dependabot.yml` and keep only `dependencies`.
- Existing derived repos would need to copy the updated `setup-labels.yml` and run it once with `workflow_dispatch`, because template sync skips `.github/workflows/`.

Related: #85 (asks for `pending-coderabbit-review`, `template_sync` and `auto_pr` in `setup-labels.yml`; its PR #89 was closed as stale). That issue is about labels that are created automatically. This issue is about labels GitHub never creates.
