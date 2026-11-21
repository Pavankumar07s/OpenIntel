# [FEATURE]: Add tests for untested paths: `cmd/relay` wiring and purge loop, `/health`, `APIKeyAuth`, `HandleCreate` error responses

## Feature and its Use Cases

The test suite is in good shape: 54 tests, all passing, 64.8% of statements covered. Some of the code behind the relay's security properties is still never executed by any test, so a regression there would not be noticed.

`go test -count=1 -coverprofile=c.out ./... && go tool cover -func=c.out` on `main` @ `f026f22`:

| Package / function | Coverage | What is untested |
|---|---|---|
| `cmd/relay` (`main`, `purgeLoop`, `runPurge`) | 0% | No tests. The middleware order (CORS → API key → rate limiter → router) is only tested in pieces, never as wired in `main` |
| `handler.HandleHealth` | 0% | Response shape, and that `/health` works without an API key |
| `middleware.APIKeyAuth` | 61.5% | No `apikey_test.go`. The empty-key passthrough, the `/health` exemption and the **valid-key** path are never executed. Only the `401` path runs, via `cors_test.go` |
| `handler.HandleCreate` | 60% | No test for `413` (body over `max_payload_size`), invalid JSON, a missing `to`/`from`/`payload` (`400`), or a store error (`500`) |
| `model.CreateMessageRequest.Validate` | 0% in its package (57% with `-coverpkg=./...`) | Only the success path runs, indirectly |
| `middleware.RateLimiter.cleanup` | 0% | Also no assertion on `Retry-After` or on the counter resetting after the window |
| `store.NewSQLiteStore` | 53.8% | Directory-creation and ping failures |

**Proposal**

1. **Prerequisite:** `.gitignore` line 34 (`relay`) matches the `cmd/relay/` directory, so a new `cmd/relay/main_test.go` is silently ignored by `git add`. Either change the pattern to `/relay` or add the file with `git add -f`. Otherwise the test never reaches the PR.
2. Move the router and middleware setup out of `main` into something like `func newHandler(cfg *config.Config, s store.Store) (http.Handler, func())`. Test it end to end with `httptest`: a preflight without a key gets `204`, a request without a key gets `401`, the request after the limit gets `429`, and `/health` is reachable without a key.
3. Make the purge interval injectable, so `purgeLoop` can be tested in milliseconds, including stopping on context cancel.
4. Add `internal/middleware/apikey_test.go` (no key configured, missing key, wrong key, correct key, `/health` exemption) and `internal/handler/health_test.go`.
5. Add table tests for the `HandleCreate` error paths, using a failing `store.Store` stub for the `500` case.
6. Optionally add `Retry-After` and window-reset tests for the rate limiter, with an injectable clock.

## Additional Context

- No CI workflow runs `go test` today, so these tests would also need to be run in CI to protect anything.
- Environment: Linux, Go 1.24.6 (`go.mod` says `go 1.23.12`). `go test -race ./...` passes on `main`.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
