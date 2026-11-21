# [BUG]: Scorecard push trigger targets a `dev` branch that doesn't exist; stack-specific CodeRabbit JS/TS rules and a few template leftovers

## Bug Description

These are small, independent problems that every new repo inherits. The first two change behavior; the rest are leftover text.

1. **Scorecard never runs on push.** `.github/workflows/scorecard.yml:17` has `push: branches: ["dev"]`. The template's default branch is `main` and there is no `dev` branch. The job condition (`:28-32`) also requires `default_branch == ref_name`, so push-triggered scans can't happen. All 26 Scorecard runs in Template-Repo's history were triggered by `schedule`; none by `push`. IndexedDB-Import-Export already changed this to `["main"]`. ThruBox-Server and ThruBox-Client still have `["dev"]`. The same condition checks `github.event_name == 'pull_request'`, which is not one of the workflow's triggers.

2. **The CodeRabbit JS/TS instructions assume one stack but apply to every JS/TS file.** `.coderabbit.yaml` applies these to every `**/*.{ts,tsx,js,jsx}` / `**/*.test.{ts,tsx,js,jsx}` file:
   - `:187` "Ensure that \"use client\" is being used" (Next.js only, and wrong for server components, Node backends and libraries)
   - `:204` "Use expo-secure-store for sensitive storage" (Expo/React Native only)
   - `:267` "Proper use of @testing-library/react-native"

   IndexedDB-Import-Export, ThruBox-Server and ThruBox-Client all deleted these lines (`grep -c 'use client\|expo-secure-store\|testing-library/react-native' .coderabbit.yaml` returns 0 in each). Also, `request_changes_workflow: true` (`:82`) has the comment "Automatically request changes on the PR (just leave comments)", which contradicts itself.

3. **Leftover or incorrect text**
   - `.github/workflows/stale.yml:5`: the comment says "Runs every hour at minute 35", but the cron is `0 1 * * *` (daily at 01:00 UTC).
   - `.github/workflows/create-initial-issues.yml:35`: typo "hecne" (should be "hence").
   - Stability Nexus text in the AOSSIE template: `README.md:62` links `https://TODO.stability.nexus/` (line 15 uses `TODO.aossie.org`). Lines 113 and 116 ask for review "by at least two knights of The Stable Order" and for "the Stability Nexus's logo". `public/stability.svg` is not referenced anywhere in the repo.

## Steps to Reproduce

```bash
grep -n 'branches:' .github/workflows/scorecard.yml          # 17:    branches: ["dev"]
gh api repos/AOSSIE-Org/Template-Repo --jq .default_branch    # main
gh run list -R AOSSIE-Org/Template-Repo --workflow scorecard.yml --limit 200 --json event --jq 'group_by(.event)|map({(.[0].event): length})'
#   [{"schedule":26}]
grep -n 'use client\|expo-secure-store\|testing-library/react-native\|request_changes_workflow' .coderabbit.yaml
sed -n 4,7p .github/workflows/stale.yml
grep -rn 'stability' README.md; grep -rn 'stability.svg' . --exclude-dir=.git   # no references to the SVG
```

## Logs and Screenshots

N/A. See the command outputs above.

## Environment Details

- `main` at `89ae62d`
- Compared with IndexedDB-Import-Export, ThruBox-Server and ThruBox-Client on their default branches

## Impact

Low - Minor inconvenience

## Proposed fix

1. `scorecard.yml`: set `push: branches: ["main"]` (as IndexedDB-Import-Export did) and drop the `pull_request` clause from the job `if:`.
2. `.coderabbit.yaml`: replace the Next.js/Expo/React Native lines with stack-neutral wording, for example "If this is a Next.js app, check `use client` boundaries" and "store secrets with the platform's secure storage", or move them into commented-out, opt-in examples. Fix the `request_changes_workflow` comment to describe what the option does.
3. Fix the `stale.yml` comment and the "hecne" typo.
4. Replace the Stability Nexus references in README.md with AOSSIE equivalents, or turn them into generic TODOs, and delete the unused `public/stability.svg`.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
