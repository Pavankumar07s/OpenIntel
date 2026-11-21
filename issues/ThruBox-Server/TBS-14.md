# `.gitignore` entry `relay` silently ignores new files added under `cmd/relay/`

## Bug Description

`.gitignore` line 34 is the bare pattern `relay`. It was added in f4cb7bf ("untrack local relay-server binary") to ignore the binary that `go build ./cmd/relay` writes to the repo root. A gitignore pattern with no slash matches a file **or directory** of that name at any depth, so it also matches the `cmd/relay/` directory itself.

`cmd/relay/main.go` is already tracked, so changes to it still show up. Any **new** file in that directory is ignored, though: a test file, a helper split out of `main.go`, and so on. The same goes for any future package directory named `relay` (for example `internal/relay/`). Such files:

- do not appear in `git status`,
- are skipped without any warning by `git add .` / `git add -A`,
- are refused by `git add cmd/relay/<file>` unless you pass `-f`.

The result is that a contributor can build and test locally, commit, and push a change that is missing its new file. CI and fresh clones then fail to compile, or a new `_test.go` simply never reaches the PR.

## Steps to Reproduce

On a clean checkout of `main`:

```sh
printf 'package main\n\nfunc buildInfo() string { return "relay" }\n' > cmd/relay/version.go
sed -i 's/^func main() {/func main() {\n\t_ = buildInfo()/' cmd/relay/main.go

go build ./cmd/relay            # OK locally
git status --short              # only " M cmd/relay/main.go"; version.go is not listed
git add -A && git commit -m "split main.go"
git show --stat HEAD            # only cmd/relay/main.go is committed

git clone . /tmp/fresh && cd /tmp/fresh && go build ./cmd/relay
```

Expected: `cmd/relay/version.go` shows up as untracked and gets committed.
Actual: it is ignored, the commit leaves it out, and the fresh clone fails to build.

## Logs and Screenshots

```
$ git check-ignore -v cmd/relay/version.go cmd/relay/main_test.go internal/relay/x.go
.gitignore:34:relay	cmd/relay/version.go
.gitignore:34:relay	cmd/relay/main_test.go
.gitignore:34:relay	internal/relay/x.go

$ git add cmd/relay/main_test.go
The following paths are ignored by one of your .gitignore files:
cmd/relay
hint: Use -f if you really want to add them.

# fresh clone of the commit above
$ go build ./cmd/relay
# github.com/AOSSIE-Org/ThruBox-Server/cmd/relay
cmd/relay/main.go:23:6: undefined: buildInfo
```

## Environment Details

- `main` at f026f22
- git 2.51.0, Go 1.24.6 (go.mod: go 1.23.12), Linux
- This does not depend on the OS. The `.dockerignore` file has no `relay` entry, so a local `docker build .` still picks up the untracked file and hides the problem. Only clean checkouts (CI, other contributors) break.

## Impact

Low - Minor inconvenience

Nothing is broken today because `main.go` is the only file in `cmd/relay/`. But this is a silent trap for the next person who adds a file there, such as tests for `purgeLoop`/`runPurge` or a refactor of `main.go`.

## Proposed fix

Anchor the binary pattern to the repo root so it only matches the build output:

```diff
 # Build / Release
 dist/
-relay
+/relay
```

I checked this: after the change, `go build ./cmd/relay` output (`./relay`) is still ignored, and new files under `cmd/relay/` show up as untracked. If wanted, the other binary entries (`relay-server`, `relay-server.exe`) can be anchored the same way (`/relay-server`, `/relay-server.exe`). Optionally add `/cmd/relay/relay` too, for anyone who runs `go build` from inside `cmd/relay/`.
