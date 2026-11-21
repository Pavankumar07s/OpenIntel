# [BUG]: Workflows put `${{ inputs }}` / `${{ github.ref_name }}` straight into script and shell code, and osv-scanner-release grants `security-events: write` to every job

## Bug Description

Three workflows paste GitHub expressions directly into code that then runs. GitHub substitutes `${{ … }}` as plain text before the script runs, so a value that contains quotes or shell metacharacters becomes part of the code. This is the "script injection" pattern from GitHub's [security hardening guide](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions#understanding-the-risk-of-script-injections). The guide recommends passing such values through `env:` instead.

| File:line (main @ `89ae62d`) | Expression | Runs as | Trigger |
|---|---|---|---|
| `.github/workflows/cache-reaper.yml:32` (and `:31`) | `'${{ github.event.inputs.stale_days }}'` (free-text `type: string` input) | JavaScript in `actions/github-script`, token `actions: write` | `workflow_dispatch` |
| `.github/workflows/checklist-score.yml:151-152` | `${{ github.ref_name }}` in `git fetch` / `git rebase` | bash, token `contents: write` | `push` to any branch that touches `BestPracticesChecklist.md` |
| `.github/workflows/create-initial-issues.yml:92` | `${{ github.ref_name }}` in `git pull --rebase` | bash, token `contents: write` + `issues: write` | `workflow_dispatch` (repos created from the template) |

The `stale_days` check at `cache-reaper.yml:34` doesn't help, because the injected code has already run by the time it is evaluated. Git accepts branch names such as `x$(id)` or `a;id;b` (`git check-ref-format --branch` allows them), so `ref_name` can carry shell syntax.

On the same theme of least privilege, `.github/workflows/osv-scanner-release.yml:9-15` sets `security-events: write` at workflow level. The placeholder `tests` and `release` jobs inherit it, and real release steps added in derived repos will too. Only `osv-scan` needs it, and that job already declares it at `:25-27`.

**Scope and impact:** every trigger above needs write access to the repository, either to dispatch a workflow or to push a branch. That means this is hardening, not a vulnerability that outside contributors can reach. None of the four `pull_request_target` workflows (`danger.yml`, `sync-pr-labels.yml`, `label-merge-conflicts.yml`, `anti-slop-pr.yaml`) has this problem: they don't check out PR code and they read PR data through `context` or `env`. The fix still matters because every new AOSSIE repo copies these files. The template also ships an OpenSSF Scorecard workflow and README badge, and its current Scorecard result has Token-Permissions at 0/10.

## Steps to Reproduce

1. Static check: `pip install zizmor==1.32.0 && zizmor --offline .github/workflows`. The output includes `error[template-injection]` at `cache-reaper.yml:31`, `cache-reaper.yml:32`, `checklist-score.yml:151` (×2) and `create-initial-issues.yml:92`, plus `warning[excessive-permissions]` at `osv-scanner-release.yml:13`.
2. Behaviour of `cache-reaper.yml` (local simulation): take the `script:` block, swap `${{ github.event.inputs.stale_days }}` for the input text the way the runner does, and run it with node using stub `github`/`core`/`context` objects. With the input
   ```
   7' + (console.log('INJECTED'), '') + '
   ```
   line 32 becomes
   ```js
   const staleDaysRaw = '7' + (console.log('INJECTED'), '') + '' || '7';
   ```
   and the script prints `INJECTED` before it continues normally. To check end to end on a fork, run "Cache Reaper" via *Run workflow* with that `stale_days` value and look for `INJECTED` in the step log.
3. Behaviour of `checklist-score.yml`: substitute the branch name `x$(echo${IFS}INJECTED>&2)` (a valid Git branch name) for `${{ github.ref_name }}` in `git fetch origin ${{ github.ref_name }}` and run the line in bash. It prints `INJECTED`.

## Logs and Screenshots

```
$ zizmor --offline .github/workflows   (1.32.0)
error[template-injection]  cache-reaper.yml:31, :32
error[template-injection]  checklist-score.yml:151 (x2)
error[template-injection]  create-initial-issues.yml:92
warning[excessive-permissions] osv-scanner-release.yml:13  security-events: write is overly broad at the workflow level
```

```
# local simulation of cache-reaper.yml with the payload above
INJECTED
Done: 0 deleted, 0.0 MB freed
```

## Environment Details

- `AOSSIE-Org/Template-Repo` `main` @ `89ae62d`
- zizmor 1.32.0, node 24.22.1, bash 5.2.37, git 2.51.0
- The same files are copied into every repository created from the template. Template-sync skips `.github/workflows/**`, so existing repos need the same change by hand.

## Impact

Low - Minor inconvenience

## Proposed fix

Change only the YAML. No new dependencies are needed.

```yaml
# cache-reaper.yml
      - name: Delete stale & orphaned caches
        uses: actions/github-script@v9
        env:
          DRY_RUN_INPUT: ${{ github.event.inputs.dry_run }}
          STALE_DAYS_INPUT: ${{ github.event.inputs.stale_days }}
        with:
          script: |
            const DRY_RUN = process.env.DRY_RUN_INPUT === 'true';
            const staleDaysRaw = process.env.STALE_DAYS_INPUT || '7';
```

```yaml
# checklist-score.yml (update-score job) and create-initial-issues.yml
        env:
          REF_NAME: ${{ github.ref_name }}
        run: |
          ...
          git fetch origin "$REF_NAME"
          if ! git rebase "origin/$REF_NAME"; then
          ...
          git pull --rebase origin "$REF_NAME"
```

```yaml
# osv-scanner-release.yml
permissions:
  contents: read

jobs:
  osv-scan:
    uses: "google/osv-scanner-action/.github/workflows/osv-scanner-reusable-pr.yml@v2.7.0"
    permissions:
      actions: read
      contents: read
      security-events: write
```

I tried this change locally. `zizmor` no longer reports any `template-injection` finding or the `osv-scanner-release.yml` permissions warning. With the same payload, the cache-reaper script now treats the input as data, and the existing check rejects it: `stale_days must be a positive integer, got "7' + (console.log('INJECTED'), '') + '"`.

Optional, also small:
- Add `persist-credentials: false` to the `actions/checkout` steps in `codeql.yml` and `danger.yml`. No `git push` follows in either job.
- Add a `zizmor` job (for example `zizmorcore/zizmor-action` with SARIF upload) so new workflows are checked in CI.

Out of scope: pinning actions to commit SHAs is tracked in #75. The missing `permissions:` block in `gitleaks-scanning.yml` is reported separately, together with the gitleaks Dependabot failure.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
