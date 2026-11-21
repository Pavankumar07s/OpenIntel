# [BUG]: OSV release scan can't fail on vulnerabilities (uses the PR-diff workflow), and tags pushed by version-release never trigger it

## Bug Description

`.github/workflows/osv-scanner-release.yml` is meant to block a release when OSV-Scanner finds vulnerabilities: the `release` job `needs: [tests, osv-scan]`. In practice it can't do that, for two separate reasons.

**1. It calls the PR-diff reusable workflow, which compares the tagged commit with itself.**

`osv-scanner-release.yml:19` runs on `push: tags: "*"`, but it calls
`google/osv-scanner-action/.github/workflows/osv-scanner-reusable-pr.yml@v2.7.0`. That workflow is built for pull requests. It:

1. runs `git checkout $GITHUB_BASE_REF` and scans ("existing code"),
2. runs `git checkout -f $GITHUB_SHA` and scans again ("new code"),
3. runs `osv-reporter --old=… --new=… --fail-on-vuln=true`, which fails only on vulnerabilities that are in the new results and **not** in the old results (`DiffVulnerabilityResultsByOccurrences`).

GitHub sets `GITHUB_BASE_REF` only for `pull_request` / `pull_request_target` events. On a tag push it is empty, so step 1 runs a bare `git checkout`, which does nothing and exits 0. Both scans then run on the same tagged commit, the diff is always empty, and the job passes no matter how many known vulnerabilities the release contains. It still fails if the scanner itself errors out, but it never fails because of a vulnerability.

The upstream "Scan on release" example (https://google.github.io/osv-scanner/github-action/#scan-on-release, linked on line 1 of this file) uses the full-scan workflow `osv-scanner-reusable.yml`. The template has used `-pr.yml` since the file was added (`4d924b4`). Dependabot bumps since then only changed the version.

**2. The template's own release path never starts this workflow.**

`version-release.yml:55` pushes the `vX.Y.Z` tag with the `GITHUB_TOKEN` that `actions/checkout` persists. GitHub does not start new workflow runs for events created with `GITHUB_TOKEN`, except `workflow_dispatch` and `repository_dispatch`. So this workflow never runs on those tags. The same applies to `release-goreleaser.yml`. Every repo created from the template shows this (see Logs).

A few smaller problems in the same file:
- `scan-args` is `./` without `-r`, so only the top-level directory is scanned. The comment above it says "Recursive scan supports multiple ecosystems".
- The workflow is named "Go Release Process", copied from the upstream example.
- The `tests` and `release` jobs are `echo` placeholders.

Mitigation: `osv-scanner-scheduled.yml` already runs a full scan on every push to `main` and weekly. Vulnerabilities still show up in code scanning, but nothing stops a release because of them.

## Steps to Reproduce

1. Read `.github/workflows/osv-scanner-release.yml:19` and the called workflow at the pinned ref:
   `gh api "repos/google/osv-scanner-action/contents/.github/workflows/osv-scanner-reusable-pr.yml?ref=v2.7.0" --jq .content | base64 -d | sed -n 83,150p`
2. Simulate the workflow's git steps for a tag push locally:
   ```bash
   git init -q -b main origin && cd origin
   echo '{"lockfileVersion":3,"packages":{"node_modules/lodash":{"version":"4.17.15"}}}' > package-lock.json
   git add . && git -c user.name=t -c user.email=t@t commit -qm vuln && git -c user.name=t -c user.email=t@t tag -a v1.0.0 -m rel
   cd .. && git clone -q --no-checkout origin runner && cd runner
   git checkout -q --force refs/tags/v1.0.0            # what actions/checkout does on a tag push
   export GITHUB_SHA=$(git rev-parse HEAD) GITHUB_BASE_REF=""
   git checkout $GITHUB_BASE_REF; echo "exit=$?"        # "Checkout target branch" step
   echo "old scan on $(git rev-parse --short HEAD)"
   git checkout -q -f $GITHUB_SHA
   echo "new scan on $(git rev-parse --short HEAD)"
   ```
   Both scans run on the same commit. `lodash@4.17.15` has 5 OSV advisories (`curl -s -X POST https://api.osv.dev/v1/query -d '{"package":{"name":"lodash","ecosystem":"npm"},"version":"4.17.15"}'`). The same 5 IDs are in "old" and "new", so osv-reporter reports 0 new vulnerabilities and exits 0.
3. In any repo created from this template that cut a release with `version-release.yml`, compare the tag with the Actions history: the tag exists but "Go Release Process" has 0 runs.

## Logs and Screenshots

Output of step 2 (the hash will differ on your machine):
```
exit=0
old scan on 63891f7
new scan on 63891f7
```

Derived repos with this exact `osv-scanner-release.yml` present at the tagged commit. Each `v1.0.0` tag was created by `github-actions[bot]` through `version-release.yml`:

| Repo | Tag created | `osv-scanner-release.yml` runs |
|---|---|---|
| ThruBox-Server | 2026-10-26 | 0 |
| MCP-Template | 2026-08-27 | 0 |
| PythonScripts | 2026-06-19 | 0 |
| PullRequestDashboard | 2026-06-04 | 0 |
| SkillBot | 2026-06-04 | 0 |

(`gh api repos/AOSSIE-Org/<repo>/actions/workflows/osv-scanner-release.yml/runs --jq .total_count`). In ThruBox-Server the only run on `v1.0.0` is the Docker build that its `version-release.yml` dispatches by hand. Its header says: "A tag pushed with GITHUB_TOKEN does not trigger other workflows, so the build has to be dispatched."

## Environment Details

- `AOSSIE-Org/Template-Repo` `main` @ `89ae62d`
- `google/osv-scanner-action` reusable workflows `v2.7.0` (open Dependabot PR #177 bumps the same line to v2.7.1 without changing the workflow)
- GitHub-hosted runners, `push` (tag) event

## Impact

Low - Minor inconvenience. The release gate silently does nothing. Main-branch and weekly full scans still report vulnerabilities, but derived repos that fill in real release steps would be relying on a check that cannot fail.

## Proposed fix

1. In `osv-scanner-release.yml`, switch to the full-scan workflow, as the upstream example does:
   ```yaml
   osv-scan:
     uses: "google/osv-scanner-action/.github/workflows/osv-scanner-reusable.yml@v2.7.0"
     with:
       scan-args: |-
         -r
         ./
   ```
2. Make the release path actually run it. Either:
   - add `workflow_dispatch:` to `osv-scanner-release.yml` and dispatch it from `version-release.yml` after pushing the tag (`gh workflow run osv-scanner-release.yml --ref "v$VERSION"`, with `actions: write`), as ThruBox-Server does for its Docker build; or
   - push the tag with a GitHub App token instead of `GITHUB_TOKEN`.

   The second option would also start `release-goreleaser.yml`, which is not configured yet, so the first option is safer for the template.
3. Rename the workflow, for example "OSV-Scanner Release Scan", and fix the "Recursive scan" comment so it matches the args.
