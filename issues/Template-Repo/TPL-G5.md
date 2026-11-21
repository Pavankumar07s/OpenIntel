# [BUG]: `.coderabbit.yaml` path_instructions `**/*.{py}` and `**/*.test.{sol}` never match any file, so the Python and Solidity-test rules never apply

## Bug Description

Two `reviews.path_instructions` globs in `.coderabbit.yaml` use braces around a single item:

- `.coderabbit.yaml:231`: `- path: "**/*.{py}"` (Python / PEP 8 rule)
- `.coderabbit.yaml:274`: `- path: "**/*.test.{sol}"` (Solidity test checklist)

The CodeRabbit docs (https://docs.coderabbit.ai/configuration/path-instructions) say "Glob matching follows `Minimatch` rules, including brace expansion". Minimatch only expands a brace set that has a comma or a range, like bash does. `{py}` and `{sol}` stay as literal text. So `**/*.{py}` only matches a file whose name really ends in `.{py}`, and `**/*.test.{sol}` only matches one ending in `.test.{sol}`. No real file matches either pattern, so neither block ever reaches CodeRabbit.

Two related problems with the Solidity test block:

- Even with the braces removed, `*.test.sol` is not the usual name for Solidity tests. Foundry, and Hardhat 3's Solidity tests, use `*.t.sol` (for example `test/Counter.t.sol`).
- `**/*.sol` (`:237`) matches every Solidity file, test contracts included. Test contracts therefore get only the production-contract checklist (gas optimisation, `immutable`/`constant`, event emission, unbounded loops, access control) and never the test checklist.

Both patterns came in with fec18e0. They are copied unchanged into repos created from this template, including Python repos such as Gitcord-GithubDiscordBot, WebSift, SkillBot and PythonScripts. In those repos the Python rule is dead today.

On #61, the CodeRabbit review flagged `**/*.test.{sol}` as an "unnecessary brace" and pointed out the Foundry `.t.sol` mismatch. That comment was not acted on, and the line is unchanged on `main`.

## Steps to Reproduce

1. Install the matcher the docs name (picomatch and micromatch shown for comparison):
   ```sh
   mkdir globtest && cd globtest && npm init -y >/dev/null
   npm i minimatch@10.0.1 picomatch@4.0.2 micromatch@4.0.8
   ```
2. Run:
   ```sh
   node -e '
   const {minimatch}=require("minimatch"), pm=require("picomatch"), mm=require("micromatch");
   for (const [p,f] of [
     ["**/*.{py}","src/app.py"], ["**/*.{py}","src/app.{py}"],
     ["**/*.test.{sol}","test/Vault.test.sol"], ["**/*.test.{sol}","test/Vault.t.sol"],
     ["**/*.sol","test/Vault.t.sol"],
     ["**/*.{ts,tsx,js,jsx}","src/a.ts"], ["**/*.py","src/app.py"], ["**/*.t.sol","test/Vault.t.sol"]])
     console.log(p.padEnd(22), f.padEnd(22), "minimatch="+minimatch(f,p), "picomatch="+pm.isMatch(f,p), "micromatch="+mm.isMatch(f,p));
   console.log(JSON.stringify(minimatch.braceExpand("**/*.{py}")));'
   ```

## Logs and Screenshots

```
**/*.{py}              src/app.py             minimatch=false picomatch=false micromatch=false
**/*.{py}              src/app.{py}           minimatch=true picomatch=true micromatch=true
**/*.test.{sol}        test/Vault.test.sol    minimatch=false picomatch=false micromatch=false
**/*.test.{sol}        test/Vault.t.sol       minimatch=false picomatch=false micromatch=false
**/*.sol               test/Vault.t.sol       minimatch=true picomatch=true micromatch=true
**/*.{ts,tsx,js,jsx}   src/a.ts               minimatch=true picomatch=true micromatch=true
**/*.py                src/app.py             minimatch=true picomatch=true micromatch=true
**/*.t.sol             test/Vault.t.sol       minimatch=true picomatch=true micromatch=true
["**/*.{py}"]
```

`minimatch.braceExpand` returns `**/*.{py}` unchanged. Bash gives the same result: `echo a.{py}` prints `a.{py}`.

**Expected:** `**/*.{py}` matches `.py` files and the Solidity test block matches Solidity test files.
**Actual:** neither block matches any real file, so CodeRabbit never applies them.

## Environment Details

- `main` at 89ae62d; `.coderabbit.yaml` last changed in dc7e2d1
- Node.js v24, minimatch 10.0.1, picomatch 4.0.2, micromatch 4.0.8

## Impact

Low - Minor inconvenience

## Proposed fix

```diff
     # Python files
-    - path: "**/*.{py}"
+    - path: "**/*.py"
@@
     # Solidity test files
-    - path: "**/*.test.{sol}"
+    - path: "**/*.{t,test}.sol"
```

Optionally, add `test/**/*.sol` as a second entry with the same instructions, for test contracts without a `.t.sol` suffix. Also add a line to the `**/*.sol` block saying that test contracts are covered by the Solidity test block, so the production-only checks (gas, immutability, event emission) are not pushed onto test code. Derived repos would pick this up through template sync.
