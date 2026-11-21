# [FEATURE]: Add a Go CI workflow that runs build, vet, tests (-race), golangci-lint and govulncheck on pull requests

## Feature and its Use Cases

**What is missing**

No workflow in `.github/workflows/` runs `go vet`, `go test` or a Go linter:

```
$ grep -rn "go test\|go vet\|golangci\|govulncheck" .github/workflows/
(no output)
```

- The only `actions/setup-go` step is in `release-goreleaser.yml`, which runs only on `workflow_dispatch`.
- On pull requests, `docker-publish.yml` builds the image, which runs `go build ./cmd/relay` inside the Dockerfile, and then checks `/health`. That catches compile errors in the binary. It does not run the test suite (54 `Test*` functions in 5 packages), `go vet`, or a linter, and it never compiles `_test.go` files.

So a PR that breaks behaviour covered by the existing tests gets green checks.

**Minimal repro** (on `main` @ `f026f22`)

1. Introduce an off-by-one in the rate limiter:
   ```
   sed -i 's/return v.count <= rl.limit/return v.count <= rl.limit+1/' internal/middleware/ratelimit.go
   ```
2. Run the build command from the Dockerfile, which is the only Go compile step that CI runs on PRs:
   ```
   $ CGO_ENABLED=1 go build -o relay-server -ldflags="-s -w" ./cmd/relay; echo $?
   0
   ```
3. Run the tests, which no workflow does:
   ```
   $ go test -count=1 ./internal/middleware/
   --- FAIL: TestRateLimiter_RotatingXFFIsStillLimited (0.00s)
   FAIL
   ```

- **Expected:** the PR check fails.
- **Actual:** every existing PR check passes, because none of them runs the tests.

**Benefit**

- Regressions in the handler, store, rate limiter and CORS code get caught before merge.
- `go vet` and lint warnings get caught too.
- It meets the `test_policy` and `warnings` items in `BestPracticesChecklist.md`. The checklist itself says: "No linter (`golangci-lint`/`go vet`) is wired into CI yet".
- `AGENTS.md` and `CONTRIBUTING.md` already tell contributors to run `go vet ./...` and `go test ./...`. CI would enforce what they already ask for.

**What the tools report on `main` today** (go1.24.6, linux/amd64)

- `go build ./...` and `go vet ./...`: clean.
- `go test -race -count=1 -cover ./...`: all packages pass, no races. `cmd/relay` has no tests.
- `staticcheck ./...`: clean.
- `errcheck ./...`: 14 unchecked errors.
  - 8 are in non-test code: `cmd/relay/main.go:68`, `internal/handler/health.go:24`, `internal/model/message.go:34`, and `internal/store/sqlite.go:43,49,116,183,261`.
  - 6 are in tests.
  - Most are `Close()` or `Rollback()` calls.
- `govulncheck ./...`: reports reachable standard-library vulnerabilities, 32 even with go1.24.6. The project pins `go 1.23.12` in `go.mod` and the Dockerfile, which is older still. A blocking govulncheck step therefore fails until the Go toolchain is upgraded.

## Proposed fix

1. Add `.github/workflows/go-ci.yml`.
   - Triggers: `pull_request` and `push` to `main`.
   - Permissions: `permissions: contents: read`.
   - Steps:
     - `actions/setup-go` with `go-version-file: go.mod`
     - `go build ./...`
     - `go vet ./...`
     - `go test -race -count=1 -coverprofile=cover.out ./...`, uploading `cover.out` as an artifact. CGO is required for `mattn/go-sqlite3`, and `ubuntu-latest` already has gcc.
     - `golangci/golangci-lint-action`, with a small committed `.golangci.yml`. Start with the defaults: errcheck, govet, staticcheck, ineffassign, unused.
     - `golang/govulncheck-action`.
2. Pin actions by commit SHA with a version comment, as `docker-publish.yml` and `release-goreleaser.yml` already do. Optionally add `step-security/harden-runner`, as `scorecard.yml` does.
3. In the same PR, fix the 8 non-test errcheck hits or exclude them with a reason, so the lint job starts green.
4. Upgrading the Go toolchain is a separate change. Until it lands, either run govulncheck with `continue-on-error: true` or on a schedule, or merge the toolchain upgrade first.
5. Afterwards, update the `warnings` / `test_policy` entries in `BestPracticesChecklist.md`, and consider making the new job a required status check.

No runtime dependencies are added. This only touches CI configuration, plus small error-handling fixes.

## Additional Context

- The PR build in `docker-publish.yml` stays useful as an image smoke test. The new workflow complements it and does not replace it.
- Related, not covered here: the `Analyze` job in `codeql.yml` has been skipped on every run since the repository was created, so CodeQL does not analyse the Go code either. This workflow comes from `AOSSIE-Org/Template-Repo` and is reported there; the same fix can then be applied here in a follow-up PR.

## Code of Conduct

- [ ] I have joined the Discord server and will post updates there
- [ ] I have searched existing issues to avoid duplicates
