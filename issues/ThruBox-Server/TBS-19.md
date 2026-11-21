# [BUG]: Docs are out of date: "no tests yet" notes, CORS comments, AGENTS.md layout, missing security notes

## Bug Description

Several documentation statements on `main` (`f026f22`) no longer match the code:

1. **"There are no tests" appears in three places**, but the repo has 54 tests (5 packages, 64.8% statement coverage, all passing):
   - `AGENTS.md:32`: "there are no `_test.go` files in the repository yet".
   - `CONTRIBUTING.md:137`: "The repository doesn't have test files yet".
   - `BestPracticesChecklist.md:145`, `:148`, `:156`, `:246`: `test_invocation` says "no `_test.go` files exist in the repo yet", `test_most` says "0% — no tests exist yet", and `tests_are_added` says "no tests exist in the repo". PRs #35, #36, #67 and #70 all added tests, so whether these items count as met should be looked at again.
2. **CORS middleware comments.** `internal/middleware/cors.go:29` ("OPTIONS keeps 404ing from the router") and `:73` ("a confusing 404 from the router") are wrong for the real routes. The router answers `405 Method Not Allowed` with an `Allow` header (see logs below).
3. **`AGENTS.md` "Repository Layout"** lists only `cmd/relay/`, `public/` and `brand/`. It leaves out `internal/` (config, handler, middleware, model, store), where almost all the code lives, as well as `audit/` and `skills/`.
4. **No security notes for people building on the API.** The README never says:
   - that `from` is stored exactly as the sender supplied it and is not authenticated, so applications must verify the sender inside the encrypted payload;
   - that on a relay without `api_key`, anyone who knows an address can read its ciphertext and metadata (who sent messages, when, and how often).

   The repo's own review (`audit/sec_review_2026-09-23T10-33-16Z.md`, Note 1 and "Review-specific limitations") recommends documenting both.

(The "max payload" wording, the rate limiter's "sliding window" comment and the undocumented `storage.driver` key are left out here. They belong with the related code fixes. The Go version in the README/CONTRIBUTING prerequisites is updated together with the Go toolchain upgrade, so it is left out too.)

## Steps to Reproduce

```sh
go test -count=1 -cover ./...                  # 5 packages ok, tests exist
grep -n "no \`_test.go\` files" AGENTS.md
grep -n "doesn't have test files" CONTRIBUTING.md
grep -n "no tests exist" BestPracticesChecklist.md

go build -o relay-server ./cmd/relay && RELAY_SERVER_PORT=3000 ./relay-server &
curl -si -X OPTIONS http://localhost:3000/api/messages | head -2
```

## Logs and Screenshots

```
HTTP/1.1 405 Method Not Allowed
Allow: POST
```

## Environment Details

- ThruBox-Server `main` @ `f026f22` (`VERSION` 1.0.0), Linux, Go 1.24.6.

## Impact

Low - Minor inconvenience

**Proposed fix:** one docs PR that:

- removes the "no tests" notes from `AGENTS.md` and `CONTRIBUTING.md`;
- updates the evidence text in `BestPracticesChecklist.md`, leaving the checkbox decisions to maintainers;
- corrects the two `cors.go` comments to say "405 from the router";
- adds `internal/`, `audit/` and `skills/` to the `AGENTS.md` layout;
- adds a short "Security notes" section to the README.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
