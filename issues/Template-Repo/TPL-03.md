# [BUG]: Issue AI Moderator never checks anything: GitHub Models is retired and `github/ai-moderator` is archived, so every issue is logged as "No spam"

## Bug Description

`.github/workflows/issue-moderator.yml` runs `github/ai-moderator@v1`, which sends issue text to the GitHub Models inference API (`https://models.github.ai/inference`). GitHub [fully retired GitHub Models on July 30, 2026](https://github.blog/changelog/2026-07-01-github-models-is-being-fully-retired-on-july-30-2026), and the `github/ai-moderator` repository is now archived. The workflow was added on 2026-10-21, after the retirement, so it has never worked in this repo.

The endpoint now answers every request with `200 OK`, `content-type: text/plain`, and the body `OK`, even without a token. The OpenAI client bundled in the action returns non-JSON bodies as plain text. The action's `response.choices[0]` (`dist/index.js:40781`) then throws `TypeError: Cannot read properties of undefined (reading '0')`. The action catches that error for each prompt and moves on, then logs `No spam or AI-generated content detected ✅`, and the step passes. No issue can ever get the `spam` or `ai-slop` label from this step. That also means the "Check Labels and Auto-Close" step never acts on a moderator verdict.

This is not specific to AOSSIE's org settings. Repos outside AOSSIE that pass the correct `token:` input show the same log output (for example `ampproject/amphtml` run 44589215121 and `end-4/dots-hyprland` run 44244067872). The same repos logged `APIError: 410 GitHub Models is temporarily unavailable as part of a scheduled retirement brownout` in September 2026.

Smaller problems in the same file:
- `github-token: \${{ secrets.GITHUB_TOKEN }}` (line 18) uses a wrong input name, and the backslash is a stray character. The action only accepts `token`. Every run warns `Unexpected input(s) 'github-token'`. This does not cause the failure: the action falls back to its default `token: ${{ github.token }}`, which the log shows as `token: ***`.
- The workflow triggers on `issues: [opened, edited]`, but the action only handles `opened`. On `edited` it logs `Nothing to do for event issues.` (run 42455503908). On those `edited` runs the auto-close step still comments on, closes and locks any issue that has a `spam` or `ai-slop` label, every time the issue is edited.
- Both actions target Node 20 (`github/ai-moderator@v1` declares `node16`; `actions/github-script@v7`), and every run warns that Node 20 is deprecated.

## Steps to Reproduce

1. Open the logs of any `opened` run of "Issue AI Moderator (Close & Lock)", for example run 42872094351 or 42455170625, step "Run AI Moderator".
2. Every built-in prompt fails with the `TypeError`, and the step still finishes with "No spam or AI-generated content detected ✅".
3. Check the endpoint directly. No token is needed, and adding any `Authorization` header gives the same response:
   ```sh
   curl -si -X POST https://models.github.ai/inference/chat/completions \
     -H 'content-type: application/json' -d '{}' | grep -iE '^HTTP|^content-type|^OK'
   # HTTP/2 200
   # content-type: text/plain
   # OK
   ```
   An unauthenticated request gets a plain-text `200 OK` instead of an error or a JSON completion, which shows the endpoint no longer serves inference.

## Logs and Screenshots

Run 42872094351 (issue opened 2026-11-10):
```
##[warning]Unexpected input(s) 'github-token', valid inputs are ['token', 'spam-label', 'ai-label', ...]
  github-token: \***
  token: ***
  endpoint: https://models.github.ai/inference
Evaluating content for spam and AI-generated content...
Error running prompt .../prompts/ai-detection.prompt.yml: TypeError: Cannot read properties of undefined (reading '0')
    at runPrompt (file:///home/runner/work/_actions/github/ai-moderator/v1/dist/index.js:40781:40)
Error running prompt .../prompts/link-spam-detection.prompt.yml: TypeError: Cannot read properties of undefined (reading '0')
Error running prompt .../prompts/spam-detection.prompt.yml: TypeError: Cannot read properties of undefined (reading '0')
No spam or AI-generated content detected ✅
```
Annotation on the same run: `Node.js 20 is deprecated. The following actions target Node.js 20 but are being forced to run on Node.js 24: actions/github-script@v7, github/ai-moderator@v1.`

## Environment Details

- `main` @ `89ae62d`, `.github/workflows/issue-moderator.yml` (added in `ace6306`)
- `github/ai-moderator@v1` = `81159c3` (repository archived)
- GitHub-hosted `ubuntu-latest` runner
- Repos created from the template carry the same file. `AOSSIE-Org/BabyNest` (run 44014474038) shows the same log output.

## Impact

Medium - Feature works but has issues. The workflow passes on every issue but performs no moderation, so it looks like spam protection while providing none. Every new repo created from the template inherits it.

## Proposed fix

- Remove the `github/ai-moderator@v1` step. It depends on a retired service and an archived action, so changing its inputs or org settings cannot make it work.
- If auto-close is still wanted, make it label-driven. Trigger on `issues: types: [labeled]` and act only when `github.event.label.name` is `spam` (or `ai-slop`). A maintainer's label then closes and locks the issue once, and later edits no longer re-run the step.
- If AI-based screening is still wanted, it needs a different, supported model provider and a repository secret. That is a decision for maintainers and should be a separate change.
- Whatever stays, set `token:` correctly (or drop the input), and move `actions/github-script` to v9 (open Dependabot PR #168 does only this bump).
