# [FEATURE]: Run build, tests, lint and format checks on pull requests, and before a release is tagged

### Feature and its Use Cases

**No workflow runs the project's own checks on pull requests.** None of the workflows in `.github/workflows/` runs `npm test`, `npm run lint`, `npm run format:check`, `tsc` or `npm run build` on `pull_request`. The checks that do show up on PRs are only label-conflicts, danger, sync-labels, CodeRabbit and Socket (for example `gh pr checks 117` and `gh pr checks 133`). CodeQL is configured for PRs, but it doesn't run the tests either, and on PR #100 its analyze job was skipped.

Consequences visible on `main` (4b90eca) today:

```
$ npm run format:check
[warn] src/utils/ssr.ts
[warn] Code style issues found in the above file. Run Prettier with --write to fix.
```

That missing trailing comma has been on `main` since the file was added. PR #117's description also mentions `format:check` failures that were already on `main`. `npm test` (98 tests), `npm run lint` and `npx tsc --noEmit` pass right now, but nothing stops a PR from breaking them.

**Tests run only after the release is created.** In `version-release.yml`, the `release` job creates and pushes the tag and publishes the GitHub release. Only then does the `build` job run `npm ci`, `npm run build` and `npm test`. A failure in `build` leaves a published GitHub release with no npm package. This already happened with v1.1.0 (run 36532195076, a `npm pack` failure). PR #98 then added the manual "resume" path to recover from it.

`BestPracticesChecklist.md` lists the same gaps: "no CI step running `npm run lint` or `npm test` on every PR" and "no test coverage measurement".

**Proposal:**

1. Add `.github/workflows/ci.yml`, triggered on `pull_request` and `push` to `main`:
   - `actions/setup-node` with Node 22 and 24 (matrix), `npm ci`
   - `npm run format:check`, `npm run lint`, `npx tsc --noEmit`, `npm run build`, `npm test`
   - minimal `permissions: contents: read`; actions pinned by commit SHA, as `version-release.yml` already does for `setup-node` and `upload-artifact`
2. Fix `src/utils/ssr.ts` formatting (`npm run format`) in the same PR, so the new workflow is green from the start.
3. In `version-release.yml`, run build and tests **before** the tag and release are created. Either move them into a job that `release` depends on, or require the CI workflow to have passed on the commit.
4. Optional: add `@vitest/coverage-v8` as a devDependency and a `test:coverage` script (`vitest.config.ts` already has a `coverage` block). The current suite measures 91.7 % statements / 90.6 % branches.

### Additional Context

- No existing issue or PR proposes a CI workflow.
- Once this exists, maintainers can make the CI job a required status check for `main`.

### Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
