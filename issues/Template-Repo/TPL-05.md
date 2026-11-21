# [BUG]: gitleaks workflow fails on every Dependabot PR because `GITLEAKS_LICENSE` is not available to Dependabot runs

## Bug Description

`.github/workflows/gitleaks-scanning.yml` runs on `push` to any branch and passes `GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}` to `gitleaks/gitleaks-action@v3`.

GitHub does not give Actions secrets to workflow runs triggered by Dependabot. Those runs only get **Dependabot** secrets. So when Dependabot pushes its `dependabot-*` branch, `GITLEAKS_LICENSE` is empty. `AOSSIE-Org` is an organization, so the action requires a license and exits with code 1 before it scans anything. The gitleaks check therefore fails on every Dependabot PR, whatever the PR changes.

Run history for this workflow on Template-Repo:

| Trigger | Runs | Result |
|---|---|---|
| `push` by `dependabot[bot]` (on `dependabot-*` branches) | 33 | **33 failed** |
| `push` by anyone else (`main` and other branches) | 56 | 56 passed |
| `schedule` | 168 | 168 passed |

Repos created from this template show the same thing. In ThruBox-Server, all 24 Dependabot push runs failed. In ThruBox-Client, all 3 failed.

There is a second, smaller problem in the same file: it has no `permissions:` block. On `main` and scheduled runs, the "Set up job" log shows the `GITHUB_TOKEN` gets **write** access to every scope (Contents, Actions, Packages, PullRequests, …), and that token is handed to a third-party action. For `push`, `schedule` and `workflow_dispatch` events, the action only calls the public `GET /users/{owner}` endpoint. Checkout only needs `contents: read`.

## Steps to Reproduce

1. Open any Dependabot PR, for example #178. On its checks, `gitleaks` is the only failing check.
2. Open that check's job log (run `43838686929`). The env dump shows `GITLEAKS_LICENSE:` with an empty value, followed by the error below.
3. For comparison, open the push-to-`main` run `42983922863`. It shows `GITLEAKS_LICENSE: ***` and `✅ No leaks detected`.
4. Count the results across all runs:
   ```sh
   gh api --paginate "repos/AOSSIE-Org/Template-Repo/actions/workflows/gitleaks-scanning.yml/runs?per_page=100" \
     --jq '.workflow_runs[] | "\(.event) \(.actor.login) \(.conclusion)"' | sort | uniq -c
   ```

## Logs and Screenshots

Dependabot push run `43838686929` (2026-11-16):
```
GITLEAKS_LICENSE:
[AOSSIE-Org] is an organization. License key is required.
##[error]🛑 missing gitleaks license. Go grab one at gitleaks.io and store it as a GitHub Secret named GITLEAKS_LICENSE.
```
The same run's token is read-only: `Contents: read`, `PullRequests: read`, and so on. GitHub applies this to Dependabot-triggered runs.

Push-to-`main` run `42983922863` (2026-11-11):
```
GITLEAKS_LICENSE: ***
[AOSSIE-Org] is an organization. License key is required.
... no leaks found
✅ No leaks detected
```
That run's token has `Contents: write`, `Actions: write`, `Packages: write`, `PullRequests: write`, and write access to every other scope.

The license check that fails is in `gitleaks/gitleaks-action@v3` (`e0c47f4`), `dist/index.js`:
```js
if (shouldValidate && !process.env.GITLEAKS_LICENSE) {
  core.error("🛑 missing gitleaks license. ...");
  process.exit(1);
}
```

## Environment Details

- Repo: `AOSSIE-Org/Template-Repo`, `main` @ `89ae62d`
- Workflow: `.github/workflows/gitleaks-scanning.yml` (`on: push` with no branch filter, plus `workflow_dispatch` and `schedule`)
- Action: `gitleaks/gitleaks-action@v3` (resolves to `e0c47f4`), runner `ubuntu-latest`
- Repository owner type: Organization, so the action requires a license key
- Also affects repos created from the template, e.g. ThruBox-Server and ThruBox-Client

## Impact

Low - Minor inconvenience. Every Dependabot PR shows a red check that has nothing to do with its change, in this repo and in every repo created from it. This teaches people to ignore the gitleaks check. Dependabot commits are still scanned after merge, by the push-to-`main` run and the daily scheduled run.

## Proposed fix

Either of these fixes the failure:

- **Org setting, no code change:** add `GITLEAKS_LICENSE` as an organization **Dependabot** secret (Settings → Secrets and variables → Dependabot). This needs an org admin.
- **Template change:** skip the scan for Dependabot runs, so the check shows as skipped instead of failed:
  ```yaml
  permissions:
    contents: read

  jobs:
    scan:
      if: github.actor != 'dependabot[bot]'
  ```
  Another way is to limit `push` to `branches: [main]`. The scheduled run already scans the default branch every day.

Adding `permissions: contents: read` at the top level also fixes the over-broad token described above.

Do **not** re-add a `pull_request` trigger. PRs from forks get no secrets either, and every fork-PR run failed the same way until `pull_request` was removed in `62780d1`.

A longer-term option is to run the MIT-licensed gitleaks CLI or container image directly. It needs no license key, so it could also scan fork PRs.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
