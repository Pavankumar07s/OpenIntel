# [BUG]: Best Practices scorer ignores the inline "`[ ]` N/A" boxes, counts checkboxes outside the six categories, and commits a date-only change on most runs

## Bug Description

I found three problems in the `update-score` job of `.github/workflows/checklist-score.yml`, in the "Parse checklist and compute score" step. #110 (dynamic `CATEGORY_TOTALS`) covers none of them.

1. **The inline N/A boxes are never counted.** The legend says to mark N/A items with a leading `- [~]`. But 21 sub-bullets also show their own N/A box, for example:
   ```
   - [ ] 🔴 **documentation_basics** — ...
     - *Evidence URL:* `[ ]` N/A — *Justification:*
   ```
   The scorer only matches `- [x]`, `- [X]` or `- [~]` at the start of a line (line 62). A maintainer who ticks the inline box (`` `[x]` N/A ``), as the hint invites, still gets 0 for that item.

2. **The category is never reset.** `current_cat` (line 52) is set when a known category heading is found, but no other `## ` heading clears it. Everything after `## 🔬 Analysis` counts toward Analysis, including `## 📎 Project-Specific Notes`, which asks maintainers to "Add domain-specific notes here". Checkbox notes added there push Analysis, and the overall score, above 100%. No cap is applied either.

3. **Every run on a new day commits.** `"updated": str(date.today())` (line 83) changes on every new day, so `git diff --staged` is never empty and the bot pushes a commit even when no count changed. In IndexedDB-Import-Export, ThruBox-Server and ThruBox-Client, the second commit is `chore: update best practices score [skip ci]`, and each score commit I checked changed only that field:
   ```
   IndexedDB-Import-Export 1604c7c   -  "updated": "2026-05-25",  +  "updated": "2026-05-28",
   ThruBox-Client          e4ce688   -  "updated": "2026-05-25",  +  "updated": "2026-07-11",
   ThruBox-Client          9c5d565   -  "updated": "2026-08-12",  +  "updated": "2026-09-23",
   ThruBox-Server          8a9a4c5   -  "updated": "2026-10-26",  +  "updated": "2026-10-29",
   IndexedDB-Import-Export bc4b140   -  "updated": "2026-08-25",  +  "updated": "2026-09-23",
   ```

## Steps to Reproduce

Extract the step's Python and run it on modified copies of the checklist:

```bash
python3 - <<'EOF'
import yaml
wf = yaml.safe_load(open('.github/workflows/checklist-score.yml'))
run = wf['jobs']['update-score']['steps'][1]['run']
open('/tmp/score.py', 'w').write(run.split("<< 'EOF'\n", 1)[1].rsplit("EOF", 1)[0])
EOF

# (1) tick only the inline N/A boxes (21 of them)
mkdir -p /tmp/s1 && sed 's/`\[ \]` N\/A/`[x]` N\/A/' BestPracticesChecklist.md > /tmp/s1/BestPracticesChecklist.md
(cd /tmp/s1 && python3 /tmp/score.py)        # Score: 0/49 (0%)

# (2) tick every item, then add checkbox notes under "Project-Specific Notes"
mkdir -p /tmp/s2 && sed 's/^- \[ \]/- [x]/' BestPracticesChecklist.md > /tmp/s2/BestPracticesChecklist.md
printf '\n- [x] Slither runs in CI\n- [x] ZAP baseline scan\n- [x] Prompt-injection tests\n' >> /tmp/s2/BestPracticesChecklist.md
(cd /tmp/s2 && python3 /tmp/score.py && jq -c '.categories.analysis' checklist-status.json)
# Score: 52/49 (106%)   {"met":10,"total":7}
```

(3) Run the workflow twice on different days with no checklist change. The second run still commits.

## Logs and Screenshots

See the outputs and the commit diffs above.

## Environment Details

- `main` at `89ae62d`, `.github/workflows/checklist-score.yml` lines 52-63 and 83
- Same code in IndexedDB-Import-Export, ThruBox-Server and ThruBox-Client

## Impact

Low - Minor inconvenience

## Proposed fix

1. Reset `current_cat = None` on any line starting with `## ` that is not one of the six category headings.
2. Pick one N/A syntax. Either count an item as met when its sub-bullet contains `` `[x]` N/A `` (or `[~]`), or remove the inline `` `[ ]` N/A `` hints and keep only the leading `- [~]`.
3. Cap each category's `met` at its `total` (this pairs naturally with #110).
4. Only write a new `updated` date when `met` or any category count changes, for example by comparing with the existing `checklist-status.json` before writing.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
