# [BUG]: Docs: every CONTRIBUTING.md table-of-contents link is broken, U+FFFD characters in headings, broken PR-template link, and dead links in `initial-issues.json`

## Bug Description

Every repo created from the template inherits these doc problems. All of them are still present in IndexedDB-Import-Export, ThruBox-Server and ThruBox-Client.

1. **All 8 CONTRIBUTING.md table-of-contents links are broken.** The TOC links to `#how-can-i-contribute`, `#coding-with-ai`, and so on. Each heading starts with an emoji (`## 🤝 How Can I Contribute?`), so GitHub's anchors begin with a hyphen: `-how-can-i-contribute`, `-coding-with-ai`, `-getting-started`, … Clicking a TOC entry does nothing.
2. **U+FFFD replacement characters** (bytes `EF BF BD`, rendered as `�`) appear in `CONTRIBUTING.md:7` (`## � IMPORTANT: Discord Communication is Mandatory`) and `CONTRIBUTING.md:19` (`## �📋 Table of Contents`). `README.md:189` has the same problem, but open PR #167 already fixes that line.
3. **The PR template's CONTRIBUTING link is broken.** `.github/PULL_REQUEST_TEMPLATE.md:20` uses `[Contributing Guidelines](./CONTRIBUTING.md)`. GitHub renders the href as `./CONTRIBUTING.md`, so on a PR page it resolves to `/AOSSIE-Org/Template-Repo/pull/CONTRIBUTING.md`. That redirects (302) to `/pull/new/CONTRIBUTING.md`, which tries to open a PR from a branch called `CONTRIBUTING.md`.
4. **CONTRIBUTING.md shows a different PR template.** The "PR Description Template" block (`CONTRIBUTING.md:197-218`) uses `## Description` / `## Related Issue` and other checklist items. The real `.github/PULL_REQUEST_TEMPLATE.md` uses `### Addressed Issues:`, and `dangerfile.js` checks for that heading and those checklist items. A PR that follows the CONTRIBUTING sample gets Danger's "PR Template Check" warnings.
5. **Dead links**
   - `.github/initial-issues.json`, in issues that get copied into every repo where "Create Initial Issues" runs:
     - `https://github.com/AOSSIE-Org/Template-Repo/blob/main/README_TEMPLATE.md` (404), in 4 "Documentation:" issues. No such file exists on `main`.
     - `https://github.com/StabilityNexus/Info/blob/main/KYA.md` (404)
     - `https://wagmi.sh/react/hooks/useNetwork` (404; `useNetwork` was removed in wagmi v2)
     - `https://www.freecodecamp.org/news/error-handling-in-javascript/` (404)
     - `https://www.honeycomb.io/blog/structured-logging-and-your-team` (404)
   - `BestPracticesChecklist.md:195`: `https://www.keylength.com/en/4/` (404)

## Steps to Reproduce

```bash
# 1. GitHub's real anchors for the CONTRIBUTING headings
curl -s https://github.com/AOSSIE-Org/Template-Repo/blob/main/CONTRIBUTING.md | grep -o '"anchor":"[^"]*"' | head -12
#  "anchor":"-important-discord-communication-is-mandatory"
#  "anchor":"-table-of-contents"
#  "anchor":"-how-can-i-contribute"
#  ...

# 2. U+FFFD bytes
grep -n $'\xef\xbf\xbd' CONTRIBUTING.md README.md

# 3. PR template link as rendered on an existing PR
gh api repos/AOSSIE-Org/Template-Repo/pulls/174 -H "Accept: application/vnd.github.html+json" --jq .body_html | grep -o '<a[^>]*>Contributing Guidelines'
#  <a href="./CONTRIBUTING.md">Contributing Guidelines
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" https://github.com/AOSSIE-Org/Template-Repo/pull/CONTRIBUTING.md
#  302 https://github.com/AOSSIE-Org/Template-Repo/pull/new/CONTRIBUTING.md

# 5. dead links
for u in https://github.com/AOSSIE-Org/Template-Repo/blob/main/README_TEMPLATE.md \
         https://github.com/StabilityNexus/Info/blob/main/KYA.md \
         https://wagmi.sh/react/hooks/useNetwork \
         https://www.freecodecamp.org/news/error-handling-in-javascript/ \
         https://www.honeycomb.io/blog/structured-logging-and-your-team \
         https://www.keylength.com/en/4/; do
  curl -sL -o /dev/null -w "%{http_code} $u\n" "$u"; done
```

## Logs and Screenshots

See the command outputs above.

## Environment Details

- `main` at `89ae62d`
- Checked against github.com with `curl` and `gh api`

## Impact

Low - Minor inconvenience

## Proposed fix

1. Change the TOC links to match GitHub's anchors (`#-how-can-i-contribute`, …), or move the emoji out of the heading text.
2. Delete the U+FFFD characters in `CONTRIBUTING.md:7` and `:19`.
3. Change the PR template link to an absolute or repo-relative URL that works from a PR page, for example `../blob/HEAD/CONTRIBUTING.md`, or the full `https://github.com/<org>/<repo>/blob/main/CONTRIBUTING.md` (which then becomes a placeholder to replace).
4. Replace the "PR Description Template" block in CONTRIBUTING.md with a short pointer to `.github/PULL_REQUEST_TEMPLATE.md`, so there is only one template.
5. In `initial-issues.json`, change `README_TEMPLATE.md` to `README.md`, and replace or remove the other dead links. Update or drop the keylength.com link in `BestPracticesChecklist.md`.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
