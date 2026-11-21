# [BUG]: GoReleaser setup can't produce a release: Go build runs by default, CHANGELOG.md is missing, syft isn't installed, release repo is hard-coded

## Bug Description

`.github/workflows/release-goreleaser.yml` runs `goreleaser release --clean` on every pushed `v*.*.*` tag. The `.goreleaser.yaml` it uses cannot finish a release, either as shipped or after following the file's own setup steps ("uncomment the sections relevant to your project type"):

1. **The default config still builds Go.** `builds:` is commented out (`.goreleaser.yaml:22-42`). When `builds` is empty, GoReleaser adds a default build with `builder: go`, `main: .` and binary `project_name` (`internal/pipe/build/build.go`, `buildWithDefaults`). In any repo without a Go `main` package, which covers most repos made from this template, the run fails with `build for template-repo does not contain a main function`. Commenting the block out does not turn GoReleaser off.
2. **`CHANGELOG.md` is listed in `archives[0].files` (`:86`), but the template has no such file.** GoReleaser treats a literal path that does not exist as an error (`globbing failed for pattern CHANGELOG.md: ... file does not exist`), so the archive step fails even after a real build is configured. The same file says changelogs come from Release Drafter (`:185`), and Release Drafter never writes a `CHANGELOG.md`.
3. **`sboms:` is enabled (`:114-116`) and uses GoReleaser's default `syft` command,** but the workflow never installs syft, and the `ubuntu-latest` image does not include it. The SBOM step fails once archives exist.
4. **`release.github` is hard-coded to `owner: AOSSIE-Org` / `name: template-repo` (`:172-175`).** When `release.github.name` is set, GoReleaser uses it as given and does not read the git remote. A derived repo therefore tries to create its release in `AOSSIE-Org/template-repo`. The job's `GITHUB_TOKEN` cannot write there, so the publish fails. `project_name: template-repo` (`:8`) also ends up in every archive and release name.
5. **Step 2 of the header says "Fill in placeholders marked with `<REPLACE_ME>`" (`:3`), but the file contains no `<REPLACE_ME>` markers.** Nothing points users to `project_name` or `release.github`.
6. **Step 3 of the header says to run `goreleaser check`.** On the shipped file it exits non-zero (exit code 2) because `archives.format_overrides.format` (`:90-92`) is deprecated in favour of `formats`.

Context: tags that `version-release.yml` creates are pushed with `GITHUB_TOKEN`, so they do not start this workflow. Today it runs only when a maintainer pushes a `vX.Y.Z` tag by hand. Template-derived repos that have the workflow and `v1.0.0`/`v1.0.1` tags created by the bot have no "Release" runs. Wiring the two workflows together is a separate topic. This issue covers making the GoReleaser config itself work.

## Steps to Reproduce

Local, with GoReleaser v2 (`version: latest` in the workflow currently resolves to v2.20.2):

```bash
git clone https://github.com/AOSSIE-Org/Template-Repo && cd Template-Repo
goreleaser check                                   # exit 2: archives.format_overrides.format is deprecated
goreleaser release --snapshot --clean --skip=publish
# => build failed: build for template-repo does not contain a main function

# Simulate a project that has "configured" a Go build:
go mod init example.com/stub && printf 'package main\n\nfunc main() {}\n' > main.go
goreleaser release --snapshot --clean --skip=publish
# => globbing failed for pattern CHANGELOG.md: ... file does not exist

touch CHANGELOG.md
goreleaser release --snapshot --clean --skip=publish
# => sbom step fails if syft is not on PATH (as on ubuntu-latest)
```

In CI: in a repo created from this template, push a tag by hand (`git tag v0.0.1 && git push origin v0.0.1`). The "Release" workflow fails at the first of the steps above.

## Logs and Screenshots

```
build failed: build for template-repo does not contain a main function
Learn more at https://goreleaser.com/errors/no-main
```

```
globbing failed for pattern CHANGELOG.md: matching "./CHANGELOG.md": file does not exist
```

## Environment Details

- `main` @ `89ae62d`
- GoReleaser v2.20.2 (`goreleaser/goreleaser-action` with `version: latest`)
- Runner: `ubuntu-latest` (Ubuntu 24.04 image, no syft preinstalled)

## Impact

Low - Minor inconvenience. The workflow is opt-in in practice: it runs only on manually pushed tags. But every repo that tries to enable GoReleaser by following the template hits these failures one at a time.

## Proposed fix

- `.goreleaser.yaml`
  - Remove `CHANGELOG.md` from `archives[0].files`, or change it to the glob `CHANGELOG*`. A glob that matches nothing only logs a warning.
  - Comment out `sboms:` by default, the same way `builds:` is, or keep it and install syft in the workflow (`anchore/sbom-action/download-syft`, pinned by SHA).
  - Delete the hard-coded `release.github` block so GoReleaser takes owner and name from the git remote. Either drop `project_name` (it defaults to the repo name) or mark it `<REPLACE_ME>`, so the header's instructions match the file.
  - In `format_overrides`, replace the deprecated `format: zip` with `formats: [zip]`.
- `.github/workflows/release-goreleaser.yml`
  - Skip the release while the config is still the unconfigured template, for example with an early step that exits successfully and sets an output when `.goreleaser.yaml` has no top-level `builds:` key, used as an `if:` on the GoReleaser step. Alternatively, make the workflow `workflow_dispatch`-only until a build is configured.
  - Pin `version: "~> v2"` instead of `latest`.
