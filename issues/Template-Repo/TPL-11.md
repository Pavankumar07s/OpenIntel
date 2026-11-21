# [BUG]: dependabot.yml gives every Dependabot PR a `chore(deps)(deps):` title, and no entry sets a `cooldown`

## Bug Description

One problem in `.github/dependabot.yml`, plus one hardening gap:

1. **Scope added twice in PR titles.** Every entry sets `commit-message.prefix: "chore(deps)"` together with `include: "scope"`. Dependabot appends the scope `(deps)` or `(deps-dev)` after the prefix, so the result is `chore(deps)(deps): bump …`. That is not a valid Conventional Commits header. Every Dependabot PR in this repo has this title, e.g. #168, #170, #175, #177, #178.
2. **No `cooldown`.** No entry sets `cooldown`, so Dependabot proposes a release on the day it is published. Supply-chain hardening guidance recommends waiting a few days before adopting new releases.

Related, not covered here: the 11 enabled ecosystems that have no manifest in the template root fail every week in each new repo until Dependabot pauses them after 15 failures (139–166 failed "Dependabot Updates" runs in Template-Repo, IndexedDB-Import-Export, ThruBox-Client and ThruBox-Server). How to enable only the ecosystems a project uses belongs to #72 (directory globs / auto-discovery), so that evidence is added there instead.

## Steps to Reproduce

1. Look at any Dependabot PR title in this repo or in a repo created from it: it starts with `chore(deps)(deps):`.
2. Search `.github/dependabot.yml` for `cooldown`: no entry sets it.

The title logic is in dependabot-core `common/lib/dependabot/pull_request_creator/pr_name_prefixer.rb`:

```ruby
prefix += "(#{scope})" if commit_message_options&.include_scope   # scope = "deps" / "deps-dev"
prefix += ":" if prefix.match?(/[A-Za-z0-9\)\]]\Z/)
```

With `prefix: "chore(deps)"` this gives `chore(deps)(deps): `. With `prefix: "chore"` it gives `chore(deps): ` (or `chore(deps-dev): `).

## Logs and Screenshots

Recent Dependabot PR titles: #168, #170, #175, #177 and #178 all start with `chore(deps)(deps):`.

## Environment Details

- `main` at 89ae62d, `.github/dependabot.yml` (unchanged since 2026-02-21)
- GitHub-hosted Dependabot version updates

## Impact

Low - Minor inconvenience

## Proposed fix

- Change every `prefix: "chore(deps)"` to `prefix: "chore"` and keep `include: "scope"`. Titles become `chore(deps): …` / `chore(deps-dev): …`.
- Add `cooldown:` with `default-days: 7` (or similar) to each entry. It passes the SchemaStore dependabot-2.0 schema. Security updates are not delayed by it.
- This edits the same file as the open grouping PR #172 (for #73), so the two changes should be coordinated to avoid conflicts. #172 only adds `groups` and does not fix either point above.
