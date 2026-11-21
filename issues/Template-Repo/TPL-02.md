# [BUG]: `.gitignore` is GitHub's LaTeX template, so `.env`, `node_modules/` and build output are not ignored in generated repos

## Bug Description

The template's `.gitignore` has not changed since the initial commit (`78c27b7`). It is a copy of GitHub's `TeX.gitignore`: line 1 reads `## Core latex/pdflatex auxiliary files:` and all 326 lines are TeX/LaTeX rules. Every repository created from this template inherits it.

This causes two problems:

1. **Common secret and dependency paths are not ignored.** `.env`, `.env.local`, `node_modules/`, `dist/`, `build/`, `coverage/`, `__pycache__/`, `venv/`, `.venv/`, `.idea/`, `.DS_Store`, `Thumbs.db` and `.vscode/settings.json` are all tracked by default. Meanwhile, the template tells people to create exactly these files:
   - `README.md:220-228` ("Configure Environment Variables") says "Create a `.env` file in the root directory" with `API_KEY=` / `DATABASE_URL=`.
   - `.vscode/settings.example.json` is meant to be copied to a local `settings.json`.
   - `skills/security-review/SKILL.md` saves unremediated findings to `unremediated-security-reviews/`, which "must never be committed to a public repository". The skill adds the ignore rule itself when it runs, but the template does not ship that rule.

   A contributor who follows the README and runs `git add .` or `git add -A` stages their `.env`.

2. **Some ordinary source files are ignored.** The TeX rules match file types that are not LaTeX output, for example `*.[1-9]` (man pages such as `man/tool.1`, or a file named `v1.2.3`), `*.el` (Emacs Lisp), `*.md5`, `*.idx`, `*.table`, `*.loc`, `*.bak` and `*.out` (also Go's `coverage.out`).

Generated repos either have to notice this and replace the file by hand, or they keep it. Among the 24 AOSSIE-Org repositories created from this template since December 2025, 18 still start with the LaTeX header, and 12 of those 18 have no `.env` rule at all. ThruBox-Server (`633fec2`, "fix: replace LaTeX .gitignore with Go-specific patterns") and ThruBox-Client (`4c2e91f`, "fix: replace LaTeX .gitignore with Node.js/TypeScript patterns") replaced it by hand. So did IndexedDB-Import-Export (`c8de460`).

## Steps to Reproduce

```bash
git clone https://github.com/AOSSIE-Org/Template-Repo.git && cd Template-Repo
head -1 .gitignore          # ## Core latex/pdflatex auxiliary files:

mkdir -p node_modules/x dist __pycache__ .venv unremediated-security-reviews man src
echo 'API_KEY=secret' > .env
touch .env.local node_modules/x/index.js dist/app.js __pycache__/m.pyc .venv/pyvenv.cfg \
      .DS_Store .vscode/settings.json unremediated-security-reviews/r.md man/tool.1 src/init.el

for f in .env .env.local node_modules/x/index.js dist/app.js __pycache__/m.pyc .venv/pyvenv.cfg \
         .DS_Store .vscode/settings.json unremediated-security-reviews/r.md man/tool.1 src/init.el; do
  git check-ignore -q "$f" && echo "ignored     $f" || echo "NOT ignored $f"
done

git add -A && git diff --cached --name-only   # .env, .env.local, node_modules/..., etc. are staged
git reset -q
```

## Logs and Screenshots

Output on `main` (`89ae62d`):

```
NOT ignored .env
NOT ignored .env.local
NOT ignored node_modules/x/index.js
NOT ignored dist/app.js
NOT ignored __pycache__/m.pyc
NOT ignored .venv/pyvenv.cfg
NOT ignored .DS_Store
NOT ignored .vscode/settings.json
NOT ignored unremediated-security-reviews/r.md
ignored     man/tool.1
ignored     src/init.el
```

`git check-ignore -v` attributes the last two to `.gitignore:100:*.[1-9]` and `.gitignore:309:*.el`.

**Expected:** a template meant for any stack (the README lists Node.js / Python / Flutter) ignores secrets, dependencies, build output, OS/editor files and the security-review output folder, and does not ignore normal source files.

**Actual:** only LaTeX artifacts are ignored. `.env` and dependency/build folders get committed unless each generated repo rewrites `.gitignore`.

## Environment Details

- `AOSSIE-Org/Template-Repo` `main` @ `89ae62d`; `.gitignore` blob `9308a4b` (unchanged since the initial commit)
- git 2.x on Linux; the behaviour is the same on any OS
- Partial mitigations exist, but they detect secrets rather than prevent commits: the `detect-secrets` pre-commit hook runs only if a contributor installs pre-commit, and `gitleaks-scanning.yml` only runs after a push.

## Impact

Medium - Feature works but has issues

## Proposed fix

Replace `.gitignore` with a short, language-neutral baseline and drop the TeX rules, for example:

```gitignore
# Secrets / local config
.env
.env.*
!.env.example
.vscode/settings.json

# Security-review reports (see skills/security-review)
unremediated-security-reviews/

# Dependencies and build output
node_modules/
dist/
build/
coverage/
__pycache__/
*.py[cod]
.venv/
venv/

# Logs, OS and editor files
*.log
.DS_Store
Thumbs.db
.idea/
*.swp

# Stack-specific additions (uncomment as needed)
# Go:        /bin/  *.test  coverage.out
# Flutter:   .dart_tool/  .flutter-plugins*  /build/
# Solidity:  out/  cache/  artifacts/
```

Note: `.gitignore` is not listed in `.templatesyncignore`, and `actions-template-sync` pulls with `-X theirs` by default. So the next template-sync PR in derived repos would also propose this file. That fixes repos still on the LaTeX file, but it would overwrite repos that already wrote their own `.gitignore`. Maintainers may want to decide whether `.gitignore` should go into `.templatesyncignore` as part of the same change.
