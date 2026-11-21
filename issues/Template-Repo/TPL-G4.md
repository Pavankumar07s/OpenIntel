# [FEATURE]: security-remediation: warn before publishing the exploit details of unfixed High/Critical findings, and separate "accepted risk" from "fix pending"

## Feature and its Use Cases

**Current behaviour.** The two bundled skills work as a pair. Repos created from this template (ThruBox-Server, ThruBox-Client and IndexedDB-Import-Export, for example) get identical copies.

- `skills/security-review/SKILL.md:577`: each finding must include `Exploit Scenario: <a concrete attack path an attacker could follow>`. The report is saved under the git-ignored `unremediated-security-reviews/`, which "keeps unfixed findings out of the public repository in the meantime" (`:664-666`).
- `skills/security-remediation/SKILL.md:175-180` (Step 6.4): publication is offered once every finding has *either* a confirmed remediation commit *or* "a user-provided non-remediation explanation". The agent lists the not-remediated findings with their explanations and asks the user to approve.
- On approval, `:192-200` moves the original review into the tracked `security-reviews/` folder unchanged. The skill may not edit the review "beyond relocating it" (`:216-219`), so the published file still holds each unfixed finding's Description and Exploit Scenario.

Publishing accepted findings is intended (`security-review/SKILL.md:20-22`: "Once findings here are fixed (or explicitly accepted)…"), and the user has to approve it. The gaps are in how that approval is set up:

1. **Severity is never used.** Step 2 (`:52-53`) extracts each finding's severity, but no later step uses it. An unfixed Critical goes through the same approval prompt as an unfixed Low.
2. **The approval prompt does not say what becomes public.** Step 6.4 lists findings "with their explanations". It does not say that the original review, including the step-by-step Exploit Scenario of every unfixed finding, will be committed to a public folder.
3. **"Accepted risk" and "fix pending" are not defined.** Any free-text reason counts as a resolution in Step 6.4. Only the parenthetical in Step 6.5 (`:201-202`, "or remediation is still in progress") suggests that a deferred fix should block publication. So an answer like "no time this release, will fix later" can be read either way.
4. **There is no middle option.** Because the review cannot be edited, the user can either publish everything, including a working attack path for a live bug, or hold everything back, including the record of the findings that *were* fixed.

This sits awkwardly with the org security policy (`AOSSIE-Org/.github` `SECURITY.md:42`: "Coordinate disclosure and release a fix before public disclosure whenever possible"). It also conflicts with the natural reading of `security-review/SKILL.md:666`.

**Steps to reproduce (following the skill text):**

1. Run `security-review`. It saves `unremediated-security-reviews/sec_review_<ts>_<commit>.md` with a `Severity: Critical` finding that includes an `Exploit Scenario`.
2. Make no fix. Run `security-remediation`. At Step 4, give a reason such as "accepting the risk for now, no time this release".
3. Step 6.4 now counts every finding as resolved and asks for approval. The prompt lists the finding and its reason, but says nothing about severity or the Exploit Scenario.
4. Approve. Both files are moved into `security-reviews/`, and the next commit contains:
   ```
   security-reviews/sec_review_..._89ae62d.md:11:- Severity: Critical
   security-reviews/sec_review_..._89ae62d.md:15:- Exploit Scenario: Deploy a contract whose receive() re-enters withdraw() and drain the vault in one transaction.
   security-reviews/sec_review_..._89ae62d_remediations.md:13:This finding was not remediated because we accept the risk for now; no time this release.
   ```
   (Output from walking through Steps 6.4 and 6.5 by hand in a scratch clone with a sample Critical finding.)

**Expected:** before an unfixed Medium+ finding's exploit details go into a tracked public folder, the user is told about it plainly and offered a safer option.
**Actual:** the approval prompt leaves out severity and exploit details, and the only choices are "publish all" or "publish nothing".

## Additional Context

- Environment: `main` at `89ae62d`. The `skills` branch has the same content. Both files are byte-identical in ThruBox-Server, ThruBox-Client and IndexedDB-Import-Export. None of these repos has a `security-reviews/` folder yet, so nothing has been exposed. This is about future use.
- Open PR #167 (agent skills framework) does not touch `skills/security-review` or `skills/security-remediation`.

## Proposed fix

Text-only changes to the two `SKILL.md` files:

1. In Step 4 of `security-remediation`, ask the user to classify each unremediated finding as **accepted risk / won't fix**, **false positive**, or **fix pending**. Only the first two count as a resolution in Step 6.4. **Fix pending** always blocks publication, as Step 6.5 already suggests.
2. In the Step 6.4 approval prompt, show each unfixed finding's severity. State that the original review, including its Description and Exploit Scenario, will be committed to `security-reviews/`. For unfixed findings rated Medium or higher, recommend holding publication, or recording the details in a draft GitHub Security Advisory, until a fix ships.
3. Optionally, allow publishing the remediations file on its own (or a copy of the review with the Exploit Scenario of unfixed findings replaced by "withheld until fixed"), so fixed findings can be published without disclosing open ones. This needs a narrow exception to the "never edit the original report" rule at `:216-219`.
4. Reword `security-review/SKILL.md:664-666` and `:699-701` so they match: unfixed findings stay private until they are fixed or explicitly accepted, and accepted findings are published with their details.

## Code of Conduct

- [ ] I have joined the Discord server and will post updates there
- [ ] I have searched existing issues to avoid duplicates
