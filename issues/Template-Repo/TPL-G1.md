# [BUG]: LICENSE is an incomplete, edited copy of GPL-3.0 that stops at Section 3, so GitHub and Scorecard don't recognize it

## Bug Description

The README says the project is under GPL v3.0 (`README.md:274`), and GPL v3 was the license asked for in #2. But `LICENSE` holds only part of the GPL-3.0 text. It stops at the heading of Section 3, followed by a placeholder:

```
3. Protecting Users' Legal Rights From Anti-Circumvention Law.

... (Full GPLv3 text continues) ...

For the full, unabridged GNU General Public License version 3 text,
see https://www.gnu.org/licenses/gpl-3.0.txt
```

The file has 1,320 words. The official text has 5,644. The part that is there has also been edited, although the header says "changing it is not allowed":

- line 42-43: "modify the software." (official: "modify it.")
- line 140: "software packages that are used unmodified" (official: "programs which are used unmodified")
- Section 1 is missing its last two paragraphs ("The Corresponding Source need not include anything that users can regenerate automatically…" and "The Corresponding Source for a work in source code form is that same work."), and Section 2 has only its first paragraph.

So Sections 3-17 are missing, including the warranty disclaimer and the limitation of liability. GPL-3.0 Section 4 also requires giving recipients a copy of the License, and this file is not that copy.

The truncated text came in with 3fd9900 (part of #2), which replaced the earlier MIT `LICENSE`.

Effects:
- GitHub can't identify the license. `repos/AOSSIE-Org/Template-Repo/license` returns `key: other`, `spdx_id: NOASSERTION`.
- The OpenSSF Scorecard workflow's License check gives 9/10 with `Warn: project license file does not contain an FSF or OSI license.` `BestPracticesChecklist.md:6-8` leaves the License criterion to Scorecard, so this warning is what that criterion actually gets.
- Generated repos inherit the file. 20 repos generated from this template have exactly the same blob (`e00fcc6`) and all show NOASSERTION. Some examples are IndexedDB-Import-Export, ThruBox-Server, ThruBox-Client, OrgExplorer, SmartNotes, Skills and MCP-Template. OpenVerifiableLLM already replaced its copy (AOSSIE-Org/OpenVerifiableLLM#243) and is now detected as GPL-3.0.
- The truncated file is published on npm. `@aossie-org/idb-backup@1.1.0` and `@aossie-org/thrubox-client@1.1.0` declare `"license": "GPL-3.0"` but ship this file as `package/LICENSE`.
- `.templatesyncignore` lists `LICENSE`, and so do the derived repos' copies. Older generated repos have no template-sync at all. So fixing the file here will not reach existing repos automatically.

## Steps to Reproduce

```sh
git clone https://github.com/AOSSIE-Org/Template-Repo && cd Template-Repo
tail -n 5 LICENSE                                              # placeholder shown above
wc -w < LICENSE                                                # 1320
curl -s https://www.gnu.org/licenses/gpl-3.0.txt | wc -w       # 5644

gh api repos/AOSSIE-Org/Template-Repo/license -q '.license.spdx_id'   # NOASSERTION
for r in IndexedDB-Import-Export ThruBox-Server ThruBox-Client; do
  gh api repos/AOSSIE-Org/$r/license -q '[.license.spdx_id, .sha] | @tsv'
done   # NOASSERTION  e00fcc6c5d138fe0244500e8ad59055a6d6e4049 (x3)

npm pack @aossie-org/idb-backup@1.1.0
tar -xzf aossie-org-idb-backup-1.1.0.tgz package/LICENSE -O | grep -c 'Full GPLv3 text continues'   # 1
```

**Expected:** `LICENSE` is the verbatim GPL-3.0 text. GitHub reports `GPL-3.0`, and Scorecard's License check reports no warning.

**Actual:** the text is truncated and edited. GitHub reports `NOASSERTION`, and Scorecard warns that the license is not FSF/OSI.

## Logs and Screenshots

Scorecard run 43575834893 ("Scorecard supply-chain security", main @ 89ae62d), License check:

```
{"details":["Info: project has a license file: LICENSE:0","Warn: project license file does not contain an FSF or OSI license."],"score":9,"reason":"license file detected","name":"License", ...}
```

The same result appears in ThruBox-Server run 43575826135, IndexedDB-Import-Export run 43575896461 and ThruBox-Client run 43575701389.

## Environment Details

- Template-Repo `main` at 89ae62d (`LICENSE` blob e00fcc6, unchanged since 3fd9900)
- GitHub REST API `/repos/{owner}/{repo}/license`, gh CLI
- npm registry, packages `@aossie-org/idb-backup@1.1.0` and `@aossie-org/thrubox-client@1.1.0`

## Impact

Medium - Feature works but has issues

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates

## Proposed fix

1. Replace `LICENSE` with the unmodified https://www.gnu.org/licenses/gpl-3.0.txt. Its git blob is `f288702`, the same file GitHub's GPL-3.0 template uses. This is the change OpenVerifiableLLM#243 made.
2. `LICENSE` is in `.templatesyncignore` on purpose, because a generated repo may choose another license, so leave that as it is. Instead, add a short note (README or release notes) asking repos generated before the fix to copy the new `LICENSE`. The npm packages pick it up on their next release.
