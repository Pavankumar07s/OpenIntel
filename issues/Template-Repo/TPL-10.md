# [BUG]: OpenSSF criteria sync job never detects anything: upstream ID regex matches 0 criteria, yet the run passes

## Bug Description

The `sync-criteria` job in `.github/workflows/checklist-score.yml` runs every Monday. It is supposed to compare `BestPracticesChecklist.md` with the upstream OpenSSF Best Practices criteria and open a "OpenSSF Criteria Update" PR when they differ. It has never compared anything.

- Line 200 extracts upstream criterion IDs with `\[([a-z][a-z0-9_]+)\]\(#\1\)`, which matches Markdown links like `[id](#id)`. The upstream `docs/criteria.md` doesn't use that format. It marks each criterion with an HTML anchor, `<a name="description_good"></a>`, and the reference is also HTML: `<sup>[<a href="#description_good">description_good</a>]</sup>`. The regex therefore finds **0** IDs.
- Lines 203-205 then print `⚠️ Warning: No criteria IDs found in upstream. Format may have changed.` and call `sys.exit(0)`, so every run is reported as **success**.

I checked the upstream file at commits `97734a51f2` (2024-12), `feaa41d84b` (2026-02) and `2fc9ac1525` (2026-06, current). All three have 68 `<a name="…">` anchors and 0 matches for the current regex. The job has been a no-op since it was added in #108.

Minor related points:
- The first URL (`…/main/docs/criteria/criteria.md`) returns 404 on every run. The fallback (`…/main/docs/criteria.md`) returns 200, so this is noise rather than a failure. The upstream org has also been renamed from `coreinfrastructure` to `ossf`. The old raw URL still resolves today, but `ossf/best-practices-badge` is the canonical location.
- If the regex is fixed as-is, the diff is not empty: 19 upstream passing-level IDs are missing from the checklist (list below). Because the branch name is `chore/openssf-sync-<date>` and the "PR already exists" check looks only at that branch, the job would then open a **new PR every week** until the checklist or an exclusion list accounts for those IDs. A second run on the same day would also fail at `git push`, because the branch already exists on the remote with a different commit.

## Steps to Reproduce

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  https://raw.githubusercontent.com/coreinfrastructure/best-practices-badge/main/docs/criteria/criteria.md   # 404
curl -sf -o upstream_criteria.md \
  https://raw.githubusercontent.com/coreinfrastructure/best-practices-badge/main/docs/criteria.md            # 200

python3 - <<'PY'
import re
up  = open("upstream_criteria.md").read()
loc = open("BestPracticesChecklist.md").read()   # run from a Template-Repo checkout
print("current regex:", len(set(re.findall(r'\[([a-z][a-z0-9_]+)\]\(#\1\)', up))))
anchors = set(re.findall(r'<a name="([a-z0-9_]+)"', up))
local   = set(re.findall(r'\*\*([a-z][a-z0-9_]+)\*\*', loc))
print("anchor IDs:", len(anchors), "local IDs:", len(local))
print("missing locally:", sorted(anchors - local))
PY
```

Output on `main` (`89ae62d`):

```
current regex: 0
anchor IDs: 68 local IDs: 49
missing locally: ['crypto_floss', 'crypto_pfs', 'crypto_weaknesses', 'delivery_mitm', 'floss_license', 'floss_license_osi', 'homepage_url', 'license_location', 'maintained', 'no_leaked_credentials', 'repo_interim', 'repo_public', 'repo_track', 'sites_https', 'static_analysis', 'test', 'test_continuous_integration', 'vulnerabilities_critical_fixed', 'vulnerabilities_fixed_60_days']
```

Running the workflow's own Python step unchanged against the upstream file prints only the warning and exits 0.

## Logs and Screenshots

All 27 scheduled "Best Practices" runs since 2026-05-18 finished with `success`. Every `sync-criteria` log still available (2026-09-14 through 2026-11-16) contains the same line. For example, run 43837059874 (2026-11-16):

```
2026-11-16T09:08:28.9687920Z ⚠️ Warning: No criteria IDs found in upstream. Format may have changed.
```

No "OpenSSF Criteria Update" PR and no `chore/openssf-sync-*` branch has ever been created.

## Environment Details

- Workflow: `.github/workflows/checklist-score.yml`, job `sync-criteria` (lines 160-256), `main` @ `89ae62d`
- Runner: `ubuntu-latest`, triggered by the `schedule` event (Mondays 09:00 UTC)
- Only runs in `AOSSIE-Org/Template-Repo` (`if: github.repository == 'AOSSIE-Org/Template-Repo'`). Derived repos are not affected.

## Impact

Low - Minor inconvenience

The drift check passes every week without checking anything, so changes in the upstream criteria are never surfaced. Some of the 19 missing IDs fall under areas the checklist header delegates to Scorecard: license, maintained, SAST, CI tests, vulnerabilities. Others are not Scorecard checks and are left out without a documented reason: `delivery_mitm`, `no_leaked_credentials`, `crypto_weaknesses`, `crypto_pfs`, `crypto_floss`, `homepage_url`, `repo_public`, `repo_track`, `repo_interim`, `sites_https`.

## Proposed fix

1. Parse upstream IDs from the anchors: `re.findall(r'<a name="([a-z0-9_]+)"', upstream_content)`.
2. Add an explicit, commented exclusion set (for example `INTENTIONALLY_EXCLUDED = {...}`) for criteria the checklist deliberately leaves to Scorecard or treats as met by GitHub hosting. Maintainers should decide which of the 19 IDs belong there and which should be added to the checklist.
3. Make "0 upstream IDs parsed" fail the job (`sys.exit(1)` / `::error::`) instead of exiting 0, so a future format change is visible.
4. Use a fixed branch name (for example `chore/openssf-criteria-sync`), force-push it, and update the existing open PR instead of opening a new one each week.
5. Drop the 404 URL and point the fetch at `https://raw.githubusercontent.com/ossf/best-practices-badge/main/docs/criteria.md`.

Note: `gh pr create` with `GITHUB_TOKEN` also needs "Allow GitHub Actions to create and approve pull requests" enabled for the repo/org. That couldn't be verified from outside, and it has never been exercised because the job never reached that step.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
