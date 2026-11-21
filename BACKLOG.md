# Team backlog: IndexedDB-Import-Export, ThruBox-Server, ThruBox-Client, Template-Repo

- Team: **Abhinav**, **Pavan**, **Ekansh**. Built 2026-11-20.
- Built from: the adversarial **VERDICTS** (every audit finding re-checked by an independent skeptic, plus the second-pass `-G` gap findings), the end-to-end **PROTOCOL** verification (ThruBox client/server), the four audit reports in this folder, and the GitHub snapshots in `_gh/`. The snapshots were taken 2026-11-19, so re-check live state before filing anything.
- Draft paths are relative to `/home/pavan/Desktop/gsoc/analysis/`.
- Points: S = 1, M = 3, L = 8 (no L items survived verification). Efforts marked `*` were not rated by a skeptic (protocol-only items) and are lead estimates.
- Verdict codes: **C** = CONFIRMED, **PC** = PARTIALLY_CONFIRMED (the correction is shown and must be applied before filing), **AH** = ALREADY_HANDLED, **UV** = UNVERIFIED.
- Fit is the skeptic's maintainer-fit call: likely / uncertain. `n/r` means not rated.

---

## 1. Summary

### Verdict counts

| Repo | Confirmed | Partially confirmed | Refuted | Already handled | Unverified | Total | Issues to file (after merges) |
|---|---:|---:|---:|---:|---:|---:|---:|
| IndexedDB-Import-Export | 3 | 19 | 0 | 2 | 0 | 24 | 22 new, plus 1 existing issue to claim (#131) |
| ThruBox-Client | 6 | 16 | 0 | 0 | 0 | 22 | 21 new (incl. protocol PROTO-3; TBC-G2 and TBC-G5 folded), plus 1 existing (#75) |
| ThruBox-Server | 7 | 17 | 0 | 0 | 1 | 25 | 25 new (incl. protocol PROTO-1 and PROTO-2; TBS-G4 folded; TBS-02 on hold) |
| Template-Repo | 3 | 22 | 0 | 0 | 0 | 25 | 24 new, plus 1 comment on an existing issue (TPL-06 → #112) |
| **All repos** | **19** | **74** | **0** | **2** | **1** | **96** | **92 new issues, 2 existing claims, 1 comment** |

Protocol mismatches (12 rows): 11 CONFIRMED and 1 PARTIALLY_CONFIRMED (S-P7). Three get their own drafts (PROTO-1 and PROTO-2 in the server, PROTO-3 in the client). Eight map onto existing findings. One (P-8, POST retry) is already tracked on GitHub (ThruBox-Client #82 / PR #83).

Severity after correction, over the 93 fileable verdict items: 1 high (TPL-01); 20 medium (IDB 7, TBC 2, TBS 5, TPL 6); 72 low. PROTO-1 is rated medium in its own draft but was not skeptic-rated. Every other high claim in the reports was lowered. Skeptics lowered severity on most PC items, so the corrected value is what goes in the issue.

### Headline items

| Repo | Headline items |
|---|---|
| IndexedDB-Import-Export | **IDB-01**: under overwrite, a backup with a valid `databaseVersion` but a broken structure deletes the database before it fails. PR #117 fixes only the missing-`stores` case. **IDB-02** (NEW): a blocked merge still applies its schema upgrade later and holds the connection queue. **IDB-14**: no PR CI at all, and `format:check` fails on main. **IDB-G5**: `merge` has no conflict policy and the README calls it "additive". Existing **#131** (Blob/File lost) is free to claim. |
| ThruBox-Client | **TBC-09**: `poll()` never pages, so a large mailbox on a slow link times out forever (0 deliveries in 32 s). **TBC-13**: no PR CI, and the release workflow tags before it tests (this happened with v1.1.0). **TBC-12**: `lint` and `format:check` fail on a clean main. **TBC-03 (+G2)**: a non-array 200 response with `after` set kills the Node process. |
| ThruBox-Server | **TBS-01**: the image is built with end-of-life Go 1.23.12 on end-of-life Alpine 3.20. The target must be at least go1.27.4 or go1.26.11, with the `go` directive bumped too. **TBS-04**: the rate limit can be bypassed by rotating addresses inside one IPv6 /64. **TBS-03**: addresses of unbounded length, stored 5 times and partly kept forever. **TBS-G2**: saved `after` cursors silently skip messages after a database reset or restore. **PROTO-1**: the CORS preflight blocks `X-API-Key` on keyless relays. **TBS-13**: no Go CI. |
| Template-Repo | **TPL-01** (high): CodeQL has never analysed code (0 of 182 runs), in the template and in every derived repo. **TPL-G3**: Socket skips Dependabot PRs and Go/Rust/Java/Ruby/.NET manifests. **TPL-G1**: LICENSE is a truncated GPL-3.0, copied into 20 derived repos and the npm packages. **TPL-02**: `.gitignore` is the LaTeX template. **TPL-03**: the AI moderator has never worked. It was added after GitHub Models was retired, and it logs "No spam" for every issue. **TPL-08**: template sync resets per-project files. **TPL-15**: version-release leaves stray tags behind. |

### The split at a glance (details in §7)

| Person | Home area | New issues | Existing | Points |
|---|---|---:|---:|---:|
| Abhinav | IndexedDB-Import-Export library code and docs; license/docs/scanner-config files in Template-Repo | 24 | #131 | 47 |
| Pavan | ThruBox-Client SDK source; Template-Repo workflow automation and its copies in derived repos | 36 (+1 comment) | #75 | 48 |
| Ekansh | ThruBox-Server (all Go, Docker, docs); CI, lint and packaging in all three code repos | 32 | none | 47 |

---

## 2. Cross-repo merges

When one underlying problem appears in several places, it is filed once, in the repo that owns the fix. The other places get a cross-reference, a follow-up PR, or nothing.

### 2a. Client/server protocol items

| Problem | Where found | Issue goes to | Other side | Owner |
|---|---|---|---|---|
| Timestamp precision: POST returns nanoseconds, GET and duplicate POST return whole seconds | P-1, S-P3, TBS-11 | **ThruBox-Server TBS-11** (truncate in the handler) | Client: nothing. The SDK passes the strings through. | Ekansh |
| `max_payload_size` limits the whole body, while the docs and `RelayPayloadTooLargeError` say "payload" | P-2, S-P4, TBS-03 sub-item, TBS-19 wording | **ThruBox-Server TBS-03** (payload-wording sub-item) | Client wording fix goes inside TBC-16, after the server decides. | Ekansh, then Pavan |
| Error contract: text/plain with a trailing `\n`, no machine-readable code | P-3, S-P8 | **ThruBox-Server PROTO-2** (design issue; agree on Discord first) | Client trims the text now in **TBC-08**. Parsing the envelope is a client follow-up after PROTO-2 is decided, with no issue yet. | Ekansh / Pavan |
| `Retry-After` hidden from browsers, and always the full 60 s | P-4, S-P5, TBS-12 | **ThruBox-Server TBS-12** (Expose-Headers, remaining time, fix the "sliding window" comment) | Client: **TBC-04** (honour `retryAfter`, capped) and existing **#75** (clamp negative values). Different code, so each gets its own ticket, cross-referenced. | Ekansh / Pavan |
| `X-API-Key` allowed by CORS only when the relay has a key | P-5, S-P6 | **ThruBox-Server PROTO-1** | Client: a docs mention in TBC-16 only. | Ekansh |
| Addresses are case-sensitive exact strings (EIP-55 vs lowercase) | P-6, S-P2, TBS-07 | **ThruBox-Server TBS-07** (decision and docs proposal) | Client: a docs note in TBC-16. An optional normaliser only if maintainers choose that option. | Ekansh |
| `send()` cannot tell a new message (201) from a deduplicated one (200) | P-7 | **ThruBox-Client PROTO-3** | Server: nothing. | Pavan |
| POST never retried, although the server deduplicates | P-8, S-P10 | **Nothing to file**: ThruBox-Client #82 / PR #83 | n/a | n/a |
| Unbounded mailbox reads vs a `poll()` that never pages | S-P1, TBC-09, TBS-02 | **ThruBox-Client TBC-09** | Server half **TBS-02 is on hold** (UNVERIFIED, no draft; see §3). | Pavan; Ekansh re-verifies TBS-02 |
| Default rate limit vs default polling | S-P7 (PC) | No new issue | Covered by TBC-04, TBS-12, and a polling-budget note in TBC-16. | n/a |
| No version or limits discovery | S-P9, TBS-17 | **ThruBox-Server TBS-17** | Client `HealthResponse.version` field as a later follow-up. | Ekansh |
| `from` is unauthenticated | S-P11, TBC-16, TBS-19 | Docs in both repos: **TBS-19** (server README), **TBC-16** (client README and `Message.from` JSDoc) | Each repo documents its own side. | Ekansh / Pavan |
| `"."` / `".."` / empty addresses | TBS-03, TBC-08 | Both repos (different code): the server rejects them on POST (TBS-03), the client rejects them before fetch (TBC-08) | Cross-reference the two. | Ekansh / Pavan |
| Cursors skip messages after a database reset | TBS-G2 | **ThruBox-Server TBS-G2** first (expose a DB id and `last_seq`) | SDK follow-up (store the id with the cursor) once the server shape is agreed. | Ekansh, then Pavan |
| Purge blocks sends ("database is locked") | TBS-G1 | **ThruBox-Server TBS-G1** (batched purge) | The client-side retry already lives in #82 / PR #83. | Ekansh |

### 2b. The same fix pattern in several repos (one owner each, rule d)

| Pattern | Items | Decision | Owner |
|---|---|---|---|
| CommonJS consumers get the ESM `.d.ts` | IDB-15, TBC-05 (+ TBC-G1 edits the same `exports` block) | One issue per repo, since each `package.json` is separate. TBC-G1 waits for TBC-05. | Ekansh |
| No PR CI (build, test, lint, format) | IDB-14, TBC-13, TBS-13 | One issue per repo. Template-Repo has no language CI that could sync down, and template-sync skips workflows anyway. TBC-G5 (Node 18 vs toolchain) is folded into TBC-13. | Ekansh |
| Lint/format broken on main, must be fixed before CI enforces it | IDB-18 Prettier part, TBC-12, TBS errcheck hits | IDB: the `src/utils/ssr.ts` fix goes into the IDB-14 PR. TBC-12 is filed and merged before TBC-13. TBS: the 8 non-test errcheck hits are fixed or excluded inside the TBS-13 PR. | Ekansh |
| Stale bot closes Dependabot PRs, and sync-pr-labels strips `dependencies` | TBS-15, TPL-07 (strips `dependencies`), the same `stale.yml` in ThruBox-Client and Template-Repo | File **TBS-15** in ThruBox-Server (strongest evidence: the Go and Alpine bumps were suppressed). The Template `sync-pr-labels` part rides in **TPL-07**. The ThruBox-Client `stale.yml` gets a follow-up PR referencing TBS-15, not a new issue. | Pavan |
| GitHub Actions SHA pinning | TPL-14, TBC-17, TBS-13, TBS-15 claims | **No new issue anywhere.** Add the two concrete inconsistencies as a comment on Template-Repo **#75** (assigned to kpj2006). | Pavan |
| Tags pushed with `GITHUB_TOKEN` don't trigger other workflows | TPL-04, TPL-09, TPL-15 | Filed **once, in TPL-04**. TPL-09 and TPL-15 reference it. | Pavan |

### 2c. Template bugs that also live in derived repos

`.github/workflows/**` and `LICENSE` are excluded from template sync, so a template fix only reaches new repos. Policy: **one issue, in Template-Repo**, listing the affected derived repos. Follow-up PRs in derived repos only after the template issue is accepted and the derived repo's maintainer agrees. Those PRs reference the template issue; no new issue is opened.

| Template item | Also present in | Follow-up PRs (no new issue) | Owner |
|---|---|---|---|
| TPL-01 CodeQL matrix skip | ThruBox-Server, ThruBox-Client, IndexedDB (Analyze skipped). TBS-13 item 5 points here. | `codeql.yml` in all three, after TPL-01 | Pavan |
| TPL-05 gitleaks failing on Dependabot | ThruBox-Server (24 of 24 Dependabot runs), ThruBox-Client (3 of 3) | `gitleaks-scanning.yml` in TBS and TBC | Pavan |
| TPL-11 `chore(deps)(deps)` prefix | All three (for example IDB Dependabot PRs #118-#122) | `dependabot.yml` in all three | Pavan |
| TPL-20 Scorecard push trigger on a non-existent `dev` | ThruBox-Server, ThruBox-Client (IDB already fixed it) | `scorecard.yml` in TBS and TBC | Pavan |
| TPL-G1 truncated GPL-3.0 LICENSE | IDB, ThruBox-Server, ThruBox-Client (the same blob, also shipped in both npm packages) | `LICENSE` in all three | Abhinav |
| TPL-G2 Dependency Review allow-list | All three (identical file hash) | `dependency-review-action.yml` in all three | Abhinav |
| TPL-19 CONTRIBUTING TOC and U+FFFD | All three (IDB-18 item 4 and TBC-17's mojibake point move here) | `CONTRIBUTING.md` in all three, in the final wave | Abhinav |
| TPL-02 LaTeX `.gitignore` | Already replaced in all three derived repos | none | n/a |
| TPL-13 Release Drafter | Already fixed in IDB (PR #96) | none | n/a |
| TPL-15 version-release ordering | Already rewritten in all three derived repos; their pattern is backported | none | n/a |

### 2d. Overlaps with existing GitHub work and within-repo merges

| Item | Overlap | Decision |
|---|---|---|
| IDB-01 | #114 / PR #117 edit the same function | File IDB-01 (envelope and schema validation). Comment on PR #117 first so the two don't conflict. |
| IDB-03 (AH) | Duplicate of #114 | Do not file. Comment on #114/#117 with the ConstraintError/DataError repro, which PR #117 does not fix. |
| IDB-10 | Same root cause as #114 (IDB-03) | File as a feature that references #114. Agree on Discord: a new strategy, or changed `overwrite` semantics. |
| IDB-02, IDB-G2 | #126 / PR #130 (overwrite path only) | File IDB-02 now (merge path, NEW). IDB-G2 later, referencing #126 and IDB-02. |
| IDB-07, IDB-G1, #131 | The same `serialize()` fall-through | One owner (Abhinav). IDB-07 is narrowed: the CryptoKey sentence and the throw-on-unknown fix item are already removed from the draft. Claim #131. File IDB-G1 later as its own issue, cross-referenced, after draft PR #128 merges and #131's approach is agreed. |
| IDB-17, IDB-18 | Both touch the `idb` wording in README/AGENTS | IDB-18 is narrowed to `package.json`/lock (Ekansh). IDB-17 leaves the `idb` wording alone, and Abhinav updates it after IDB-18's direction is decided (re-raises #99). |
| IDB-16 | PR #130 and PR #128 add some of these tests | Draft is already narrowed. The `__proto__` test and the `toStrictEqual` switch go with IDB-09 and IDB-08. |
| TBC-03, TBC-G2 | Same crash (floating `doPoll()` promise); G2 adds the realistic trigger | **Merged** into one issue (the TBC-03 draft now includes G2's `receive()` shape check). |
| TBC-13, TBC-G5 | The CI matrix half of G5 is already in the TBC-13 draft | **Merged**: G5's CONTRIBUTING "Node 18" prerequisite fix is item 4 of the TBC-13 draft. |
| TBC-01, TBC-10 | Same signal plumbing through `request()` | Same owner. TBC-01 first, then TBC-10. |
| TBC-02, TBC-04 | Delay overflow | TBC-04 caps its own delay below 2^31-1 ms. TBC-02 adds the validation. |
| TBC-06, TBC-07 | PR #81 edits the same catch block; PR #83 the same constructor line | Mention it in each issue and rebase. |
| TBS-05, TBS-G4 | Same `Validate` function | **Merged** into one config-validation issue (ttl bound and IPv6 host are items 6-7 of the TBS-05 draft). |
| TBS-13, TBS-20 | errcheck hits | TBS-13 fixes or excludes the 8 non-test hits. TBS-20 is narrowed to its 4 real items. |
| TBS-19 | Payload wording, the "sliding window" comment, `storage.driver`, Go prerequisite | Moved to TBS-03, TBS-12 and TBS-05; the Go prerequisite is already in TBS-01. The SECURITY.md / private reporting point is a repo-settings request to maintainers, not issue content. |
| TBS-09 | Maintainers' own `audit/sec_review_2026-09-23` Finding 2 | File as tracking that documented follow-up. |
| TPL-06 | #112 (assigned to kpj2006) and #176 (reporter asked to be assigned) | **No new issue.** Comment on #112 offering the link and indent fix, and add the bot-PR evidence to #176. |
| TPL-12 | #85 (assigned to Okd2006; PR #89 stale-closed) | Comment on #85 first. File only if the assignee agrees or the scope is clearly different. |
| TPL-17 | #110 (assigned to kpj2006) | File, but coordinate on #110 (N/A boxes, category reset and date churn are not in #110). |
| TPL-16 | #102 (unassigned; PR #103 stale-closed) | Pair with #102 in a late wave. |
| TPL-11 | #72, #73 / PR #172 (same file) | File the prefix and cooldown parts only. Unused ecosystems are left to #72. Coordinate with PR #172. |
| TPL-14 | #75 (pinning), TPL-05 (gitleaks permissions), #11 | Scope is the 5 injection points plus the osv permissions. Pinning goes to #75, gitleaks permissions to TPL-05. Do not claim that it "answers #11". |

---

## 3. Do-not-file list

### 3a. Refuted, already handled, unverified, or already tracked

There were **no refuted findings**.

| Item | Verdict | Reason (one line) | Do this instead |
|---|---|---|---|
| IDB-03 | AH | Duplicate of open #114, which already covers the overwrite ConstraintError loss. PR #117's author offered the follow-up himself. | Comment on #114/#117 that PR #117 leaves the ConstraintError and DataError cases open, or send a PR that references #114. |
| IDB-04 | AH | The hang and leak are fully fixed by draft PR #128 (verified), and #131 cites it. Encoding Invalid Date would reverse the PR #27 design. | Optional review suggestion on #128: put the store name and key in the error. |
| TBS-02 | UV | No independent verdict and no draft. The protocol pass did confirm the unbounded GET end to end (8.2 MB response, truncation, 0 deliveries), but the finding's details and fix shape are unverified and need a maintainer decision. | **Hold.** Ekansh re-verifies before filing. The client half (TBC-09) goes ahead. |
| P-8 / S-P10 | protocol C | POST retry is already tracked. | ThruBox-Client #82 / PR #83. Review PR #83: its `{ cause }` change fails `tsc` on main (see TBC-12). |
| S-P7 | protocol PC | The result is periodic loss, not bucket exhaustion. The proxy case is already documented. | Covered by TBC-04, TBS-12 and the TBC-16 docs note. |

### 3b. Folded into another issue (no separate issue)

| Folded item | Into | Why |
|---|---|---|
| TBC-G2 | TBC-03 | Same floating-promise crash. TBC-03's prototype already has the `Array.isArray` guard. |
| TBC-G5 | TBC-13 | The TBC-13 draft already uses Node 22/24 plus a Node 18/20 dist smoke job. Only the CONTRIBUTING prerequisite line is new. |
| TBS-G4 | TBS-05 | Same `Validate` function. The maintainer was expected to prefer one config-validation issue. |
| IDB-18 Prettier part | IDB-14 | The IDB-14 draft already fixes `src/utils/ssr.ts` in the same PR. |
| IDB-17 `idb` wording | IDB-18 follow-up (docs PR by Abhinav, W5) | The README/`AGENTS.md` wording depends on the drop-or-adopt decision in IDB-18. |
| TBS-19 Go-prerequisite item | TBS-01 | The TBS-01 draft already updates the Go version in README and CONTRIBUTING. |
| TBC-14 coverage thresholds | TBC-15 | Keeps `vitest.config.ts` with one owner (Ekansh) in W5. |
| TPL-11 unused-ecosystem failures | Comment on #72 | #72 owns enabling only the ecosystems a project uses. |

### 3c. Sub-claims dropped by the skeptics (keep them out of every issue and PR)

| Finding | Dropped claim | Reason |
|---|---|---|
| IDB-01 | JSON arrays get the DB deleted; unknown `strategy`; `backupVersion` check | Rejected by the existing check before any delete. The strategy is a ConstraintError with data intact. The version check is hardening, not data loss. |
| IDB-02 | "A connection nobody can close"; the same bug at :185/:272 | GC releases it in Chrome. :185 is reworked by PR #130. |
| IDB-05 | "Document storeNames to avoid the bump" | Already in the JSDoc (PR #73). |
| IDB-06 | "Bump the version to add indexes" | Contradicts IDB-05 and breaks the app's next `open()`. |
| IDB-07 | Float16Array "corrupted on export"; CryptoKey and the catch-all | Export is complete, only the rebuild fails. CryptoKey goes to IDB-G1, after PR #128. |
| IDB-08 | Change `recordCounts` to `{}` | A null prototype is the better dictionary choice. |
| IDB-12 | Configurable `batchSize`; paged `getAll()` speedup | It breaks atomicity, and the speedup is unmeasured. |
| IDB-17 | SECURITY.md sub-point | The org SECURITY.md exists. |
| IDB-18 | `migrations.ts`, `revokeObjectURL`, dev advisories, mojibake | Intentional placeholder, speculation, Dependabot's job, and upstream (TPL-19) respectively. |
| IDB-G3 | New NDJSON backupVersion 2; default to compact JSON | One-record-per-line output is still valid v1. Compact JSON hits the same ceiling. |
| IDB-G4 | encode/decode hooks on the file helpers; transform as a codec for unsupported types | Encryption already works with the existing helpers. A sync callback can't read Blob contents. |
| IDB-G5 | Built-in "newer wins"; skip via `add()` plus `preventDefault` | The library doesn't know the timestamp field. A unique-index ConstraintError would drop records silently. |
| IDB-G6 | Throw by default; `allowEmpty` | Breaks the tested 1.x behaviour. |
| TBC-02 | Backoff overflow at `retries >= 23` | Only reachable after 48.5 days of sleeps. |
| TBC-04 | Auto-stop polling on 401; "never backs off" | That would be a behaviour change. 5xx and network errors already back off. |
| TBC-05 | `"./package.json"` subpath | Unrelated. |
| TBC-06 | Missing global `DOMException`; the credentials case | Not reachable on supported runtimes. The credentials case is TBC-07. |
| TBC-08 | `RelayValidationError` with statusCode 0; a hard-coded 1000 cap | Clashes with RelayNetworkError's status 0 and ties the SDK to one server. |
| TBC-17 | `sideEffects`, `./package.json` export, `.secrets.baseline`, `audit fix`, 5xx body | Noise, or it belongs elsewhere. |
| TBC-G3 | Security framing; `redirect: 'manual'` by default | Already reviewed by the maintainers. It would break working 307/308 setups. |
| TBS-01 | Target go1.27.3 (#71); "MaxBytesReader crash reachable" | 1.27.3 still has GO-2026-6859. The crash needs concurrent read and write, which `HandleCreate` doesn't do. |
| TBS-03 | Prune `mailboxes` rows after the TTL | Intentional design (#69). |
| TBS-05 | Invalid env values should be a startup error | A test pins "warning, not failure". Fix the wording and log level only. |
| TBS-06 | Readiness via `db.PingContext` | A no-op in go-sqlite3. It doesn't detect a full disk. |
| TBS-07 | Merge-mailboxes migration as the default | Renumbers `seq` and breaks cursors. |
| TBS-08 | 403 for any request with an unlisted Origin | Breaks same-origin proxied apps. |
| TBS-13, TBS-15 | "Every other workflow is SHA-pinned" | False. Pinning belongs to #75. |
| TBS-15 | Add a `go` label to the exempt list; exempt `docker`/`github-actions` | The label doesn't exist, and that would also exempt human PRs. |
| TBS-16 | Per-mailbox cap on by default | Lets an attacker lock a victim's mailbox (`from` is unauthenticated). |
| TBS-17 | ldflags from VERSION in the Dockerfile; keyless `/api/info` | `.dockerignore` excludes VERSION. The design is the maintainers' choice. |
| TBS-20 | noctx, DL3018, purge-log order, the `address == ""` branch, goreleaser leftover | Lint noise or already acknowledged. |
| TPL-02 | "Commits added `unremediated-security-reviews/`" | Inaccurate history. |
| TPL-03 | Rename the input to `token`; enable Models for the org | GitHub Models is retired, so neither helps. |
| TPL-04 | `merge_group` | Upstream design. |
| TPL-05 | Re-add `pull_request` | Fork PRs get no secrets (the trigger was removed in 62780d1). |
| TPL-06 | The core "flags compliant PRs" bug | That is #112. |
| TPL-09 | `version: latest`, zizmor nits | Overlap with #75. |
| TPL-12 | "54 labels never created"; the 7 stale-closed PRs | Most labels are auto-created. The stale policy is a separate question. |
| TPL-13 | Optional autolabeler job | There are no autolabeler rules. |
| TPL-14 | 41 unpinned refs; artipacked ×9; gitleaks permissions | #75, mostly noise, and TPL-05 respectively. |
| TPL-16 | Missing secrets | `GITLEAKS_LICENSE` is org-wide; the others are commented out or fall back. |
| TPL-17 | Tag-push / any-branch trigger concern | Unverified. |
| TPL-19 | README U+FFFD and logo alt text; placeholder logo; INSTALL_GUIDE; CoC link | PR #167, an intentional placeholder (PR #63 closed), "(if you have one)", and the org fallback respectively. |
| TPL-20 | yamllint "errors", the ESLint boolean, `.gitattributes`, CODEOWNERS | Valid YAML, a legacy value VS Code still accepts, redundant, or opinion. |
| TPL-G1 | CI check that the license is GPL-3.0 | Would break derived repos that choose another license. |
| TPL-G2 | Keep GPL-2.0-or-later; add BSD-3-Clause-Clear / Python-2.0.1 | `-or-later` still matches `-only`. No evidence the additions are needed. |
| TPL-G4 | The private-vulnerability-reporting point | Irrelevant to maintainers' own drafts. |

---

## 4. Security items to raise privately first

**No verdict is marked `private=true`, so nothing has to go to maintainers privately before a public issue.** The skeptics judged every security-typed item safe to file publicly:

| Item | Why public filing is fine (per verdict) |
|---|---|
| TBS-01 (EOL Go and Alpine) | Public stdlib CVEs. The reachable ones are low impact, and the headline MaxBytesReader crash is not triggerable as the code stands. |
| TBS-04 (IPv6 /64 limiter bypass) | The maintainers fixed the sibling XFF bypass (#68 / PR #67) in public. It needs control of a /64. |
| TBS-08 (cross-origin `text/plain` POST) | Low. The relay is open to non-browser clients by design. |
| TBS-09 (API-key guesses not rate limited) | Already published in the maintainers' own `audit/sec_review_2026-09-23` Finding 2. |
| TPL-01 (CodeQL never runs) | "Not an exploitable vulnerability, so it can be filed publicly." |
| TPL-14 (template injection, `security-events: write`) | Every injection point needs write access. This is hardening, with no privilege escalation. |
| TPL-G3 (Socket coverage gaps) | Defense-in-depth configuration, not directly exploitable. |

Advice: private vulnerability reporting is **disabled** on ThruBox-Server, ThruBox-Client and IndexedDB-Import-Export (TBS-19 and TPL-16 evidence). If anyone later finds something that does need privacy, **DM a maintainer on Discord first** and do not open a public issue. For the items above, the normal flow applies: open the issue, then post it in the project channel.

---

## 5. Ranked backlog per repo

Only confirmed and partially confirmed items are listed. Ranking: corrected severity, then maintainer fit, then bug before feature, then smaller effort. "Owner·W" is the owner and wave from §7.

### 5.1 IndexedDB-Import-Export

| # | ID | Issue title | Type | Sev | Eff | Verdict | Overlap / refs | Fit | Owner·W | Draft | Correction applied (PC) |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | IDB-01 | [BUG]: Overwrite import deletes the database before checking the backup's structure, so a malformed backup wipes existing data | bug | medium | M | PC | Partially covered: #114 / PR #117 (fixes the missing-`stores` case only), PR #130, PR #100 precedent, maintainers' 2026-09-19 review | likely | Abhinav·W2 | `issues/IndexedDB-Import-Export/IDB-01.md` | Only backups with a valid `databaseVersion` and a broken structure reach the delete; arrays and foreign JSON are rejected first. Importer rejects (not silent), but after the data is gone. high→medium. Unknown strategy and backupVersion are not data loss. Missed case: merge without `schema` never settles. |
| 2 | IDB-02 | [BUG]: A blocked merge import still applies its schema upgrade after the other connection closes, and leaves a connection open | bug | medium | S | PC | NEW. #126 / PR #130 cover the overwrite path only | likely | Abhinav·W1 | `…/IDB-02.md` | In Chrome the stray connection lasts until GC or reload (permanent only in fake-indexeddb). high→medium; no data loss. Scope is the merge upgrade at :235-252. Added effect: the abandoned request holds the connection queue. |
| 3 | IDB-14 | [FEATURE]: Run build, tests, lint and format checks on pull requests, and before a release is tagged | ci | medium | S | PC | NEW. BestPracticesChecklist gap; PR #98 | likely | Ekansh·W2 | `…/IDB-14.md` | PR #98's resume path came from an `npm pack` ENOENT, not tests. version-release.yml still uses tag refs. The ssr.ts issue dates from PR #28. Use Node 22/24 (20 is EOL). Includes the ssr.ts Prettier fix from IDB-18. |
| 4 | IDB-G5 | [FEATURE]: Add a conflict policy to `merge` imports (overwrite / skip existing / custom resolver) | feature | medium | M | PC | NEW. PR #12 / test :294 lock in upsert | likely | Abhinav·W4 | `…/IDB-G5.md` | Upsert is intended and tested; the gap is having no policy, plus the "additive" README wording. The dexie comparison was wrong. "Skip" must check the key with `get()`, not catch ConstraintError. The resolver must be synchronous. |
| 5 | IDB-G1 | [BUG]: CryptoKey, FileSystemHandle and RTCCertificate values are exported as `{}` with no warning, and restoring in place replaces the stored key | bug | medium | M | PC | Partially covered by #131 (same fall-through); needs PR #128; conflicts with PR #127; the IDB-07 draft | uncertain | Abhinav·W5 | `…/IDB-G1.md` | high→medium; key loss only on an in-place restore. Reproducible in fake-indexeddb (Node CryptoKey). Merge also replaces the key. A throw-based fix needs #128 and must not break DOMRect/DOMMatrix. De-duplicate against IDB-07. |
| 6 | IDB-G3 | [BUG]: Backups with more than ~380 MiB of binary data can't be saved or read in Chromium (downloadJSON throws RangeError: Invalid string length) | bug | medium | M | PC | NEW. #26 (design) | uncertain | Abhinav·W5 | `…/IDB-G3.md` | The ceiling is V8/Chromium only (Firefox has about 2×). Compact JSON doesn't help. No new format is needed. The import side fails with a misleading NotReadableError. The export failure is the main problem. |
| 7 | IDB-10 | [FEATURE]: Add an atomic restore that clears and refills stores in one transaction instead of deleting the database | feature | medium | M | PC | NEW. #114 root cause (IDB-03), #126 | uncertain | Abhinav·W5 | `…/IDB-10.md` | Guarantees hold only if the target already has every store with a matching keyPath; otherwise a versionchange is still needed. Coordinate with #114. |
| 8 | IDB-09 | [BUG]: exportDB silently drops an object store named `__proto__` | bug | low | S | C | NEW. PR #87 precedent | likely | Abhinav·W2 | `…/IDB-09.md` | none (the draft uses a self-contained `Object.fromEntries` fix) |
| 9 | IDB-15 | [BUG]: CommonJS TypeScript consumers get the ESM type declarations, so `import` fails to type-check under `module: node16` | bug | low | S | C | NEW. Same pattern as TBC-05 | likely | Ekansh·W4 | `…/IDB-15.md` | (C) Low: only node16, and nodenext before TS 5.8. |
| 10 | IDB-07 | [BUG]: Error objects become `{}` in a backup, and Float16Array and boxed primitives are not restored | bug | low | M | PC | Partially covered: #131; #81 / PR #82 precedent; PRs #125, #127, #133 touch `serialize()` | likely | Abhinav·W3 | `…/IDB-07.md` | Float16Array exports fine, only the rebuild fails (with a warning). The BigInt wrapper fails loudly. Boxed primitives keep their value. Only Error is a silent loss. medium→low. Draft edited: the CryptoKey sentence and the throw-on-unknown fix item are removed (they live in IDB-G1). |
| 11 | IDB-13 | [FEATURE]: Speed up base64 encoding of binary values (serialize is about 12× slower than it needs to be on large buffers) | improvement | low | S | C | NEW. PR #127 touches the call sites | likely | Abhinav·W3 | `…/IDB-13.md` | (C) Measured in Node. Use `toBase64` only "when available", with feature detection. |
| 12 | IDB-17 | [BUG]: README overstates SSR safety and has stale/incorrect details; exportDB/importDB throw a raw ReferenceError outside the browser | docs | low | S | PC | Partially covered: #99 and #101 closed | likely | Abhinav·W3 | `…/IDB-17.md` | #99 was auto-closed by PR #100. Drop the SECURITY.md point. Add ESLint 9→10 and the bigint SyntaxError. Link open issues instead of listing them as limitations. Draft edited: the `idb` wording item is removed (it follows IDB-18's decision as a docs follow-up), so this PR no longer touches `AGENTS.md`. |
| 13 | IDB-16 | [FEATURE]: Add tests for the untested exportDB/importDB error paths, and type-check the test files | test | low | S | PC | Partially covered: PR #130, PR #128 | likely | Abhinav·W5 | `…/IDB-16.md` | importer.ts :451/:454 are also uncovered. The blocked and serialization-failure paths are left to PRs #130/#128. The `__proto__` test and `toStrictEqual` go with IDB-09/08. |
| 14 | IDB-05 | [BUG]: `merge` import that creates a missing store bumps the DB version, so the app's own `indexedDB.open(name, version)` fails with `VersionError` | bug | low | S | PC | NEW. PR #12, PR #73 | uncertain | Abhinav·W4 | `…/IDB-05.md` | The JSDoc does mention the bump; it is intended and tested. Frame it as docs plus an opt-in `createMissingStores: false`, not a code defect. Missed effect: the app's next migration can be skipped. Draft Impact corrected from Medium to Low. |
| 15 | IDB-06 | [BUG]: `merge` ignores schema differences on stores that already exist (missing indexes skipped, keyPath mismatch fails with an unwrapped `DataError`) | bug | low | M | PC | NEW. #114 root cause (half-applied merge) | uncertain | Abhinav·W4 | `…/IDB-06.md` | medium→low. The skip is intentional. Drop "bump the version". The real defects: synchronous `put()` throws reach the caller unwrapped, and the merge is half-applied. |
| 16 | IDB-08 | [BUG]: `deserialize()` and `serialize()` return null-prototype objects, unlike the records IndexedDB itself returns | bug | low | S | PC | NEW. PR #87 (deliberate), PR #127 conflicts | uncertain | Abhinav·W5 | `…/IDB-08.md` | Deliberate security choice. Frame it as API consistency for serialize/deserialize output only. Keep `recordCounts` null-prototype (draft now says so instead of leaving it open). `serialize()` is affected too. |
| 17 | IDB-18 | [BUG]: Unused `idb` runtime dependency is installed for every consumer | cleanup | low | S | PC | Partially covered: #99 and #101 closed | uncertain | Ekansh·W4 | `…/IDB-18.md` | **Narrowed to the unused `idb` dependency** (`package.json`/lock); draft retitled and rewritten. The format fix ships in IDB-14. Mojibake (TPL-19) and dev advisories are out of the draft. README/`AGENTS.md` wording is a later docs follow-up by Abhinav. Ask on Discord first whether to drop or adopt `idb`. |
| 18 | IDB-G6 | [FEATURE]: Let exportDB reject storeNames that don't exist in the database (a typo currently produces an incomplete or empty backup) | improvement | low | S | PC | NEW. #67 / PR #73 mirror rule | uncertain | Abhinav·W5 | `…/IDB-G6.md` | Deliberate, maintainer-tested behaviour. Propose an opt-in `strict` flag (default false) plus docs. |
| 19 | IDB-11 | [FEATURE]: Return a result from importDB (imported vs. aborted, record counts) | feature | low | S | PC | NEW. #68 / PR #87 | uncertain | Abhinav·W5 | `…/IDB-11.md` | An ergonomics improvement, not a correctness gap. Keep non-boolean hook returns allowed. |
| 20 | IDB-G2 | [FEATURE]: Let exportDB/importDB time out or be aborted instead of waiting forever behind a queued version change | improvement | low | M | PC | NEW. #126 / PR #130 | uncertain | Abhinav·W5 | `…/IDB-G2.md` | The app's own `open()` hangs the same way (raw IndexedDB behaviour). medium→low. Most common trigger: the library's own blocked import. A signal cannot withdraw a queued delete. Test with fake-indexeddb, not Playwright. |
| 21 | IDB-12 | [FEATURE]: Add an onProgress callback to exportDB and importDB for large databases | feature | low | M | PC | NEW | uncertain | Abhinav·W5 | `…/IDB-12.md` | The fake-indexeddb timings don't show a real need. File only an `onProgress` inside the existing transaction. No `batchSize`, and no paged-`getAll()` follow-up (removed from the draft). |
| 22 | IDB-G4 | [FEATURE]: Add per-record `filter` and `transform` options to `exportDB` and `importDB` | feature | low | M | PC | NEW. #67/#73, #68/#87 | uncertain | Abhinav·W5 | `…/IDB-G4.md` | Encryption already works (docs recipe only). This is an ergonomic API, not something impossible today. Narrowed to filter/transform, `includeData` and the recipe. medium→low. |

### 5.2 ThruBox-Client

| # | ID | Issue title | Type | Sev | Eff | Verdict | Overlap / refs | Fit | Owner·W | Draft | Correction applied (PC) |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | TBC-09 | [FEATURE]: Add a page size (`limit`) to `poll()` and fetch the next page right away when a page is full | feature | medium | M | C | NEW. Server half TBS-02 (on hold); #64 / PR #63; same loop as TBC-01/04 | likely | Pavan·W2 | `issues/ThruBox-Client/TBC-09.md` | (C) A default `limit` of 100 doesn't bound bytes (100 × 500 KB). `receiveAll` stays optional. |
| 2 | TBC-13 (+G5) | [FEATURE]: Add a CI workflow that builds, tests and lints every pull request, and test before tagging a release | ci | medium | M | C | NEW. TBC-12 prerequisite; PR #66; same pattern as IDB-14/TBS-13 | likely | Ekansh·W2 | `…/TBC-13.md` (G5 draft kept as evidence only) | (C) Node 22/24 for build and test, plus a Node 18/20 dist smoke job. PR #66 already added re-runs. G5's CONTRIBUTING "Node.js 18" prerequisite fix is now item 4 of the draft. README/`AGENTS.md`/`engines` stay untouched in this PR; a floor bump goes only to ≥22, as a separate breaking change. |
| 3 | TBC-12 | [BUG]: `npm run lint` and `npm run format:check` fail on a clean checkout of `main` | cleanup | low | S | C | NEW. PR #81/#83 call these failures "pre-existing"; #83's `cause` change fails tsc | likely | Ekansh·W1 | `…/TBC-12.md` | (C) Prefer `singleQuote: false` (+30/-61 lines vs +197/-228) to limit conflicts with #81/#83. Use lib `ES2022.Error`. |
| 4 | TBC-07 | [BUG]: `RelayClient` accepts a base URL with a query string, fragment or credentials, then every request fails | bug | low | S | C | NEW. PR #83 edits the same constructor line | likely | Pavan·W4 | `…/TBC-07.md` | (C) The report's alternative fix was wrong. Also reject a bare `?`/`#` (store origin + pathname). The error message echoes credentials. |
| 5 | TBC-03 (+G2) | [BUG]: `poll()` loop promise is never caught: a non-array response (or an unstringifiable throw) exits the Node process and stops polling | bug | low | S | PC | NEW. PR #63; #80 / PR #81 adjacent | likely | Pavan·W3 | `…/TBC-03.md` (+ `…/TBC-G2.md`) | medium→low. The realistic trigger is a non-array 200 with `after` set. The fix must reschedule (`finally` or catch-and-reschedule) plus an `Array.isArray` guard in `receive()`. **G2 folded in (draft updated):** the reference server always returns an array, a `null` body loops silently, the shape check lives in `receive()`, and the typed error carries the real status. |
| 6 | TBC-04 | [BUG]: `poll()` ignores `RelayRateLimitError.retryAfter` and keeps polling every `intervalMs` while rate limited | bug | low | S | PC | NEW. #75, TBS-12, TBC-02 | likely | Pavan·W3 | `…/TBC-04.md` | 5xx and network errors already back off; only non-retried 4xx lack it. The server window isn't extended by ignored requests. Keep the "keep polling" contract. Clamp **and cap** the delay (Infinity, overflow, negatives). medium→low. |
| 7 | TBC-01 | [BUG]: `poll()` stop function does not abort the in-flight request or its retries | bug | low | S | PC | NEW. Plumbing shared with TBC-10 | likely | Pavan·W4 | `…/TBC-01.md` | medium→low. Extra requests only happen when the in-flight receive is failing. The open attempt isn't aborted until it times out. Use a manual listener, since `AbortSignal.any` needs Node 18.17+. |
| 8 | TBC-05 | [BUG]: CommonJS TypeScript projects get ESM type declarations from the `require` export (TS1479) | bug | low | S | PC | NEW. Same pattern as IDB-15; TBC-G1 same block | likely | Ekansh·W4 | `…/TBC-05.md` | Only `node16`/`node18`, or `nodenext` on TS < 5.8. medium→low. Drop the `./package.json` subpath. |
| 9 | TBC-02 | [BUG]: `timeout` and `intervalMs` above 2147483647 ms pass validation, then fire after ~1 ms (requests time out at once, `poll()` floods the relay) | bug | low | S | PC | NEW | likely | Pavan·W4 | `…/TBC-02.md` | medium→low. Browsers wrap the delay modulo 2^32 rather than clamping it. Drop the backoff-overflow sub-claim. Reject vs clamp is the maintainers' choice. |
| 10 | TBC-06 | [BUG]: `RelayNetworkError` discards the underlying fetch error, so Node network failures only say "fetch failed" | improvement | low | S | PC | NEW. #80 / PR #81 same catch block | likely | Pavan·W4 | `…/TBC-06.md` | A diagnostics gap on Node only. The DOMException point is optional hardening. The credentials case belongs to TBC-07. No lib bump is needed. medium→low. |
| 11 | TBC-08 | [BUG]: Empty, "." or ".." address fails with a misleading `RelayNotFoundError("404 page not found\n")`; server error messages keep a trailing newline | bug | low | S | PC | NEW. TBS-03 (server side), PROTO-2 | likely | Pavan·W4 | `…/TBC-08.md` | Narrowed: reject `''`/`.`/`..` before fetch and trim the server text. No status-0 error class and no 1000 cap. The maintainers' review already called the dot case harmless for security. |
| 12 | TBC-G4 | [BUG]: Unread 401/413/429/5xx response bodies keep sockets open on Node until GC (bodies of 64 KiB or more) | bug | low | S | PC | NEW. PR #83 nearby | likely | Pavan·W4 | `…/TBC-G4.md` | Only bodies of about 64 KiB or more. Not reachable against ThruBox-Server alone. GC releases them. A hygiene fix. |
| 13 | TBC-10 | [FEATURE]: Accept an `AbortSignal` in `send()`, `receive()` and `health()` | feature | low | M | C | NEW. TBC-01 | likely | Pavan·W4 | `…/TBC-10.md` | (C) medium→low. No `AbortSignal.any()`. |
| 14 | TBC-16 | [FEATURE]: README: document browser/CORS setup, security notes, all error classes and retry timing | docs | low | S | PC | Partially covered: #82 / PR #83; TBS-19, TBS-07, TBS-12, PROTO-1 | likely | Pavan·W5 | `…/TBC-16.md` | Only the `Message.from` note came from the maintainers' review. The tagline isn't false. The P-4/P-5 fixes are server-side; here they are only documented. The polling-budget note (S-P7) and the payload-wording point (P-2, final text after TBS-03) are now points 6 and 7 of the draft. |
| 15 | PROTO-3 | [FEATURE]: Let callers tell whether `send()` stored a new message or got back an existing one | feature | n/r | S* | protocol C | NEW. Complements #82 / PR #83 | n/r | Pavan·W5 | `…/PROTO-3.md` | none (verified end to end: statuses [201, 200]; a repeated payload is not delivered as new) |
| 16 | TBC-14 | [FEATURE]: Test the uncovered branches in `src/client.ts` | test | low | S | C | NEW. PR #83, #75 | uncertain | Pavan·W5 | `…/TBC-14.md` | (C) :328-329 are unreachable; :281 is also uncovered. **Thresholds** in `vitest.config.ts` moved to Ekansh's TBC-15 (draft retitled; this PR is tests only). |
| 17 | TBC-G3 | [BUG]: `send()` fails with a misleading `405 Method Not Allowed` when the relay URL answers with a 301/302 redirect | bug | low | S | PC | Partially covered: maintainers' review (PR #72) | uncertain | Pavan·W5 | `…/TBC-G3.md` | Not security. Only 301/302/303 break; 307/308 work. Offer a clearer error naming `response.url`, not `redirect: 'manual'` (that alternative is removed from the draft). Browsers show a generic network error instead. |
| 18 | TBC-G1 | [BUG]: Error `instanceof` checks fail when both the CJS and ESM builds are loaded (dual-package hazard) | bug | low | M | PC | NEW. TBC-05 same block | uncertain | Ekansh·W5 | `…/TBC-G1.md` | medium→low; verified on Node 24 only. Literal names alone don't enable narrowing (a union or type guards is needed). The ESM wrapper costs tree-shaking. Keep this PR to `package.json`, `tsup.config.ts` and a new smoke test under `tests/smoke/` (draft narrowed: no README edit, the `errors.ts` guards/literal names are a separate follow-up). |
| 19 | TBC-11 | [FEATURE]: Add `headers` and `fetch` options to `ClientOptions` | feature | low | S | PC | NEW. Server CORS (PROTO-1) limits browser use | uncertain | Pavan·W5 | `…/TBC-11.md` | The CF-Access case is hypothetical. A proxy can be set with undici. Custom headers are blocked cross-origin. Bind the default fetch. |
| 20 | TBC-17 | [FEATURE]: Small cleanup: old "AOSSIE Relay" naming in JSDoc, CONTRIBUTING.md leftovers, outdated checklist entries | cleanup | low | S | PC | NEW. TPL-19 (mojibake) | uncertain | Pavan·W5 | `…/TBC-17.md` | Keep the JSDoc naming, the Python leftover, the test policy and the checklist entries. Mojibake goes to TPL-19 (removed from the draft). Drop the noise items. |
| 21 | TBC-15 | [FEATURE]: Add integration tests that run the SDK against a real ThruBox Server | test | low | M | PC | NEW. Needs TBC-13; server items TBS-03/11/12, PROTO-1 | uncertain | Ekansh·W5 | `…/TBC-15.md` | The CORS cases are browser-only, so a Node test won't catch them. Needs TBC-13. medium→low. Assert current server behaviour. Takes over TBC-14's coverage thresholds (added to the draft; set them after TBC-14's tests merge). |
| folded | TBC-G2 | → TBC-03 | bug | low | S | PC | see TBC-03 | uncertain | (Pavan) | `…/TBC-G2.md` (marked do-not-file) | see row 5 |
| folded | TBC-G5 | → TBC-13 | ci | low | S | PC | see TBC-13 | likely | (Ekansh) | `…/TBC-G5.md` (marked do-not-file) | see row 2 |

### 5.3 ThruBox-Server

| # | ID | Issue title | Type | Sev | Eff | Verdict | Overlap / refs | Fit | Owner·W | Draft | Correction applied (PC) |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | TBS-01 | Docker image is built with end-of-life Go 1.23.12 and runs on end-of-life Alpine 3.20 | security | medium | S | PC | Partially covered: Dependabot #71, #16, #47 stale-closed; TBS-13, TBS-15 | likely | Ekansh·W1 | `issues/ThruBox-Server/TBS-01.md` | "All reachable" is static only (TLS/x509/HTTP2 code is unused). The MaxBytesReader crash isn't triggerable. Target **≥ go1.27.4 or 1.26.11** and bump the `go` directive (GODEBUG). high→medium. |
| 2 | TBS-04 | Rate limiter can be bypassed by rotating IPv6 addresses within one /64 | security | medium | S | PC | NEW. #68 / PR #67 sibling; TBS-12 same file | likely | Ekansh·W3 | `…/TBS-04.md` | Map growth alone is low (bounded to 1-2 minutes of sources). Go maps don't shrink, and cleanup holds the lock about 127 ms per 1M entries. Not every subscriber gets a /64. Refusing new buckets locks out clients. Lead with the bypass. |
| 3 | TBS-03 | `POST /api/messages` accepts addresses of any length: a ~500 KB `to` is stored 5 times, and 2 copies are never removed by the TTL purge | bug | medium | S | PC | NEW. P-2, TBC-08, TBS-07, TBS-16, #69 | likely | Ekansh·W3 | `…/TBS-03.md` | `.`/`..` are readable via `%2E`. Keeping `mailboxes` rows is intentional; the defect is the missing length cap. The address is stored 5× (about 1 MB permanent). The payload-wording sub-item (P-2) is low. |
| 4 | TBS-G2 | [BUG]: Saved `after=<seq>` cursors silently skip new messages after the database is recreated or restored from a backup | bug | medium | M | PC | NEW. #69, PR #67, TBS-17 | likely | Ekansh·W3 | `…/TBS-G2.md` | Every new message with seq ≤ the old cursor is lost for good. An epoch alone doesn't cover restores (also expose `last_seq` or reject `after > last_seq`). The compose named volume keeps data. Chainvoice (SDK 1.0.2) is unaffected. API change: agree on Discord first. |
| 5 | PROTO-1 | [BUG]: CORS preflight rejects `X-API-Key` on a relay without an API key, so browser clients that pass `apiKey` fail every request | bug | medium (draft) | S* | protocol C | NEW. #32 / PR #36 made it conditional on purpose | n/r | Ekansh·W4 | `…/PROTO-1.md` | none (verified in headless Chrome; a 4-line patch makes all calls succeed) |
| 6 | TBS-13 | [FEATURE]: Add a Go CI workflow that runs build, vet, tests (-race), golangci-lint and govulncheck on pull requests | ci | medium | S | PC | NEW. TBS-01 first; same pattern as IDB-14/TBC-13; CodeQL point → TPL-01 | likely | Ekansh·W2 | `…/TBS-13.md` | high→medium. The Docker build already compiles on PRs. Not every workflow is SHA-pinned. govulncheck is blocking only after TBS-01. Fix or exclude the 8 non-test errcheck hits in the same PR. The CodeQL note now says it is reported in Template-Repo (TPL-01) and only follows here as a PR. |
| 7 | TBS-09 | [BUG]: Wrong `X-API-Key` guesses are never rate limited because `APIKeyAuth` runs before the rate limiter | security | low | S | C | Partially covered: maintainers' review Finding 2 (deferred) | likely | Ekansh·W4 | `…/TBS-09.md` | (C) Frame it as tracking Finding 2. State the swap trade-off: keyless requests would use the per-IP budget. A 401 vs 429 oracle remains. |
| 8 | TBS-08 | [BUG]: Any web page can store messages through a visitor's browser: `POST /api/messages` accepts `text/plain`, so no CORS preflight is sent | security | low | S | PC | NEW. #32 / PR #36 | likely | Ekansh·W4 | `…/TBS-08.md` | The cause is the handler accepting any Content-Type; it also happens in the default config. Private-network access checks limit the localhost/LAN impact. The test helper needs the header. A 415 is a behaviour change (CHANGELOG). |
| 9 | TBS-12 | [BUG]: Rate limiter always sends `Retry-After: 60`, browsers cannot read it, and the "sliding window" is a fixed window | bug | low | S | C | NEW. P-4 / S-P5; TBC-04; #75; PROTO-1 (same cors.go block) | likely | Ekansh·W4 | `…/TBS-12.md` | (C) 60 s is a valid overestimate. Fix the comment rather than the algorithm. Keep at least one WARN line per IP per window (the README relies on it). |
| 10 | TBS-11 | [BUG]: POST /api/messages returns nanosecond createdAt/expiresAt, but the stored message (GET, retried POST) has second precision | bug | low | S | C | NEW. P-1 / S-P3 | likely | Ekansh·W4 | `…/TBS-11.md` | (C) Truncate in the handler. Do **not** switch to RFC3339Nano (it breaks string comparison). |
| 11 | TBS-G5 | [BUG]: Rate limiter ignores X-Forwarded-For entries that include a port (IP:port) and limits by the proxy's address instead | bug | low | S | C | NEW. #68 / PR #67; TBS-04 same function | likely | Ekansh·W4 | `…/TBS-G5.md` | (C) No documented setup writes ports (Azure App Gateway does). Fail-closed, so not security. Also handle a bracketed IPv6 address with no port. |
| 12 | TBS-10 | [BUG]: Switching to `ttl_days: 0` leaves already-expiring messages on disk after they expire | bug | low | S | C | NEW. PR #70; TBS-14 if a cmd/relay test is added | likely | Ekansh·W4 | `…/TBS-10.md` | (C) "Forever" means until `ttl_days > 0` again. Bounded volume. |
| 13 | TBS-14 | `.gitignore` entry `relay` silently ignores new files added under `cmd/relay/` | bug | low | S | PC | NEW. Prerequisite for TBS-18 | likely | Ekansh·W1 | `…/TBS-14.md` | medium→low. Repo config, not CI. A trap, not a blocker. It also matches any future `relay` directory. Fix with `/relay`. |
| 14 | TBS-05 (+G4) | [BUG]: Config silently ignores typo'd keys and accepts values that break the relay (`max_payload_size: 0`, `rate_limit: 0`, `ttl_days` > 106751, IPv6 `server.host`, any `storage.driver`) | bug | low | S+S | PC | NEW. PR #36 pattern, PR #34 test | likely | Ekansh·W4 | `…/TBS-05.md` (G4 draft kept as evidence only) | medium→low. Keep "invalid env value = warning" (pinned by a test); fix its wording and log level (logged as INFO). Log `max_payload_size` and auth at startup. **G4 folded in (draft updated, items 6-7, retitled):** `ttl_days` > 106751 wraps to an arbitrary, often past, expiry; `::1` fails at bind (`[::1]` works); strip one bracket pair before `JoinHostPort`. |
| 15 | TBS-G1 | [BUG]: Purging a large expired backlog blocks every send until it finishes; after 5 s, POST /api/messages returns 500 "database is locked" | bug | low | S | PC | NEW. Client #82 / PR #83 for retries | likely | Ekansh·W4 | `…/TBS-G1.md` | About 700k rows on an idle NVMe host (the threshold depends on the machine). The sustained-writer part is a stress test. SQLite does serialize writers. medium→low. The batched purge is the fix; a 503 alone won't help. |
| 16 | TBS-G3 | [BUG]: Docker image sets `RELAY_STORAGE_PATH`, so `storage.path` in a mounted config.yaml is ignored (and the README `docker run` config example mounts no `/data` volume) | bug | low | S | PC | NEW. #27 / PR #35 | likely | Ekansh·W4 | `…/TBS-G3.md` | Not silent (the startup log shows the path). Workaround: `-e RELAY_STORAGE_PATH=`. The data-loss half is a docs nit. Removing the ENV needs a CHANGELOG migration note. |
| 17 | TBS-06 | [BUG]: `/health` counts against the per-IP rate limit (returns 429), and the image has no `HEALTHCHECK` | improvement | low | S | PC | NEW. PR #67 noted the /health slot | likely | Ekansh·W4 | `…/TBS-06.md` | The DB-dead repro was contrived; the real case is a full disk, and `PingContext` wouldn't detect it. Docker doesn't auto-restart unhealthy containers. Use `${RELAY_SERVER_PORT:-${PORT:-3000}}`. Frame it as hardening. |
| 18 | TBS-15 | [BUG]: Stale workflow auto-closes unreviewed Dependabot PRs (incl. golang/alpine base-image bumps), and Dependabot then skips those versions | ci | low | S | PC | NEW. Same `stale.yml` in TBC and Template; TPL-07 | likely | Pavan·W4 | `…/TBS-15.md` | Drop the pinning claim. medium→low. These are version updates, not security updates. Also #19/#23, and #78/#79 are next. No `go` label exists. The fix: stop sync-pr-labels removing `dependencies`, then exempt `dependencies` (the `docker`/`github-actions` exemption option is removed from the draft). |
| 19 | TBS-07 | [FEATURE]: Define how addresses are matched: the checksummed and lowercase forms of one `0x` address are two separate mailboxes | improvement | low | S | PC | NEW. P-6, TBC-16 | likely | Ekansh·W5 | `…/TBS-07.md` | A design gap, not a bug ("dumb mailbox"). File it as a decision/docs proposal. No merge migration by default. |
| 20 | TBS-18 | [FEATURE]: Add tests for untested paths: `cmd/relay` wiring and purge loop, `/health`, `APIKeyAuth`, `HandleCreate` error responses | test | low | M | C | NEW. Needs TBS-14; TBS-13, TBS-09 | likely | Ekansh·W5 | `…/TBS-18.md` | (C) medium→low. `Validate` is at 57% with `-coverpkg`. The valid-key path is also uncovered. |
| 21 | TBS-19 | [BUG]: Docs are out of date: "no tests yet" notes, CORS comments, AGENTS.md layout, missing security notes | docs | low | S | C | NEW. TBC-16 (client side); maintainers' audit Note 1 | likely | Ekansh·W5 | `…/TBS-19.md` | (C) Move the payload wording → TBS-03, sliding window → TBS-12, `storage.driver` → TBS-05. SECURITY.md / private reporting is a settings request (removed from the draft). cors.go:73 is stale too. The Go-prerequisite item is dropped and the draft retitled, because TBS-01 (W1) already updates the README/CONTRIBUTING Go version. Add the unauthenticated-`from` note (S-P11). |
| 22 | TBS-17 | [FEATURE]: Report the server version (`--version`, startup log, `/health`) and let clients discover `max_payload_size` | feature | low | S | PC | NEW. S-P9; PROTO-2 | uncertain | Ekansh·W5 | `…/TBS-17.md` | `.dockerignore` excludes VERSION, so use a build-arg. Page size, TTL and API key are already discoverable. The real gaps are the version and `max_payload_size`. |
| 23 | TBS-20 | [BUG]: Small robustness cleanups: database is opened before the port is bound, `os.Exit` skips `db.Close`, `relay.db` is world-readable, one index is never used | cleanup | low | S | PC | NEW. TBS-13 (errcheck), TBS-01 | uncertain | Ekansh·W5 | `…/TBS-20.md` | Narrowed to the 4 real items: startup order and `os.Exit`, file permissions, the redundant index, the `/health` Encode error. |
| 24 | TBS-16 | [FEATURE]: Configurable storage limits: nothing caps how much one mailbox or the whole database can hold | feature | low | M | PC | NEW. TBS-02, TBS-03, TBS-04 | uncertain | Ekansh·W5 | `…/TBS-16.md` | The repro needed the limiter off (defaults ≈ 15 MiB/min per IP). Per-mailbox caps enable a targeted lockout, so a global DB cap comes first and per-mailbox caps are opt-in. medium→low. |
| 25 | PROTO-2 | [FEATURE]: Machine-readable error responses (stable error codes) so clients don't have to match English text | feature | n/r | M* | protocol C | NEW. P-3 / S-P8; TBC-08 (trim) | n/r | Ekansh·W5 | `…/PROTO-2.md` | none. Needs a maintainer decision on the shape and codes. JSON only when `Accept: application/json` is sent, for backward compatibility. |
| folded | TBS-G4 | → TBS-05 | bug | low | S | PC | see TBS-05 | likely | (Ekansh) | `…/TBS-G4.md` (marked do-not-file) | see row 14 |
| hold | TBS-02 | (no draft) | n/a | n/a | n/a | UV | S-P1 | n/a | Ekansh (re-verify) | none | see §3a |

### 5.4 Template-Repo

| # | ID | Issue title | Type | Sev | Eff | Verdict | Overlap / refs | Fit | Owner·W | Draft | Correction applied (PC) |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | TPL-01 | [BUG]: CodeQL workflow never analyzes code: `set-codeql-language-matrix@v1` has no `matrix` output, so the Analyze job is always skipped | security | **high** | S | C | NEW. #121 masked it; #99; #75 (pinning); upstream #20 | likely | Pavan·W1 | `issues/Template-Repo/TPL-01.md` | (C) It has never run (0 of 182 runs since 2026-03-08). Pinning v1.6.0 alone is incomplete: Go `build-mode: manual`, `standard-language-names`, and the `{"include":[]}` guard. |
| 2 | TPL-G3 | [BUG]: socket.yml skips Socket "Pull Request Alerts" on Dependabot PRs and on Go/Rust/Java/Ruby/.NET manifest changes | security | medium | S | C | NEW. PR #142; MCP-Template proof | likely | Abhinav·W2 | `…/TPL-G3.md` | (C) The PR #178 evidence is irrelevant; MCP-Template is the proof, and it is already affected. The triggerPaths gaps are wider (Pipfile, poetry.lock, uv.lock, bun.lock…). The Project Report check still runs. |
| 3 | TPL-G1 | [BUG]: LICENSE is an incomplete, edited copy of GPL-3.0 that stops at Section 3, so GitHub and Scorecard don't recognize it | bug | medium | S | PC | NEW. PR #2; OpenVerifiableLLM #243 precedent | likely | Abhinav·W1 | `…/TPL-G1.md` | high→medium. Introduced in 3fd9900. 20 derived repos carry it. No CI license check. The fix touches LICENSE only. |
| 4 | TPL-02 | [BUG]: `.gitignore` is GitHub's LaTeX template, so `.env`, `node_modules/` and build output are not ignored in generated repos | bug | medium | S | PC | NEW. PR #167 leaves it; TPL-08 (sync) | likely | Pavan·W1 | `…/TPL-02.md` | high→medium; a bug, not security. Some commit claims were inaccurate. 18 of 24 derived repos carry it. Decide about `.templatesyncignore` (`-X theirs`). |
| 5 | TPL-03 | [BUG]: Issue AI Moderator never checks anything: GitHub Models is retired and `github/ai-moderator` is archived, so every issue is logged as "No spam" | bug | medium | S | PC | Partially covered: PR #168 (bump only) | likely | Pavan·W2 | `…/TPL-03.md` | The root cause is the retirement plus the archived action, so renaming the input won't help. Remove the step or make it label-driven. Edited events are ignored. |
| 6 | TPL-08 | [BUG]: Template Sync PRs reset per-project files (VERSION, CONTRIBUTING.md, MAINTAINERS.md, checklist score) because `.templatesyncignore` only excludes README/LICENSE/workflows | bug | medium | S | PC | NEW. #76/#98 closed; PR #167; MCP-Template #27 | likely | Pavan·W3 | `…/TPL-08.md` | `-X theirs` verified: any differing file is reset and deleted files come back. The monthly failure only affects pre-ca52ca1 repos. A maintainer must merge for damage to happen. Don't ignore `.coderabbit.yaml`. |
| 7 | TPL-15 | [BUG]: version-release.yml pushes the tag before it finds a release, so a missing draft leaves a stray tag and every re-run fails | bug | medium | M | PC | NEW. #139 builds on it; TPL-04 owns the GITHUB_TOKEN point | likely | Pavan·W4 | `…/TPL-15.md` | Manual cleanup still works. The best evidence: the initial-commit race failed in 2 of 3 derived repos. The GITHUB_TOKEN point lives in TPL-04. The range is :113-123. |
| 8 | TPL-05 | [BUG]: gitleaks workflow fails on every Dependabot PR because `GITLEAKS_LICENSE` is not available to Dependabot runs | ci | low | S | PC | NEW. 62780d1; #75; carries TPL-14's permissions half | likely | Pavan·W4 | `…/TPL-05.md` | Don't re-add `pull_request`. medium→low. 33 of 33 Dependabot runs (24 in TBS). Offer the org-level Dependabot secret option. |
| 9 | TPL-04 | [BUG]: OSV release scan can't fail on vulnerabilities (uses the PR-diff workflow), and tags pushed by version-release never trigger it | ci | low | S | PC | NEW. PR #177 (bump only); TPL-09/15 reference it | likely | Pavan·W4 | `…/TPL-04.md` | It can't fail *on a vulnerability*. medium→low (the scheduled scan covers main). Drop `merge_group`. Add `-r`. The GITHUB_TOKEN point is filed here only. |
| 10 | TPL-14 | [BUG]: Workflows put `${{ inputs }}` / `${{ github.ref_name }}` straight into script and shell code, and osv-scanner-release grants `security-events: write` to every job | security | low | S | PC | Partially covered: #75 (pinning), #168, #11; TPL-05 | likely | Pavan·W4 | `…/TPL-14.md` | Pinning goes to a #75 comment. Drop the artipacked noise. The gitleaks permissions go to TPL-05. Do not say it "answers #11". |
| 11 | TPL-13 | [BUG]: Release Drafter fails on merged fork PRs and closed Dependabot PRs, lists PRs twice, and uses options removed or deprecated in v7 | ci | low | S | PC | NEW. #116; IDB PR #96 | likely | Pavan·W4 | `…/TPL-13.md` | The failures are fork merges and closed-unmerged Dependabot PRs (15 of 89). The autolabeler input only warns. Use `semver-increment`, not the deprecated top-level `version-resolver`. |
| 12 | TPL-11 | [BUG]: dependabot.yml gives every Dependabot PR a `chore(deps)(deps):` title, and no entry sets a `cooldown` | ci | low | S | PC | Partially covered: #72, #73 / PR #172 | likely | Pavan·W4 | `…/TPL-11.md` | Failures pause after 15 runs. 11 of 13 ecosystems fail. Counts are 156/139. medium→low. Draft narrowed and retitled to the prefix and cooldown parts; the unused-ecosystem failures (139-166 runs per repo) go as a comment on #72. |
| 13 | TPL-10 | [BUG]: OpenSSF criteria sync job never detects anything: upstream ID regex matches 0 criteria, yet the run passes | bug | low | S | PC | NEW. PR #108; #110 (same workflow) | likely | Pavan·W4 | `…/TPL-10.md` | 27 runs, and the regex never matched. The 404 is noise (use `ossf/`). 10 of the 19 missing IDs aren't Scorecard-delegated, so the maintainers decide. medium→low. |
| 14 | TPL-G2 | [BUG]: Dependency Review `allow-licenses` rejects common permissive licenses (0BSD, BlueOak-1.0.0, Unicode-3.0, Zlib) but accepts GPL-2.0-only | bug | low | S | PC | NEW. PR #105, PR #127 | likely | Abhinav·W4 | `…/TPL-G2.md` | Remove **both** GPL-2.0 entries. The BlueOak ranges were corrected. Not a merge block. medium→low. |
| 15 | TPL-12 | [BUG]: `triage-needed` and Dependabot ecosystem labels are never created, so GitHub silently drops them | bug | low | S | PC | Partially covered: **#85 (assigned to Okd2006)**, PR #51 | likely | Pavan·W5 | `…/TPL-12.md` | Only `triage-needed` and 10 Dependabot labels are really missing. `maintenance` and `python` exist. The stale exemption point is separate. medium→low. Coordinate on #85 first. |
| 16 | TPL-09 | [BUG]: GoReleaser setup can't produce a release: Go build runs by default, CHANGELOG.md is missing, syft isn't installed, release repo is hard-coded | bug | low | S | PC | NEW. PR #122, #120; TPL-04 | likely | Pavan·W5 | `…/TPL-09.md` | Latent (0 runs anywhere). Publishing to the wrong repo fails rather than lands. `check` exits 2. Lead with the defects that survive configuration. medium→low. |
| 17 | TPL-07 | [BUG]: sync-pr-labels mislabels PRs: copies labels from any `#N` in the body, strips Dependabot's `dependencies`, marks bots as first-time contributors, misses the maintain role | bug | low | M | PC | NEW. PR #53, #113; TBS-15 | likely | Pavan·W4 | `…/TPL-07.md` | Admins do get org-member; only the maintain role is missed. The casing doesn't matter. The #178 flip happened within one run. The Dependencies-section claim is narrower. medium→low. |
| 18 | TPL-17 | [BUG]: Best Practices scorer ignores the inline "`[ ]` N/A" boxes, counts checkboxes outside the six categories, and commits a date-only change on most runs | bug | low | S | PC | Partially covered: **#110 (assigned to kpj2006)**; PR #108 | likely | Pavan·W5 | `…/TPL-17.md` | >100% needs extra checkboxes. It commits whenever the date differs. The tag-push / branch-trigger concern is dropped (removed from the draft). Coordinate on #110. |
| 19 | TPL-20 | [BUG]: Scorecard push trigger targets a `dev` branch that doesn't exist; stack-specific CodeRabbit JS/TS rules and a few template leftovers | cleanup | low | S | PC | NEW. #95, #96 | likely | Pavan·W5 | `…/TPL-20.md` | Keep the real items. Drop yamllint, the ESLint boolean, `.gitattributes` and CODEOWNERS. |
| 20 | TPL-19 | [BUG]: Docs: every CONTRIBUTING.md table-of-contents link is broken, U+FFFD characters in headings, broken PR-template link, and dead links in `initial-issues.json` | docs | low | S | PC | Partially covered: PR #167 (README part), PR #63 | likely | Abhinav·W4 | `…/TPL-19.md` | README U+FFFD and alt text are already in PR #167. Placeholders are intentional. The CoC comes via the org fallback. Danger only warns. |
| 21 | TPL-18 | [BUG]: The template fails its own pre-commit hooks (30 files rewritten, `check-json` fails on `.vscode/extensions.json`), and no CI job runs them | ci | low | S | C | NEW. #104 closed not_planned | uncertain | Pavan·W6 (**last**) | `…/TPL-18.md` | (C) Safe whitespace changes. It touches 30 files, so it goes after every other Template PR. |
| 22 | TPL-06 | [BUG]: Danger "PR Template Check" comment links to the wrong page, flattens its nested lists, and is posted on every Dependabot PR | bug | low | S | PC | Partially covered: **#112 (assigned)**, #176 | uncertain | Pavan·W5 | `…/TPL-06.md` | The core bug is #112, so don't refile. The bot skip is about #176. The "404" is unverified (it's a 302). medium→low. **Filed as a comment on #112, plus evidence on #176.** The draft is now written as those two comments, not as an issue. |
| 23 | TPL-G5 | [BUG]: `.coderabbit.yaml` path_instructions `**/*.{py}` and `**/*.test.{sol}` never match any file, so the Python and Solidity-test rules never apply | bug | low | S | PC | NEW. PR #61 (flagged, not fixed); #95, #96 | uncertain | Pavan·W5 | `…/TPL-G5.md` | The same globs live in 11 or more other AOSSIE repos. The matcher semantics come from the docs, not a runtime check. The `.t.sol` block adds to the `**/*.sol` rules. |
| 24 | TPL-G4 | [FEATURE]: security-remediation: warn before publishing the exploit details of unfixed High/Critical findings, and separate "accepted risk" from "fix pending" | improvement | low | S | PC | NEW. PR #167 | uncertain | Abhinav·W5 | `…/TPL-G4.md` | Publishing with approval is the intended design. The real gaps: severity is unused, the prompt doesn't mention the exploit text, and there's no redaction option. security/medium→improvement/low. |
| 25 | TPL-16 | [FEATURE]: Good First Issue form links to the non-existent `AOSSIE-Org/TODO` repo in every derived repo; add a placeholder check and a post-creation setup checklist | feature | low | M | PC | Partially covered: #102 (unassigned), PR #103 | uncertain | Pavan·W5 | `…/TPL-16.md` | The secrets part was overstated. #102 covers placeholder validation. Initial issues not being created is the maintainers' choice. medium→low. Pair with #102. |

---

## 6. Existing GitHub issues the team can pick up directly

Checked against `_gh/*-issues.json` and `_gh/*-prs.json` (snapshot 2026-11-19): **open, no assignee, and no open PR that references the issue**. The snapshot has no comments, so the "asked to work on it" notes come from the audit reports. **Re-check live and ask on the issue and on Discord before claiming.**

| Repo | Issue | Status | Recommendation | Who |
|---|---|---|---|---|
| IndexedDB-Import-Export | [#131 Blob and File values are silently lost in exportDB](https://github.com/AOSSIE-Org/IndexedDB-Import-Export/issues/131) | Unassigned, no PR. The author asked about the approach and got no answer. | **Pick up** (M). Ask the author first. Agree the approach on Discord: async export after the cursor transaction finishes, or fail loudly. Coordinate with IDB-07/IDB-G1. | Abhinav·W1 |
| ThruBox-Client | [#75 RelayRateLimitError.retryAfter is negative when server sends Retry-After: -5](https://github.com/AOSSIE-Org/ThruBox-Client/issues/75) | Unassigned, no PR. The reporter asked to work on it in a comment. | **Pick up only if the reporter isn't working on it** (S). Clamp with `Math.max(0, n)` and reject non-finite values. Pair with TBC-04. | Pavan·W2 |
| ThruBox-Server | none | 0 open issues | n/a | n/a |
| Template-Repo | [#139 Add Automated NPM Publish Workflow](https://github.com/AOSSIE-Org/Template-Repo/issues/139) | Unassigned, no PR | Stretch, after TPL-15 (it builds on version-release). | Pavan (stretch) |
| Template-Repo | [#102 Template initialization script to replace placeholders](https://github.com/AOSSIE-Org/Template-Repo/issues/102) | Unassigned; PR #103 stale-closed | Stretch, paired with TPL-16. | Pavan (stretch) |
| Template-Repo | [#72 Wildcard/Auto-Discovery Support for Dependabot](https://github.com/AOSSIE-Org/Template-Repo/issues/72) | Unassigned, `planned` | Stretch, after TPL-11. Would also fix the failing-ecosystem runs. Post the failed-run counts (139-166 per repo, cut from the TPL-11 draft) here as a comment when TPL-11 is filed. Verify `directories:` globs on a test repo first. | Pavan (stretch) |
| Template-Repo | [#11 security check for sync pr labels](https://github.com/AOSSIE-Org/Template-Repo/issues/11) | Unassigned, `planned`, vague | No PR. Optionally post the TPL-14 analysis (no PR checkout; data passed via context/env) as a comment. | Pavan (comment) |
| Template-Repo | [#176 PR Template Validation re-runs on every edit](https://github.com/AOSSIE-Org/Template-Repo/issues/176) | Unassigned, no PR, but the reporter asked to be assigned 9 days ago | **Don't take it.** Add the TPL-06 bot-PR evidence as a comment. | n/a |
| Template-Repo | [#96 Add more path_filters to .coderabbit.yaml](https://github.com/AOSSIE-Org/Template-Repo/issues/96) | Unassigned | Low value. The maintainer wants a minimal, use-case-driven set. Skip. | n/a |
| Template-Repo | [#66 Optional role-based first-claim assignment](https://github.com/AOSSIE-Org/Template-Repo/issues/66) | Unassigned | Needs maintainer design sign-off. Skip. | n/a |
| Template-Repo | [#135 GUAC supply-chain knowledge graph](https://github.com/AOSSIE-Org/Template-Repo/issues/135) | Unassigned, `planned` | Research-sized. Skip. | n/a |
| Template-Repo | [#55 Welcome Bot](https://github.com/AOSSIE-Org/Template-Repo/issues/55) | Unassigned, but kpj2006 said he is working on it | Skip. | n/a |
| Template-Repo | [#16 Template enhancement ideas](https://github.com/AOSSIE-Org/Template-Repo/issues/16) | Idea thread that says "Do not open a pull request" | Skip. | n/a |

Not pickable: IDB #132, #126, #124, #114, #95, ThruBox-Client #80, #82 and Template #73 all have open PRs. Template #112, #110, #95, #91, #85 and #75 are assigned; CONTRIBUTING says to avoid assigned issues even when the assignee is inactive. Review opportunities instead: PR #117 (IDB-03 gap), PR #130 (merge path, IDB-02), PR #81 (needs a test), PR #83 (fails `tsc` on main, see TBC-12).

---

## 7. Assignment

### 7a. Home areas (rule a: no two people's in-flight work touches the same file)

| Person | Owns these files | Cross-repo pattern they own (rule d) |
|---|---|---|
| **Abhinav** | IDB `src/core/*`, `src/serialization/*`, `src/types/*`, `src/utils/file.ts`, `src/index.ts`, `tests/*`, `README.md`, `AGENTS.md`, `CONTRIBUTING.md`. Template `LICENSE`, `socket.yml`, `dependency-review-action.yml`, `skills/*`, `CONTRIBUTING.md`, `.github/PULL_REQUEST_TEMPLATE.md`, `.github/initial-issues.json`. `LICENSE` in IDB, TBC and TBS. | LICENSE (TPL-G1 plus 3 derived repos); dependency-review allow-list (TPL-G2 plus 3 derived); CONTRIBUTING mojibake follow-ups |
| **Pavan** | TBC `src/client.ts`, `src/types.ts`, `src/errors.ts`, `src/index.ts`, `tests/client.test.ts`, `README.md`, `BestPracticesChecklist.md`. All other Template-Repo files. The derived copies of template workflows: `codeql.yml`, `gitleaks-scanning.yml`, `scorecard.yml`, `stale.yml`, `sync-pr-labels.yml`, `.github/dependabot.yml`. | Template workflow fixes and their derived-repo follow-ups; stale/labels (TBS-15 plus TPL-07) |
| **Ekansh** | All of ThruBox-Server except `LICENSE` and the template-workflow copies above. In IDB and TBC: `package.json`, `package-lock.json`, `tsconfig.json`, `.prettierrc`, `tsup.config.ts`, `vitest.config.ts`, `src/utils/ssr.ts` (IDB), `.github/workflows/ci.yml`, `.github/workflows/version-release.yml`, `tests/integration/*`, `tests/smoke/*`. | PR CI in 3 repos; CJS types in 2 repos; lint/format-before-CI |

Shared files are handed over **in wave order only**. The script check found **0 same-wave overlaps between different people**, using the narrowed scopes now written into the drafts: IDB-18 = `package.json`/lock only; TBC-13 (+G5) = workflows, `CONTRIBUTING.md` and `tests/smoke/*` (no README/AGENTS); TBC-G1 = `package.json`, `tsup.config.ts` and `tests/smoke/*` (no README, `errors.ts` or `index.ts`); TBC-14 = tests only (thresholds in TBC-15). With the raw verdict file lists there were 6 collisions (IDB README in W4, TBC README in W2 and W5, TBC `src/index.ts`, `tests/client.test.ts` and `vitest.config.ts` in W5), so keep those scopes. The only files that cross homes, and their order:

| File | Order of PRs |
|---|---|
| TBC `src/client.ts`, `src/errors.ts`, `src/index.ts`, `tests/client.test.ts` | Ekansh **TBC-12 (W1, reformat)** must merge before any of Pavan's TBC code (W2 onward) |
| TBC `CONTRIBUTING.md` | Ekansh TBC-13 (W2) → Pavan TBC-17 (W5) → Abhinav mojibake follow-up (W6) |
| IDB `package.json` | Ekansh IDB-14 (W2), IDB-15 and IDB-18 (W4) → Abhinav IDB-16 (W5) |
| Template `BestPracticesChecklist.md` | Abhinav TPL-19 (W4) → Pavan TPL-17 (W5) |
| Template (30 files) | Pavan TPL-18 goes last (W6), after every other Template PR from anyone has merged |

### 7b. Balance

| Person | W1 | W2 | W3 | W4 | W5 | W6 | Total points | New issues | Existing |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---|
| Abhinav | 5 | 5 | 5 | 9 | 23 | 0 | **47** | 24 | #131 |
| Pavan | 2 | 5 | 3 | 22 | 15 | 1 | **48** | 36 (+1 comment) | #75 |
| Ekansh | 3 | 5 | 5 | 15 | 19 | 0 | **47** | 32 | none |

Waves 1-3 are deliberately small: a quick first PR, then the high-value bugs. Wave 4 is a rolling queue of likely-fit work. Wave 5 is mostly items that need a maintainer decision or have uncertain fit, so it is the first place to cut if time runs short. Abhinav's wave 5 is heavy because most IDB features need design agreement.

### 7c. Ordered lists

**Abhinav: IndexedDB library, license and docs**

| Wave | Items (in order) | Pts | Notes |
|---|---|---:|---|
| W1 | **TPL-G1** (S, quick first PR) · **IDB-02** (S) · **#131** (M, claim) | 5 | IDB-02: mirror the cancel-on-blocked pattern from PR #130. #131: ask the author first. |
| W2 | **IDB-01** (M) · **IDB-09** (S) · **TPL-G3** (S) | 5 | IDB-01: comment on PR #117 first (same function). After TPL-G1 is accepted: LICENSE follow-up PRs in IDB, TBC and TBS (0 pts). |
| W3 | **IDB-07** (M, narrowed) · **IDB-13** (S) · **IDB-17** (S) | 5 | All three edit `serialization/index.ts`, so do them one after another. Rebase IDB-13 against PR #127. IDB-17 leaves the `idb` wording alone. |
| W4 | IDB-G5 (M) · IDB-05 (S) · IDB-06 (M) · TPL-G2 (S) · TPL-19 (S) | 9 | IDB-05 and IDB-06 share the merge path, so one after another. |
| W5 | IDB-16 (S) · IDB-G1 (M) · IDB-08 (S) · IDB-10 (M) · IDB-G6 (S) · IDB-11 (S) · IDB-G3 (M) · IDB-G2 (M) · IDB-12 (M) · IDB-G4 (M) · TPL-G4 (S) | 23 | IDB-16 after Ekansh's IDB `package.json` PRs merge. IDB-G1 after PR #128 merges. IDB-10, IDB-G3 and IDB-G4 need Discord design agreement. Update the README/AGENTS `idb` wording once IDB-18 is decided. Dependency-review follow-ups in IDB, TBC and TBS after TPL-G2. |
| W6 | CONTRIBUTING mojibake follow-ups in IDB, TBC and TBS | 0 | After TPL-19 merges, and after TBC-13, TBC-17 and TBS-19 (same files). |

**Pavan: ThruBox-Client SDK and Template-Repo automation**

| Wave | Items (in order) | Pts | Notes |
|---|---|---:|---|
| W1 | **TPL-01** (S, quick first PR, the only high) · **TPL-02** (S) | 2 | No TBC code until Ekansh's TBC-12 merges. Ask on TBC #75 whether the reporter is still working on it. |
| W2 | **TBC-09** (M) · **#75** (S, if free) · **TPL-03** (S) | 5 | TBC-09 and #75 both edit `poll()`/`request()` (same owner). |
| W3 | **TBC-03 (+G2)** (S) · **TBC-04** (S) · **TPL-08** (S) | 3 | TBC-04 caps its own delay. After TPL-01 is accepted: CodeQL follow-up PRs in IDB, TBC and TBS. |
| W4 | TPL-15 (M) · TBC-07 (S) · TBC-01 (S) → TBC-10 (M) · TBC-02 (S) · TBC-06 (S) · TBC-08 (S) · TBC-G4 (S) · TPL-05 (S) · TPL-04 (S) · TPL-10 (S) · TPL-14 (S) · TPL-13 (S) · TPL-11 (S) · TBS-15 (S) · TPL-07 (M) | 22 | TPL-14 only after TPL-01, TPL-04 and TPL-10 (shared files). TPL-04 first only swaps the reusable workflow; the version-release dispatch goes with or after TPL-15. TPL-11: coordinate with PR #172. TBC-06 and TBC-07: rebase on #81/#83. |
| W5 | TBC-16 (S) · PROTO-3 (S) · TBC-14 (S) · TBC-G3 (S) · TBC-11 (S) · TBC-17 (S) · TPL-12 (S) · TPL-09 (S) · TPL-17 (S) · TPL-20 (S) · TPL-06 (S, comment) · TPL-G5 (S) · TPL-16 (M) | 15 | TPL-12: ask on #85 first. TPL-17: coordinate on #110, after TPL-19. Follow-ups: the TBC `stale.yml`/`sync-pr-labels.yml` change (after TBS-15), gitleaks in TBS and TBC (after TPL-05). |
| W6 | **TPL-18** (S, last) | 1 | Only after every other Template PR has merged. Also the Scorecard and Dependabot follow-ups in derived repos. |
| Stretch | #139 (after TPL-15) · #102 (with TPL-16) · #72 (after TPL-11) | n/c | Not counted in the balance. |

**Ekansh: ThruBox-Server, CI and packaging**

| Wave | Items (in order) | Pts | Notes |
|---|---|---:|---|
| W1 | **TBC-12** (S, quick first PR) · **TBS-01** (S) · **TBS-14** (S) | 3 | TBC-12 unblocks Pavan and TBC-13; use `singleQuote: false` for a small diff. TBS-01: go ≥1.27.4 or 1.26.11 and bump the `go` directive. TBS-14 unblocks tests in `cmd/relay`. |
| W2 | **IDB-14** (S) · **TBS-13** (S) · **TBC-13 (+G5)** (M) | 5 | The CI pattern. Each starts green: the IDB `ssr.ts` fix in the same PR; TBS errcheck fixed or excluded in the same PR, govulncheck blocking only once TBS-01 has merged; TBC after TBC-12. |
| W3 | **TBS-04** (S) · **TBS-03** (S) · **TBS-G2** (M) | 5 | TBS-G2 changes the API (DB id / `last_seq`), so agree on Discord #thrubox first. |
| W4 | PROTO-1 (S) · TBS-12 (S) · TBS-09 (S) · TBS-08 (S) · TBS-11 (S) · TBS-G5 (S) · TBS-10 (S) · TBS-05+G4 (2) · TBS-G1 (S) · TBS-G3 (S) · TBS-06 (S) · IDB-15 (S) · IDB-18 (S) · TBC-05 (S) | 15 | `ratelimit.go` one at a time: TBS-04 → TBS-12 → TBS-G5 → TBS-09 → TBS-06. `cors.go`: PROTO-1 → TBS-12. IDB-18: ask on Discord whether to drop or adopt `idb`. |
| W5 | TBS-18 (M) · TBS-19 (S) · TBS-07 (S) · TBS-17 (S) · TBS-20 (S) · TBS-16 (M) · PROTO-2 (M) · TBC-G1 (M) · TBC-15 (M) | 19 | TBS-18 after TBS-14. TBS-07, TBS-16 and PROTO-2 need a maintainer decision. TBC-G1 after TBC-05. TBC-15 after TBC-13 (and it takes over TBC-14's coverage thresholds). |
| Hold | TBS-02: re-verify, then draft if it holds up | n/c | The server half of S-P1. Needs a maintainer decision on the response shape. |

Every person gets a quick first PR (TPL-G1, TPL-01, TBC-12) and meatier work: Abhinav IDB-01, #131, IDB-07, IDB-G5; Pavan TBC-09, TPL-15, TPL-07, TBC-10; Ekansh TBC-13, TBS-G2, TBS-18, PROTO-2.

### 7d. Ordering dependencies (rule e)

| Before | After | Why |
|---|---|---|
| TBC-12 (Ekansh) | TBC-13 (Ekansh); all of Pavan's TBC code | CI must start green, and the reformat would conflict with everything else. |
| `src/utils/ssr.ts` Prettier fix | IDB-14 enforcing `format:check` | Same PR. |
| TBS-01 | Blocking govulncheck in TBS-13 | 32 stdlib vulnerabilities otherwise keep CI red (or use `continue-on-error` until then). |
| 8 non-test errcheck hits fixed or excluded | TBS-13 lint job | Same PR. TBS-20 is narrowed accordingly. |
| TBS-14 (`/relay`) | TBS-18 and any `cmd/relay/*_test.go` (TBS-10) | New files there are silently ignored by git. |
| TBC-13 | TBC-15, coverage thresholds | Integration tests need the workflow. |
| TBC-14 (Pavan, W5) | TBC-15's coverage thresholds (Ekansh, W5) | Thresholds are set just below the numbers measured after TBC-14's tests land. |
| IDB-18 decided | README/`AGENTS.md` `idb` wording (Abhinav, W5) | The wording follows the drop-or-adopt decision. |
| TBC-05 | TBC-G1 | Same `exports` block. |
| TBC-01 | TBC-10 | Shared signal plumbing. |
| #75 (clamp) | TBC-04 (honour `retryAfter`) | The delay must never be negative. |
| PR #128 (external) | IDB-G1, any throw-based catch-all | A throw inside `serialize()` currently hangs `exportDB`. |
| PR #117 (external) | IDB-01 | Same function; coordinate. |
| TPL-01, TPL-04, TPL-10 | TPL-14 | Shared workflow files. |
| TPL-15 | #139, the TPL-04 version-release dispatch | Builds on the fixed release flow. |
| TPL-19 | TPL-17; derived CONTRIBUTING follow-ups | Shared file, and the template comes first. |
| Every Template PR | TPL-18 | 30-file whitespace rewrite. |
| Each template fix accepted | Its derived-repo follow-up PRs | Workflows and LICENSE are not template-synced. |
| TBS-G2, PROTO-2, TBS-17 decided | SDK follow-ups (no issue yet) | The client follows the server's contract. |

---

## 8. Filing plan

Rules for every wave:

- At most **3 open issues per person** (claims of existing issues count).
- At most **3 new issues per repo from the whole team per wave**, so no maintainer is flooded.
- Open the next issue only when an earlier one is closed, or assigned with a PR up.
- Re-check live GitHub for new duplicates and PRs before each filing.
- Post every new issue in the project's Discord channel the same day.

### Wave 1 (file now)

| Person | Files | Repo | Why these first |
|---|---|---|---|
| Abhinav | **TPL-G1** (new) | Template-Repo | Medium, verified against the GitHub license API and Scorecard, NEW, one-file fix with a merged precedent (OpenVerifiableLLM #243). |
| Abhinav | **IDB-02** (new) | IndexedDB | Medium, reproduced in Chrome and fake-indexeddb, NEW, small. |
| Abhinav | **#131** (claim existing) | IndexedDB | Existing unassigned bug with no PR. Ask the author first. |
| Pavan | **TPL-01** (new) | Template-Repo | The only high-severity item. CONFIRMED, NEW, affects every derived repo. |
| Pavan | **TPL-02** (new) | Template-Repo | Medium, reproduced, NEW, matches fixes the maintainers already made in derived repos. |
| Pavan | Comment on **#75** | ThruBox-Client | Ask whether the reporter is still working on it. Claim it in W2 if not. |
| Ekansh | **TBC-12** (new) | ThruBox-Client | CONFIRMED, blocks CI and Pavan's work, quick. |
| Ekansh | **TBS-01** (new) | ThruBox-Server | Medium security hygiene, verified, NEW as an issue (the Dependabot PRs were only bot-closed). |
| Ekansh | **TBS-14** (new) | ThruBox-Server | One-line fix, prerequisite for server tests. |

Per repo in wave 1: Template-Repo 3 new; IndexedDB 1 new plus 1 claim; ThruBox-Client 1 new plus 1 comment; ThruBox-Server 2 new.

### Later waves

| Wave | Abhinav | Pavan | Ekansh |
|---|---|---|---|
| W2 | IDB-01, IDB-09, TPL-G3 | TBC-09, #75 (claim), TPL-03 | IDB-14, TBS-13, TBC-13 (+G5) |
| W3 | IDB-07 (narrowed), IDB-13, IDB-17 | TBC-03 (+G2), TBC-04, TPL-08 | TBS-04, TBS-03, TBS-G2 (Discord first) |
| W4 (rolling, ≤3 open) | IDB-G5, IDB-05, IDB-06, TPL-G2, TPL-19 | TPL-15, TBC-07, TBC-01, TBC-10, TBC-02, TBC-06, TBC-08, TBC-G4, TPL-05, TPL-04, TPL-10, TPL-14, TPL-13, TPL-11, TBS-15, TPL-07 | PROTO-1, TBS-12, TBS-09, TBS-08, TBS-11, TBS-G5, TBS-10, TBS-05 (+G4), TBS-G1, TBS-G3, TBS-06, IDB-15, IDB-18, TBC-05 |
| W5 (gated, ≤3 open) | IDB-16, IDB-G1, IDB-08, IDB-10, IDB-G6, IDB-11, IDB-G3, IDB-G2, IDB-12, IDB-G4, TPL-G4 | TBC-16, PROTO-3, TBC-14, TBC-G3, TBC-11, TBC-17, TPL-12 (after #85), TPL-09, TPL-17 (after #110), TPL-20, TPL-06 (comment on #112), TPL-G5, TPL-16 | TBS-18, TBS-19, TBS-07, TBS-17, TBS-20, TBS-16, PROTO-2, TBC-G1, TBC-15 |
| W6 | Derived CONTRIBUTING follow-ups | TPL-18 (last) and derived follow-ups | n/a |

**Discuss on Discord before opening the issue** (design or direction questions): TBS-07, TBS-16, PROTO-2, TBS-G2, IDB-10, IDB-G3, IDB-G4, IDB-18 (drop or adopt `idb`), TPL-16 with #102, TPL-11's unused-ecosystems part (#72), TPL-03 (remove the step vs label-driven).

**Draft edits (applied 2026-11-20, during the backlog review):**
- IDB-07: CryptoKey sentence, the throw-on-unknown fix item and the matching "Expected" sentence removed.
- IDB-18: retitled; format part, README/AGENTS edits and the mojibake/advisory notes removed.
- IDB-17: `idb` wording item removed. IDB-05: Impact Medium → Low. IDB-08: keep `recordCounts` as is. IDB-12: paged-`getAll()` follow-up removed.
- TBC-03: TBC-G2's `receive()` shape check, `null`/no-`after` cases and tests merged in. TBC-G2, TBC-G5 and TBS-G4 carry a do-not-file note at the top.
- TBC-13: TBC-G5's CONTRIBUTING prerequisite fix added as item 4; `engines`/README left to a separate breaking-change PR.
- TBC-14: retitled, thresholds removed. TBC-15: thresholds added. TBC-G1: narrowed to `package.json`, `tsup.config.ts`, smoke test. TBC-17: mojibake removed. TBC-G3: `redirect: 'manual'` alternative removed. TBC-16: polling-budget and payload-wording points added.
- TBS-05: TBS-G4 merged in (items 6-7, repro, logs, fixes) and retitled. TBS-19: Go-version and private-reporting items removed, retitled. TBS-13: CodeQL note points to the Template-Repo issue. TBS-15: `docker`/`github-actions` exemption option removed.
- TPL-06: rewritten as a comment for #112 and a comment for #176 (not an issue). TPL-11: narrowed to prefix + cooldown and retitled. TPL-17: branch-trigger item removed.
- PROTO drafts: use as they are.

---

## 9. Process checklist (from each repo's CONTRIBUTING.md and AGENTS.md)

1. **Join Discord** ([invite](https://discord.gg/hjUhu33uAn)) and link GitHub through the Gitcord bot (Template CONTRIBUTING). Maintainers don't watch GitHub notifications, and "PRs without Discord updates will not be reviewed or may face delays."
2. **Check for duplicates first**: existing issues *and* existing PRs, including PRs that don't mention an issue (IDB/TBC CONTRIBUTING, "Issue Assignment").
3. **Issue first.** Use the bug or feature template. Keep the text concise and factual, and rewrite each draft in your own words.
4. **Post the issue in the project's Discord channel** (mandatory):
   - IndexedDB: `#indexeddb-import-export`
   - ThruBox-Client and ThruBox-Server: `#thrubox`
   - Template-Repo: "the relevant channel"
5. **Wait to be assigned** (preferred). One contributor per issue. Never work on an issue assigned to someone else, even an inactive one.
   - IDB and TBC: if nobody is assigned and there's no active PR for 2+ days, say you intend to start under the issue, then start.
   - Template-Repo: small changes may skip the issue and assignment.
   - ThruBox-Server: issue first, assignment preferred.
6. **Branch** from up-to-date `main` (never work on `main`/`dev`). Use conventional commit prefixes (`fix:`, `feat:`, `docs:`, `chore:`…). Rebase on `upstream/main`.
7. **Run the checks locally before pushing:**
   - IndexedDB: `npm run build`, `npm test`, `npm run lint`, `npm run format:check`
   - ThruBox-Client: `npm test`, `npm run coverage`, `npm run lint`, `npm run format:check`
   - ThruBox-Server: `gofmt`, `go vet ./...`, `go test ./...` (add `-race` once TBS-13 lands)
   - Template-Repo: pre-commit hooks
   - Until IDB-14 and TBC-12 merge, `format:check`/`lint` fail on main. Say "pre-existing on main" in the PR rather than fixing it in an unrelated PR.
8. **Add tests with the change** (AGENTS.md):
   - IDB: Vitest with `fake-indexeddb`/`happy-dom`, not a real browser
   - TBC: Vitest
   - TBS: `_test.go` next to the code, and fix `.gitignore` first for `cmd/relay`
9. **No new runtime dependencies** without discussing it in an issue first: ThruBox-Client has zero runtime deps; IDB and TBS keep a minimal list.
10. **PR description**: link the issue (`Fixes #N`), fill in the PR template and checklist, and **disclose AI assistance** with its scope (for example, "Drafted the regression test with Claude; reviewed and verified locally"). Make sure you understand every AI-generated line.
11. **Post the PR in the Discord channel.** Tag maintainers on Discord if it sits unattended for 1-2 weeks. Help review existing PRs instead of opening duplicates.
12. **After merge**: update this backlog, close the loop on Discord, and only then open your next issue (≤3 open per person).
