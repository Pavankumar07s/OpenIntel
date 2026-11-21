<!-- Team note, not issue text: do NOT open a new issue for this. Post "Comment for #112" on #112 (assigned to kpj2006) and "Comment for #176" on #176. -->

# Danger "PR Template Check" comment: broken template link, flattened nested lists, and comments on Dependabot PRs

## Comment for #112

While looking at the "⚠️ PR Template Check" comment for this issue, I found two more problems in the same comment (`dangerfile.js`, `main` @ `89ae62d`). They are small and could go into the same fix, or into a separate small PR if you prefer:

1. **The "PR template" link is broken.** `dangerfile.js:44` writes `[PR template](.github/PULL_REQUEST_TEMPLATE.md)`. GitHub does not rewrite relative links in comments, so the rendered HTML is `<a href=".github/PULL_REQUEST_TEMPLATE.md">`. On a PR page the browser resolves that to `https://github.com/AOSSIE-Org/Template-Repo/pull/.github/PULL_REQUEST_TEMPLATE.md`, which redirects (302) to `/pull/new/.github%2FPULL_REQUEST_TEMPLATE.md`: a "new pull request" page for a branch that does not exist, not the template.
2. **Nested items render as a flat list.** Sub-items are written as `- ${item}` (`dangerfile.js:43` and `:79`) inside an outer `- ${issue}` list (`:89`), with no indent, so they render at the same level as their heading ("Some required checklist items are not completed:" and "My PR addresses a single issue" appear as sibling bullets).

Repro:

```sh
# rendered link in the comment on #178
gh api -H "Accept: application/vnd.github.full+json" \
  repos/AOSSIE-Org/Template-Repo/issues/178/comments \
  --jq '.[] | select((.body//"")|test("PR Template Check")) | .body_html' \
  | grep -o '<a [^>]*>PR template</a>'
# <a href=".github/PULL_REQUEST_TEMPLATE.md">PR template</a>

curl -s -o /dev/null -w '%{http_code} -> %{redirect_url}\n' \
  https://github.com/AOSSIE-Org/Template-Repo/pull/.github/PULL_REQUEST_TEMPLATE.md
# 302 -> https://github.com/AOSSIE-Org/Template-Repo/pull/new/.github%2FPULL_REQUEST_TEMPLATE.md

# how the list renders
printf -- '- Some required checklist items are not completed:\n- My PR addresses a single issue\n' \
  | gh api -X POST /markdown -f mode=gfm -F text=@-
# <ul><li>Some required checklist items are not completed:</li><li>My PR addresses a single issue</li></ul>
```

Suggested fix:

- Build an absolute link from the PR's base repo and branch:
  ```js
  const templateUrl = `${danger.github.pr.base.repo.html_url}/blob/${danger.github.pr.base.ref}/.github/PULL_REQUEST_TEMPLATE.md`;
  ```
  For this repo that gives `https://github.com/AOSSIE-Org/Template-Repo/blob/main/.github/PULL_REQUEST_TEMPLATE.md`, which returns 200.
- Indent the sub-items and the "Please follow…" line at `dangerfile.js:43`, `:44` and `:79` with two spaces (`  - ${item}`) so they nest under their parent bullet.

## Comment for #176

Related data point: the check also runs on, and comments on, every Dependabot PR (for example #168, #177 and #178), telling the bot to add `### Addressed Issues:` and `## Checklist`, link an issue and tick three checklist items. The job's `if:` in `.github/workflows/danger.yml` only checks `github.repository_owner`, so it runs no matter who opened the PR. Skipping bot-authored PRs could go together with the concurrency change here:

```yaml
if: ${{ github.repository_owner == 'AOSSIE-Org' && github.event.pull_request.user.type != 'Bot' }}
```
