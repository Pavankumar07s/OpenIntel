# [BUG]: socket.yml skips Socket "Pull Request Alerts" on Dependabot PRs and on Go/Rust/Java/Ruby/.NET manifest changes

## Bug Description

Two settings in `socket.yml` stop the Socket GitHub app from running its **Pull Request Alerts** check on most dependency-changing PRs in repos created from this template.

1. **`githubApp.ignoreUsers` lists `dependabot[bot]`** (`socket.yml:26-28`). The [Socket docs](https://docs.socket.dev/docs/socket-yml) say `ignoreUsers` "will prevent Pull Request Alerts from running on Pull Requests that are opened by anyone specified in this array". Dependabot PRs are the ones that bring in newly published package versions, and `.github/dependabot.yml` has no `cooldown`. A supply-chain scanner most needs to see exactly these PRs.
2. **`triggerPaths` is limited to JS/Python/Dart files** (`socket.yml:11-19`): `package.json`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `requirements.txt`, `pyproject.toml`, `pubspec.yaml`, `pubspec.lock`. The patterns work like `.gitignore` patterns, and a PR alert run starts only when a changed file matches one of them. If `triggerPaths` is left out, the default is `["*"]`. As a result, a PR that changes only the following files gets no PR alerts:
   - Go: `go.mod`, `go.sum`
   - Rust: `Cargo.lock`
   - Java/Kotlin: `pom.xml`, `build.gradle.kts`, `gradle.lockfile`
   - Ruby: `Gemfile.lock`
   - .NET/NuGet: `*.csproj`, `packages.lock.json`
   - Python: `poetry.lock`, `uv.lock`, `Pipfile`, `Pipfile.lock`, `requirements-dev.txt`
   - JS: `bun.lock`

   Socket lists all of these ecosystems as GA ([language support](https://docs.socket.dev/docs/language-support)). Most of them are also enabled in this template's own `dependabot.yml` (gomod, cargo, maven, gradle, bundler, nuget, pip). Dart/pub, which *is* in the list, gets CVE-only coverage in Socket.

The "Socket Security: Project Report" check still runs on every commit. That report covers the whole project. The check that is skipped is the per-PR one, which flags packages newly added by the PR.

This already affects at least one repo: AOSSIE-Org/MCP-Template was created from this template on 2026-08-27 with the same `socket.yml`. None of its Dependabot npm PRs (#8, #10, #12-#14, #20-#23, #25, #28, #29) has a Pull Request Alerts check.

## Steps to Reproduce

1. MCP-Template, Dependabot PR #29 (`@modelcontextprotocol/sdk` bump). It changes `package.json` and `package-lock.json`, and both match `triggerPaths`:
   ```sh
   gh api repos/AOSSIE-Org/MCP-Template/commits/71ef404b95e50e2160094b6c00d148f607516107/check-runs \
     -q '.check_runs[]|select(.app.slug=="socket-security")|.name'
   ```
   Output: `Socket Security: Project Report` only. PRs #28 (`b5f0238`), #25 (`9e3faf7`) and #8 (`8a416b8`) give the same result.
2. Same repo, same `socket.yml`, human PR #4. It also changes `package.json` and `package-lock.json`:
   ```sh
   gh api repos/AOSSIE-Org/MCP-Template/commits/4a1751f2d8147ace4d7a3edd5f750cfd322b7695/check-runs \
     -q '.check_runs[]|select(.app.slug=="socket-security")|.name'
   ```
   Output: `Socket Security: Pull Request Alerts` and `Socket Security: Project Report`. The only difference from step 1 is the PR author, so `ignoreUsers` is what removes the check.
3. Repos without a `socket.yml`, where Socket's defaults apply:
   - ThruBox-Server Dependabot PR #77 changes `go.mod` and `go.sum` (head `4320726`).
   - ThruBox-Client Dependabot PR #78 changes `package-lock.json` (head `6a5264d`).

   Both get `Socket Security: Pull Request Alerts`. On #77 the result is "PR #77 Alerts: No new dependency alerts", so Socket did analyse the Go change. With this template's `socket.yml`:
   - #77 would lose the check twice over: Dependabot is the author, and `go.mod`/`go.sum` are not in `triggerPaths`.
   - #78 would lose it through `ignoreUsers`.
4. `triggerPaths` in this repo:
   - PR #145 changed `README.md` and was opened before `socket.yml` was merged. It got Pull Request Alerts.
   - PR #148 changed `MAINTAINERS.md` and was opened after the merge. It did not.

## Logs and Screenshots

| Repo / PR | Author | Files | socket.yml | Socket check runs |
|---|---|---|---|---|
| MCP-Template #29 | dependabot[bot] | package.json, package-lock.json | template copy | Project Report |
| MCP-Template #4 | human | package.json, package-lock.json | template copy | Pull Request Alerts, Project Report |
| ThruBox-Server #77 | dependabot[bot] | go.mod, go.sum | none | Pull Request Alerts, Project Report |
| ThruBox-Client #78 | dependabot[bot] | package-lock.json | none | Pull Request Alerts, Project Report |
| Template-Repo #145 | human | README.md | none yet | Pull Request Alerts ("Skipped: no net changes to dependencies"), Project Report |
| Template-Repo #148 | human | MAINTAINERS.md | present | Project Report |

## Environment Details

- `socket.yml` on `main` (`89ae62d`). It was added in #142 and last changed in `b65919d`.
- Socket GitHub app (`socket-security`), installed for the whole org. Its check runs appear on Template-Repo, MCP-Template, ThruBox-Server and ThruBox-Client.
- `socket.yml` is not in `.templatesyncignore`. Every new repo created from the template gets it, and template-sync PRs carry it to existing repos.

## Impact

Medium - Feature works but has issues

## Proposed fix

- Remove `dependabot[bot]` from `githubApp.ignoreUsers`. Keep `github-actions[bot]` if template-sync PRs should stay skipped.
- Remove `triggerPaths` so that Socket's default `["*"]` applies. A PR with no dependency changes then gets a green "Skipped: Pull request contains no net changes to dependencies" result, as #141 and #145 did before `socket.yml` existed, so the extra noise is small.
- If a filter is wanted anyway, add a manifest for every ecosystem that `dependabot.yml` enables:
  - `go.mod`, `go.sum`
  - `Cargo.toml`, `Cargo.lock`
  - `pom.xml`, `build.gradle*`, `gradle.lockfile`, `libs.versions.toml`
  - `Gemfile`, `Gemfile.lock`
  - `*.csproj`, `packages.lock.json`
  - `composer.json`, `composer.lock`
  - `Pipfile`, `Pipfile.lock`, `poetry.lock`, `uv.lock`, `requirements*.txt`
  - `bun.lock`, `npm-shrinkwrap.json`
  - `socket.yml`
- Make the same change in repos that already copied the file, such as MCP-Template.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
