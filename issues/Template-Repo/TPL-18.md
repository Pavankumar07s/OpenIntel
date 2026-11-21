# [BUG]: The template fails its own pre-commit hooks (30 files rewritten, `check-json` fails on `.vscode/extensions.json`), and no CI job runs them

## Bug Description

CONTRIBUTING.md tells contributors to run `pre-commit install`, and it has a long "Debugging Pre-commit Hooks" section. But the repo does not pass its own `.pre-commit-config.yaml`:

- `trailing-whitespace` and `end-of-file-fixer` rewrite **30 files** (222 insertions, 222 deletions). Most of it is in `.github/workflows/setup-labels.yml` (216 changed lines in `git diff --stat`), `sync-pr-labels.yml` (94), `create-initial-issues.yml` (24) and `PRIVACY.md` (22).
- `check-json` fails on `.vscode/extensions.json`, because that file uses `//` comments (JSONC).
- `pre-commit/pre-commit-hooks` is declared twice with the same `rev` (`.pre-commit-config.yaml` lines 18 and 31).

pre-commit only checks staged files. So a contributor who has the hooks installed and edits one of these files (for example to change one label in `setup-labels.yml`) has their commit blocked. The hook then rewrites dozens of unrelated lines, which end up in their diff.

No workflow runs pre-commit (`grep -rn pre-commit .github/` finds nothing), and PR checks show no pre-commit.ci app. So nothing stops this drift from coming back.

## Steps to Reproduce

```bash
git clone https://github.com/AOSSIE-Org/Template-Repo && cd Template-Repo
pip install pre-commit
pre-commit run --all-files
git diff --shortstat
```

## Logs and Screenshots

```
trim trailing whitespace.................................................Failed
- hook id: trailing-whitespace
- exit code: 1
- files were modified by this hook
...
fix end of files.........................................................Failed
- hook id: end-of-file-fixer
- exit code: 1
- files were modified by this hook
...
check yaml...............................................................Passed
check json...............................................................Failed
- hook id: check-json
- exit code: 1

.vscode/extensions.json: Failed to json decode (Expecting value: line 3 column 40 (char 64))

check toml...........................................(no files to check)Skipped
Detect secrets...........................................................Passed

$ git diff --shortstat
 30 files changed, 222 insertions(+), 222 deletions(-)
```

## Environment Details

- `main` at `89ae62d`
- pre-commit 4.7.0, Python 3, Linux
- Hook revisions from the repo config: `pre-commit-hooks` v4.5.0, `detect-secrets` v1.4.0

## Impact

Low - Minor inconvenience

## Proposed fix

1. Run `pre-commit run --all-files` once and commit the whitespace/EOF fixes in their own PR, with no other changes. The trailing spaces in `PRIVACY.md` are single spaces, not Markdown hard breaks, so removing them doesn't change the rendering.
2. Exclude JSONC files from `check-json`, for example `exclude: ^\.vscode/.*\.json$`, or switch those files to `check-json5`.
3. Merge the two `pre-commit/pre-commit-hooks` blocks into one.
4. Add a small CI job so the template stays clean:
   ```yaml
   - uses: actions/checkout@v7
   - uses: actions/setup-python@v7
   - uses: pre-commit/action@v3.0.1
   ```

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
