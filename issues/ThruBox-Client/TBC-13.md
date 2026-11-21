# [FEATURE]: Add a CI workflow that builds, tests and lints every pull request, and test before tagging a release

### Feature and its Use Cases

**No workflow builds or tests pull requests.** `.github/workflows/` has 14 workflows. The ones that run on `pull_request`/`pull_request_target` are CodeQL, Dependency Review, Danger, label sync and merge-conflict labelling. None of them runs `npm ci`, `npm run build`, `npm test`, `npm run lint` or `npm run format:check`. The checks on the open PRs #81 and #83 confirm this: Socket, CodeRabbit (skipped), `danger`, `label-conflicts`, `sync-labels`. Tests only run inside `version-release.yml`, after a release has already been tagged. As a result:

- the lint and format failures on `main` went unnoticed, and PR authors now call them "pre-existing";
- reviewers have to check out each PR to find out whether it builds and passes tests;
- `BestPracticesChecklist.md` leaves `warnings_fixed` unchecked because "lint isn't currently run as a required PR CI check".

**The release workflow tags and publishes before it tests.** In `version-release.yml`, the `release` job pushes the `vX.Y.Z` tag (step "Create and push tag", line 91) and publishes the GitHub Release (line 103). Only then does the `build` job run `npm run build` and `npm test` (lines 228-231) on that tag. This already happened with 1.1.0: [run 40285482979](https://github.com/AOSSIE-Org/ThruBox-Client/actions/runs/40285482979) tagged and released `v1.1.0`, then failed in "Pack package", and 1.1.0 was not on npm until #66 and a manual re-run. #66 made re-runs possible, but the `build` job checks out the tag. If a **test** fails, a re-run builds the same failing code, so the only way out is a new version number, while the broken tag and GitHub Release stay public.

**Proposed change**

1. Add `.github/workflows/ci.yml`:
   - triggers: `pull_request` and `push` to `main`
   - `permissions: contents: read`, actions pinned by commit SHA (as `scorecard.yml` and parts of `version-release.yml` already do)
   - steps on Node 22 and 24: `npm ci`, `npm run build`, `npx tsc --noEmit`, `npm test`, `npm run lint`, `npm run format:check`. The dev toolchain cannot run on older Node: vitest 5 needs Node `^22.12.0 || ^24.0.0 || >=26.0.0`, and ESLint 10 needs `^20.19.0 || ^22.13.0 || >=24`.
   - the package itself declares `engines: node >=18.0.0`, so add a small smoke job on Node 18 and 20 that loads the built `dist/index.js` and `dist/index.cjs` (taken from the Node 22 job as an artifact) and makes one call against a stub HTTP server.
   - optional, after it is green: `npm run coverage`, and a packaging check (`npx publint`, `npx @arethetypeswrong/cli --pack .`)
2. In `version-release.yml`, build, test and pack **before** creating the tag. Either move the build into a job that the tag/release job `needs`, or call the CI workflow (`workflow_call`) and make the release job depend on it.
3. While editing `version-release.yml`, update `actions/checkout@v4` (lines 33, 205) and `actions/github-script@v7` (lines 39, 105). The runs already show "Node.js 20 is deprecated ... being forced to run on Node.js 24: actions/checkout@v4, actions/github-script@v7". The Dependabot PRs for these (#11, #12) were closed by the stale bot. The other workflows already use `checkout@v7` and `github-script@v9`.
4. Fix the contributor prerequisite in `CONTRIBUTING.md` ("Getting Started > Prerequisites" says "Node.js 18 or later"). On Node 18.20.8, `npm test` fails at startup (rolldown, used by vitest, imports `styleText` from `node:util`, which Node 18 lacks), and `npm run lint` crashes with `TypeError: util.styleText is not a function` whenever it has a problem to report. Development needs Node.js 22.12 or later (vitest 5's `engines`). The built package itself still works on Node 18: `dist/index.js` and `dist/index.cjs` gave the same `health()`/`send()`/`receive()` results on 18.20.8 as on 24.22.1, which is what the smoke job above keeps checking.

### Additional Context

- The lint/format failures on `main` need fixing first (separate issue), otherwise the new workflow starts red. Lint and format can also be added as a follow-up step once that fix lands.
- Making the new job a required status check in branch protection is a maintainer setting and is not part of the PR.
- The package's own `engines: node >=18.0.0` and the "Works in Node.js 18+" line in the README are left unchanged here. If maintainers would rather raise the runtime floor, it should go straight to `>=22` (Node 18 and 20 are both end-of-life) and be released as a breaking change, in a separate PR.

### Code of Conduct

- [ ] I have joined the Discord server and will post updates there
- [ ] I have searched existing issues to avoid duplicates
