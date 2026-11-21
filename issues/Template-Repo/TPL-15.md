# [BUG]: version-release.yml pushes the tag before it finds a release, so a missing draft leaves a stray tag and every re-run fails

## Bug Description

`.github/workflows/version-release.yml` runs when `VERSION` changes on `main`. It does its steps in this order:

1. Fail if tag `vX.Y.Z` already exists (`:40-47`).
2. Create and push tag `vX.Y.Z` (`:49-56`).
3. Look for a draft release and publish it. If there is no draft, call `core.setFailed('No draft release found to publish')` (`:58-105`).

Because the tag is pushed (step 2) before the workflow checks for a draft (step 3), a run with no draft fails **after** the tag is already on the remote. Re-running the job then stops at step 1 with `Error: Tag vX.Y.Z already exists`. A re-run can never finish the release. Someone has to delete the tag by hand, or create the release by hand in the UI.

This happens during normal use. A repo created from the template already has `VERSION` (`1.0.0`) in its initial commit. The push of that commit starts Version Release and Release Drafter at the same moment, so whether a draft exists yet is a race. In two of the three AOSSIE repos I checked that were created from this template, Version Release failed on the initial commit with "No draft release found to publish" after it had pushed `v1.0.0` (links below).

Related problems in the same file:

- **Any draft is published.** `releases.find(release => release.draft === true)` (`:73`) takes the first draft in the list, even one written for another version (for example a stale `v0.0.1` draft from Release Drafter, or a hand-written draft for a later version). It is then renamed and published as `vX.Y.Z`.
- **The failure summary is never written.** The `Release Summary` step (`:107-123`) has no `if: always()`. After `setFailed` it is skipped, so its "✗ (no draft found)" and "Release failed" branches never run. When the step does run, "Tag Created: ✓" is hard-coded.
- **No `concurrency` and no `workflow_dispatch`.** Two quick `VERSION` changes can run at the same time, and there is no manual way to finish an interrupted release.

(Tags pushed here use `GITHUB_TOKEN`, so tag-triggered workflows such as `release-goreleaser.yml` and `osv-scanner-release.yml` never start from them. That is tracked separately and is not part of this issue.)

## Steps to Reproduce

On GitHub:
1. In an `AOSSIE-Org` repo created from this template, make sure no draft release exists (or push the initial commit before Release Drafter has made one).
2. Push a change to `VERSION` on `main`, for example `1.0.0`.
3. Version Release pushes tag `v1.0.0`, then fails at "Find and Publish Draft Release". The step summary is empty.
4. Let Release Drafter create a draft, then click "Re-run jobs". The run fails at "Check if tag already exists": `Error: Tag v1.0.0 already exists`.

Locally (no GitHub needed): I ran the workflow's own `run:` blocks against a local bare repo standing in for `origin`, with the `github-script` step replaced by the same `find(draft)` / `setFailed` logic and the default rule that a step without `if:` is skipped after a failure:

```
### Run 1: VERSION=1.0.0, no draft yet
[ok]      Check if tag already exists: ✓ Tag v1.0.0 does not exist yet
[ok]      Create and push tag:  * [new tag]         v1.0.0 -> v1.0.0
[FAILED]  Find and Publish Draft Release: No draft release found to publish
[skipped] Release Summary
origin tags: ['v1.0.0']

### Re-run after a draft exists
[FAILED]  Check if tag already exists: Error: Tag v1.0.0 already exists
[skipped] Create and push tag
[skipped] Find and Publish Draft Release
[skipped] Release Summary

### Two drafts: a stale one listed first, then the one for this version
[ok]      Find and Publish Draft Release: published draft 'v0.0.1 (stale)' as v1.0.0
```

## Logs and Screenshots

Version Release runs on the template's initial commit in repos created from it (the workflow file was still the unmodified template version):

- https://github.com/AOSSIE-Org/ThruBox-Server/actions/runs/29142122506 failed with the annotation `No draft release found to publish`. Release Drafter ran on the same push in parallel (both started 06:00:41Z).
- https://github.com/AOSSIE-Org/ThruBox-Client/actions/runs/29142161312 failed with the same annotation (both started 06:01:54Z).
- IndexedDB-Import-Export's initial-commit run passed only because Release Drafter's draft already existed when it looked.

All three repos later rewrote this workflow (180-240 changed lines each) to fix exactly these problems. The ThruBox-Server commit message (`c5af7fa`) says: *"On the template's initial commit it tagged v1.0.0 and then failed, because Release Drafter had no draft yet ... version-release.yml would have published whatever draft it found first."* ThruBox-Client (`00a4f5e`) and IndexedDB-Import-Export (`4ef93b6`) each added a way to resume after `"Tag already exists"`.

## Environment Details

- `AOSSIE-Org/Template-Repo` `main` @ `89ae62d`, `.github/workflows/version-release.yml`
- GitHub-hosted `ubuntu-latest`, `actions/checkout@v7`, `actions/github-script@v9`
- The job only runs in `AOSSIE-Org` repos (`if: github.repository_owner == 'AOSSIE-Org'`).

## Impact

Medium - Feature works but has issues

## Proposed fix

Workflow-only change, with no new dependencies. Most of it can come from the versions already running in ThruBox-Server and ThruBox-Client:

1. **Make re-runs safe.** If `vX.Y.Z` already exists **on the same commit**, reuse it instead of failing. If it points at a different commit, fail with a clear message. If the release for `vX.Y.Z` is already published, exit successfully.
2. **Don't leave a stray tag.** Either have the release create the tag (`updateRelease`/`createRelease` with `tag_name` and `target_commitish: context.sha`, or `gh release create "$TAG" --target "$GITHUB_SHA"`), or make sure a release always gets created once the tag has been pushed.
3. **Pick the right draft.** Prefer a draft whose `name` or `tag_name` is `vX.Y.Z`. If there is no draft, create the release with `generate_release_notes: true` instead of failing. This replaces the commented-out `createRelease` block.
4. Add `if: always()` to `Release Summary` and report the real outcome (`job.status`).
5. Add `concurrency: { group: version-release, cancel-in-progress: false }` and `workflow_dispatch:`, and guard the job with `github.ref == 'refs/heads/main'` so a manual run can't release from another branch.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
