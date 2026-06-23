# fn-65 land: accept a bot "reviewed-clean" comment as the silence signal (not just formal reviews)

> **Origin:** found live while landing PR #176 (fn-64) on 2026-06-17. The `silence` gate read 0 automated reviews of the final head and would have stalled at `NEEDS_HUMAN`; the merge only happened because a human recognized Codex's clean-review *comment* and merged by judgment. Captured by Gordon.

## Goal & Context

`/flow-next:land`'s `silence` review signal (the default) is satisfied by "an automated review of the current head + zero unresolved threads + patience window elapsed." Land detects automated reviews **only** via the formal reviews API (`gh api repos/<o>/<r>/pulls/<n>/reviews`, workflow.md:231) — the `AUTO_REVIEW_CURRENT` loop.

But the Codex GitHub reviewer (`chatgpt-codex-connector[bot]`) only files a **formal review** when it has findings. On a **clean** pass it posts an **issue comment** instead — e.g. *"Codex Review: Didn't find any major issues. **Reviewed commit:** `8ff0e50f46`"* — which never appears in the reviews API. So `AUTO_REVIEW_CURRENT` reads `0`, and land reports `AWAITING_REVIEW` → after the window, `NEEDS_HUMAN` ("no automated review arrived within the patience window").

Net effect: **an unattended land loop will NOT auto-merge a PR whose only change since the last finding was approved-by-silence** — the exact converged-clean state land exists to merge. This bit the fn-64 land (PR #176): CI green, P2 fixed + thread resolved, Codex re-reviewed the new head **clean via comment**, but land's reviews-API-only check couldn't see it. A human merged on judgment; the loop alone would have dead-ended.

## Architecture & Data Models

The fix lives entirely in the `silence`-signal evaluation in `plugins/flow-next/skills/flow-next-land/workflow.md §2.6` — the host-agent gate read; the only flowctl touch is seeding one config default.

- **Current detection (reviews-only).** `AUTO_REVIEW_PRESENT` / `AUTO_REVIEW_CURRENT` are computed from `pulls/<n>/reviews` filtered to `[bot]`-suffix logins + `land.automatedReviewers`, head-current via `commit_id == HEAD_OID || submitted_at > LAST_PUSH`.
- **Add a second evidence source — clean-review comments (`silence` only, gated explicitly).** Run the scan ONLY when `REVIEW_SIGNAL == silence` (it must not run on the `approve`/`<login>` paths, and must not perturb the draft review-trigger branch that also reads `AUTO_REVIEW_CURRENT`). Scan `issues/<n>/comments` for a comment by an automated-reviewer login whose body matches the clean-review pattern AND yields a head-current SHA token. A match only ever **sets** `AUTO_REVIEW_CURRENT = 1` (never resets — the reviews-API path is untouched).
- **SHA extraction is explicit + empty-guarded (CRITICAL).** Never test `[[ "$HEAD_OID" == "$comment_sha"* ]]` against an unextracted var — an empty `comment_sha` makes that prefix-glob spuriously TRUE and would satisfy the gate on any matching comment. Instead, **prefer the SHA on the reviewed-commit marker line** (e.g. the token following `Reviewed commit`), falling back to body hex tokens (`grep -oE '[0-9a-fA-F]{7,40}'`); lowercase them, and count the comment as head-current **iff ≥1 token is non-empty, ≥7 chars, AND a prefix of `HEAD_OID`** (`[[ "$HEAD_OID" == "$token"* ]]`). No qualifying token → ignored (covers the "clean comment with no SHA" case). Comment analog of the reviews path's `commit_id == HEAD_OID`.
- **Pattern config — precise contract (built-in default; explicit empty disables).** New optional key `land.cleanReviewCommentPattern`. `get_default_config()` seeds a **structured built-in ERE** (requires the review marker AND the clean phrase near the SHA — e.g. `Reviewed commit` together with `Didn'?t find any (major )?issues|No (major )?issues found` — NOT a bare "no issues" match). The workflow `cfg` read resolves: **`null`/missing (old repo, unseeded) → fall back to the same built-in default; explicit empty string `""` → the comment scan is DISABLED; any other value → use it.** This is the only contract that lets a user actually turn the feature off — "empty → default fallback" would make it un-disableable. The `land.automatedReviewers` allowlist additionally gates which logins count.
- **Dry-run / report evidence.** On a comment-driven satisfaction, set `AUTO_REVIEW_SOURCE=comment` and capture `AUTO_REVIEW_EVIDENCE` (comment author + matched SHA prefix); `--dry-run` and the verdict report surface it so a transcript reader sees WHY the gate passed.
- **Threads still gate independently.** This only supplies the "a reviewer saw this head" signal; `UNRESOLVED == 0` and `WINDOW_ELAPSED == 1` are unchanged, so a clean-comment never bypasses an open thread.

## API Contracts

- **flowctl:** one new optional config key `land.cleanReviewCommentPattern` (string ERE; structured default; empty disables the comment path), seeded in `get_default_config()` and documented alongside `land.reviewSignal` / `land.automatedReviewers`. No new handler, no merge-mechanics change.
- **workflow.md §2.6:** after the reviews-API loop, add a comment-scan loop (`gh api --paginate repos/$OWNER_REPO/issues/<n>/comments --jq '.[]|[.user.login,.created_at,.body]|@tsv'`) that sets `AUTO_REVIEW_CURRENT=1` on (login ∈ automated) ∧ (body matches pattern) ∧ (an extracted hex token, ≥7 chars, is a prefix of `HEAD_OID`). Sets `AUTO_REVIEW_SOURCE=comment` + `AUTO_REVIEW_EVIDENCE`; `--dry-run` reports the matched comment as the satisfying evidence.

## Edge Cases & Constraints

- **Stale clean-comment:** a "Reviewed commit `<old-sha>`" comment from before the last push must NOT satisfy the gate — require the CURRENT head SHA in the body (prefix match handles GitHub's abbreviated SHA).
- **Bot with no SHA in its clean comment:** if the matched comment does not name a resolvable head SHA, do NOT count it (can't prove head-current) — fall through to the existing wait/NEEDS_HUMAN path. Conservative by design.
- **`approve` and `<login>` signals unaffected (scoped out):** `land.reviewSignal=approve` still requires a formal `APPROVED`; `<login>` still reads that login's `latestReviews`. The comment path is `silence`-only — extending the same comment-evidence treatment to `<login>` (a `<login>` bot that comments-clean) is a deliberate **follow-up**, kept out of this small spec to avoid touching the `<login>` evaluation path.
- **Comment pagination:** scan all comment pages (`gh api --paginate issues/<n>/comments`) so a clean comment isn't missed on a busy PR.
- **No new merge license:** the change only lets land *see* a clean review it currently misses — CI-green, zero-unresolved-threads, and window-elapsed gates are unchanged. It never merges with an open thread or a red check.
- **Backward compatible:** empty/unset `land.cleanReviewCommentPattern` with a non-Codex reviewer = today's behavior exactly.

## Acceptance Criteria

- **R1:** When the only head-current review evidence is an automated-reviewer **issue comment** matching the clean-review pattern AND naming the current head SHA, the `silence` signal is satisfied (given CI green, 0 unresolved threads, window elapsed) and land merges.
- **R2:** A clean-review comment naming a **stale** (non-current) SHA — OR no SHA at all (empty extraction) — does NOT satisfy the gate; SHA tokens are explicitly extracted (`[0-9a-fA-F]{7,40}`, ≥7 chars, non-empty) and prefix-matched against `HEAD_OID`, so an empty token never spuriously passes.
- **R3:** A clean-review comment from a **non-automated** login (not `[bot]`-suffixed, not in `land.automatedReviewers`) is ignored.
- **R4:** The comment path never bypasses an unresolved thread or red CI — those gates are evaluated unchanged.
- **R5:** `land.cleanReviewCommentPattern` config contract: `get_default_config()` seeds a structured built-in ERE; `null`/missing → workflow falls back to that built-in; **explicit empty string `""` → comment scan disabled** (pure reviews-API behavior, the real off-switch); other value → used. The scan runs only when `REVIEW_SIGNAL == silence`, and only ever SETS `AUTO_REVIEW_CURRENT=1` (never resets a reviews-API result).
- **R7:** A comment-driven satisfaction is observable: `AUTO_REVIEW_SOURCE=comment` + `AUTO_REVIEW_EVIDENCE` (author + matched SHA prefix) surface in `--dry-run` and the verdict report.
- **R6:** Docs updated in BOTH surfaces: repo `plugins/flow-next/skills/flow-next-land/workflow.md §2.6` (+ SKILL.md config table if present) AND flow-next.dev `autonomous/land.mdx` "The merge gate, precisely" — both describe that a bot can signal clean via a SHA-named comment and how land treats it. Codex mirror regenerated if the canonical land skill prose changes.

## Boundaries

- **Scope is the `silence`-signal comment-detection only** — no change to `approve`, no change to `<login>` (its comment-evidence is an explicit follow-up), the patience window, CI tri-state, resolve-pr dispatch, the merge mechanics, or the post-merge tail.
- **Not a general NLP pass on comments** — a single configurable pattern + the automated-reviewer allowlist + a head-SHA presence check. No sentiment analysis, no "looks approving" inference.
- **No new auto-merge authority** — land's merge gate (CI + threads + window) is untouched; this only restores detection of a review land already intends to honor.
- **Don't try to make Codex file formal reviews** — that's the bot's behavior, out of our control; land adapts to it.

## Decision Context

The `silence` signal was built precisely for "bot reviewers that comment but never file formal APPROVEs" (land.mdx), but the implementation only reads the formal reviews API — which a no-findings Codex pass never writes to. So the feature's stated intent and its detection disagree exactly on the bot it was built for. PR #176 is the proof: every real gate was green and the reviewer had demonstrably re-reviewed the head clean, yet land-as-implemented could not see it.

Keeping the match **configurable + allowlist-gated + head-SHA-anchored** preserves land's conservative posture (never merge unreviewed, never honor a stale review) while closing the blind spot. The alternative — telling users to switch to `reviewSignal=approve` — doesn't work for Codex (it never APPROVEs), so the comment path is the right fix for the default bot-review flow.

## Requirement coverage

| R-ID | Task |
|---|---|
| R1, R2, R3, R4, R5, R7 | fn-65.1 — `land.cleanReviewCommentPattern` config default + test_land_config + workflow.md §2.6 comment-scan detection + SKILL.md prose + Codex mirror regen |
| R6 (remainder) | fn-65.2 — flowctl.md table + flow-next.dev land.mdx (signal table/prose/config) + CHANGELOG + version bump + flow-next.dev changelog/site.ts/package.json |
