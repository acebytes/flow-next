## Conversation Evidence

> user: "add a second loop that does the resolving, waiting for comments, fixing bla bla and eventually merging and releasing?"
> user: "fully autonomous, its a sep skill right, so no risk, ppl dont have to run it. ci green, or fix ci, wait for automated reviewers, max time 30mins, then fix if valid etc, that in a loop until no new reviews come in"
> user: "whatever is defined in the actual project, the agent will pick this up automatically, so we would put something like follow this project's release instructions if they exist, otherwise just merge"
> user: "perhaps we add a new merge-pr skill for dealing with conflicts if necessary"

## Goal & Context
<!-- Source: 65% [user] / 25% [paraphrase] / 10% [inferred] -->

A cadence-driven (`/loop`-shaped) babysit loop — **opt-in and fully autonomous, a separate skill** so projects that don't run it carry zero risk. For each open PR the build-loop authored, it keeps CI green, waits for automated reviewers, resolves valid feedback, and once converged merges and (if the project defines a release process) releases. Where the build-loop is `/goal`-shaped (drains then stops), this is `/loop`-shaped — wakes on a cadence, acts on PRs, sleeps.

## Architecture & Data Models
<!-- Source: 60% [user] / 30% [paraphrase] / 10% [inferred] -->

Per cadence tick, for each open PR the loop authored:

1. **CI** — `gh pr checks`: red → diagnose + fix + push (report `FIXING_CI`); green → continue.
2. **Reviews** — wait for automated-reviewer threads within a ~30-minute patience window; none yet & PR younger than window → report `AWAITING_REVIEW` (next tick re-checks).
3. **Resolve** — new valid threads → `/flow-next:resolve-pr` (fix-verify-reply-resolve).
4. **Converge** — repeat 2–3 until a tick finds no new reviews.
5. **Merge** — CI green + an approving automated review + threads addressed → flip the build-loop's draft PR to ready (`gh pr ready` — pilot's PRs are born draft) and `gh pr merge`, autonomously; then `flowctl spec close` on the spec — land is the pipeline terminus, and closing prevents the build-loop from re-selecting a merged spec.
6. **Release** — discover + follow the project's own release instructions (RELEASING.md / docs / scripts) if present; otherwise stop at merge. A config toggle (e.g. `land.release`) can disable the release step independently of the rest of the loop.

`resolve-pr` gets a light autonomous touch following the fn-59.2 convention — a `mode:autonomous` arg token as the primary signal (env vars do not survive across tool calls; capture's `mode:autofix` parse is the precedent) with `FLOW_AUTONOMOUS=1` env honored as the secondary, question-suppression branches only, never Ralph paths: under autonomy it skips its needs-human bucket and reports `NEEDS_HUMAN` instead of blocking on a question.

### Resolved at planning (research + gap analysis, 2026-06-11)

Binding decisions from grep research (gh 2.93.0 verified surfaces), practice/docs scouts, and gap analysis. Implementation tasks follow these; deviations need a re-plan.

- **Verdict contract (R10).** Per-PR verdicts `MERGED | RELEASED | FIXING_CI | AWAITING_REVIEW | RESOLVING | BLOCKED | NEEDS_HUMAN` echoed as evidence lines; every tick ends with exactly ONE terminal line, the LAST line of output: `LAND_VERDICT=<verdict|NO_WORK> prs=<n> pr=<deciding-pr-url|-> reason="<one line>"`. Tick-level verdict = worst severity across PRs by priority `NEEDS_HUMAN > BLOCKED > FIXING_CI > RESOLVING > AWAITING_REVIEW > RELEASED > MERGED`; `NO_WORK` when discovery finds zero authored PRs. Mirrors `PILOT_VERDICT` (transcript-blind drivers).
- **CI is tri-state, not binary (R2).** Gate on ALL checks, not `--required` (required-only ignores failing optional CI and deadlocks on repos with no required checks): `gh pr checks --json bucket` — green iff every check's bucket ∈ {pass, skipping}; any `fail` → CI-fix path; any `pending` (or exit 8) → wait, NOT a fix trigger, NOT green. An EMPTY check list right after a push means checks have not registered yet — treat as pending within the patience window, `NEEDS_HUMAN` beyond it. Never treat empty as success.
- **CI-fix contract (R2).** Per fix attempt: checkout the PR head branch → inspect the failing check (`gh pr checks --json name,bucket,link`; when the check maps to a GitHub Actions run, derive the run id from the link and use `gh run view <id> --log-failed`; external/status checks with no Actions run → use the check name/link as evidence and attempt only locally-discoverable matching validation; when logs are unavailable AND no local validation exists, report `NEEDS_HUMAN` — never pretend CI was diagnosed) → judge relatedness: failure unrelated to the PR diff (infra flake) → ONE `gh run rerun <id>` before any strike; related → scope edits to the failing surface only, run the repo's matching local validation when discoverable, stage ONLY the files edited (never `git add -A` here), commit `fix(ci): <what>`, push to the PR head branch. Increment the ledger strike BEFORE pushing (a crashed push still consumed an attempt). Report `FIXING_CI`; the push restarts the patience window.
- **Merge mechanism (R6).** Explicit `gh pr merge --squash --delete-branch --match-head-commit <head-sha>` AFTER all gates pass in-tick. NEVER `gh pr merge --auto` (on a repo with no branch protection `--auto` merges instantly — land itself is the gate, so server-side gating adds nothing). Head-SHA mismatch at merge = state moved → re-tick (`RESOLVING`), not `BLOCKED`. AFTER a successful merge, move the worktree onto the merged base before any tail step: `git checkout <base>` + `git pull --ff-only` + verify the squash commit is present (`git log --oneline -1` references the PR) — `spec close`, the tracker touchpoint, and release-follow all run from that clean base checkout, never from the (deleted) PR branch or a stale original branch. Re-poll `mergeStateStatus` on `UNKNOWN` (async recompute); `DIRTY` → conflict path (R7): mechanical rebase only — checkout PR head, `git fetch origin <base>`, `git rebase origin/<base>`; ANY conflict hunk → `git rebase --abort` → `BLOCKED` (v1 never hand-resolves conflicts autonomously); clean rebase → push `--force-with-lease` (restarts the patience window) → re-gate. `BEHIND` → same mechanical update then re-gate.
- **Review convergence (R3/R6) — `land.reviewSignal`.** Review bots (e.g. chatgpt-codex-connector) post COMMENTED reviews and never APPROVE; `reviewDecision` will not reflect them. Config `land.reviewSignal` ∈ `silence` (default) | `approve` | `<github-login>`: `silence` = ≥1 review by an AUTOMATED reviewer present AND zero unresolved threads AND no new threads within the patience window — an automated reviewer is a review author whose login ends in `[bot]`, or any login listed in `land.automatedReviewers` (csv, default empty = `[bot]`-suffix rule only); `approve` = formal `reviewDecision == APPROVED`; `<login>` = that reviewer's latest review is APPROVED/clean. `reviewDecision == ""` (no review policy) is not a block. No automated review ever + no signal configured → never merge unreviewed → `NEEDS_HUMAN` (existing edge case).
- **Patience window anchor.** Last push timestamp via `gh pr view --json commits` last commit `committedDate` falling back to `gh api .../pulls/N` `head` + commit `pushedDate` — anchored to LAST PUSH, never `createdAt`; a land-authored CI-fix push restarts the window.
- **Stale-approval loop detection (R3).** If a land-authored push flips `reviewDecision` APPROVED → REVIEW_REQUIRED while threads are already resolved (repo has dismiss-stale-approvals), do NOT re-loop: `NEEDS_HUMAN` + durable label.
- **CI-fix budget (R14).** Default 3 attempts per PR (`land.ciFixBudget`), persisted in the land ledger at `$(git rev-parse --git-common-dir)/flow-next/land-strikes.json` (same atomic jq+mv pattern as pilot's ledger, never committable); exhaustion → durable label `flow-next:needs-human` on the PR + `NEEDS_HUMAN`, skipped on later ticks while the label is present.
- **Config surface (minimal).** `land.release` (default `true`; release step also no-ops when no release docs/scripts are discovered), `land.patienceMinutes` (default 30), `land.reviewSignal` (default `silence`), `land.automatedReviewers` (default `""` — csv allowlist supplementing the `[bot]`-suffix rule), `land.ciFixBudget` (default 3). Seeded in flowctl config defaults (flowctl.py ~:1129, mirroring `work.delegate*`) so `config get` returns values, not null.
- **Discovery (Edge Cases).** Open PRs whose head branch equals an open spec's `branch_name` (contractual since fn-59.2; `gh pr list --head <branch> --state all --json url,state,number` per-spec, filter `state == "OPEN"` — bare `gh pr view` returns rc 0 for CLOSED/MERGED, fn-42 finding). Authorship requires BOTH signals before any mutation: branch match AND the make-pr breadcrumb/spec id in the PR body (`Generated by /flow-next:make-pr from <spec-id>`) — branch-only matches are reported `NEEDS_HUMAN`, never acted on (a hand-opened PR on a spec branch must not be auto-merged). Two open PRs matching one branch → `NEEDS_HUMAN`. Land only babysits PRs whose authoring spec has ALL tasks done (pilot still owns in-flight specs — this is the pilot-concurrency interlock).
- **Re-entry idempotency (R13).** A tick that finds a MERGED PR whose spec is still open resumes the tail: `spec close` → tracker flip → release — never a second merge, never an error. `flowctl spec close` hard-requires all tasks done; stray non-done tasks at close time → `NEEDS_HUMAN` (report, never force).
- **Branch hygiene.** Pilot-style guards: refuse tick on dirty non-`.flow/` tree; CI-fix checkouts happen per-PR and restore the original branch + assert clean before the next PR and before tick end. Ralph-nesting hard guard verbatim from pilot (`FLOW_RALPH`/`REVIEW_RECEIPT_PATH` → refuse).
- **resolve-pr carve-out (R5).** `flow-next-resolve-pr/SKILL.md` Forbidden line "Auto-invocation by Ralph or any other skill — user-triggered only" gains the confined exception: land may dispatch resolve-pr with `mode:autonomous`; workflow.md Phase 0 parses + strips the token (make-pr's bash parse shape), and the Phase-10 `AskUserQuestion` needs-human surface reports `NEEDS_HUMAN` lines instead of blocking. Under autonomy resolve-pr ALSO ends with one machine-readable terminal line — `RESOLVE_PR_VERDICT=<RESOLVED|PENDING|NEEDS_HUMAN> threads=<n> fixed=<n> needs_human=<n>` — which is what land gates on (RESOLVED → re-check convergence; PENDING → next tick; NEEDS_HUMAN → label + report). Bounded 2 fix-verify cycles unchanged; escalation under autonomy = `NEEDS_HUMAN`.
- **Tracker touchpoint (R13).** Opt-in leaf `tracker.perEvent.land.merged` (default off, zero overhead for non-tracker repos): after merge+close, dispatch the tracker-sync skill (status → terminal + release/verdict comment), receipts event-tagged `land.merged` per fn-57. No new transport code.
- **Release follow (R8) — bounded.** Discovery order: `docs/RELEASING.md` → `RELEASING.md` → `agent_docs/releasing.md` → release docs referenced from CLAUDE.md/AGENTS.md → none. Eligibility bounds: deterministic, non-interactive commands from the discovered docs ONLY (no invented steps, no prompts, no secrets handling); clean non-`.flow/` tree required before and asserted after; idempotency — probe for an existing tag/release for the target version BEFORE acting, so a re-entry tick never re-tags (resume past completed steps). Release-step failure AFTER a successful merge → `NEEDS_HUMAN` + durable label (merge is never retried; later ticks never blindly re-run the failed step). No generic release engine.
- **gh version note.** Gates verified against gh 2.93.0 (`--required`/`bucket`/exit-8, `--match-head-commit`, `mergeStateStatus`). Re-verify flag shape on major gh bumps; document min-version expectation in the skill.
- **Sequencing.** fn-54 (prompt optimization) must baseline resolve-pr AFTER fn-60's autonomous-mode patch lands — recorded via `spec add-dep fn-54 fn-60`.

## API Contracts
<!-- scope: technical -->

- **Invocation** `/flow-next:land`, opt-in. [user]
- **Per-PR verdict** `MERGED | RELEASED | FIXING_CI | AWAITING_REVIEW | RESOLVING | BLOCKED | NEEDS_HUMAN`. [paraphrase]
- **Tooling** — `gh pr checks` / `pr view` (review threads + decision) / `pr merge`; honors `FLOW_AUTONOMOUS`. [paraphrase]
- **Merge policy** — CI green + approving automated review + threads addressed (the gate the user specified). [user]
- **Terminal line** — `LAND_VERDICT=<verdict|NO_WORK> prs=<n> pr=<url|-> reason="<one line>"` as the last line of every tick (see Resolved at planning). [inferred]

## Edge Cases & Constraints
<!-- scope: technical -->

- ~30-minute patience window for the first automated review; convergence = no new threads since the last resolve. [user]
- Never nests under Ralph: refuse to run when `FLOW_RALPH` / `REVIEW_RECEIPT_PATH` is set (hard-error + terminal `NEEDS_HUMAN`-class verdict, same guard as pilot — alternative drivers, never nested). [inferred]
- Operates only on PRs the build-loop authored — discovered as open PRs whose head branch matches a flow spec's `branch_name` (make-pr bodies embed the spec id as a secondary signal); never arbitrary PRs. The primary signal is contractual since fn-59.2: under autonomy, work names its new branch exactly the spec's `branch_name`. [inferred]
- make-pr §4.6b (shipped 1.11.0) guarantees the PR body carries the tracker ref post-create — the body-embedded secondary discovery signal is now reliable; `branch_name` match stays primary. [inferred]
- The post-merge close (R13) flips the linked issue to its terminal state + posts the release/verdict comment + emits an event-tagged receipt through the fn-57 layer — the exact flow exercised manually at the 1.11.0 release. [inferred]
- CI-fix attempts are bounded per PR; on exhaustion the PR is durably marked (e.g. a `flow-next:needs-human` label — survives sessions, visible on GitHub), reported `NEEDS_HUMAN`, and skipped on later ticks. [inferred]
- The patience window anchors to the last push, not PR creation — a CI-fix push invalidates prior reviews and restarts the wait. [inferred]
- "Approving automated review" must be a detectable, per-project signal (a bot's APPROVE review or a configured reviewer's clean verdict) — reviewer bots differ in whether they file formal approvals. [inferred]
- If no approving automated review ever arrives and no reviewer is configured, it must **not** merge unreviewed — report `NEEDS_HUMAN`. [inferred]
- Merge conflicts: attempt rebase / resolve; if unresolvable, report `BLOCKED`. [user]
- `resolve-pr` is bounded at its existing 2 fix-verify cycles, then escalates → `NEEDS_HUMAN`. [paraphrase]
- Release is project-specific — no generic primitive; follow discovered instructions or stop. [user]

## Acceptance Criteria
<!-- scope: both -->

- **R1:** A new opt-in skill `/flow-next:land` babysits the open PRs the build-loop authored, on a `/loop` cadence, fully autonomously. [user]
- **R2:** For a PR with red CI it diagnoses + fixes + pushes and reports `FIXING_CI`; with green CI it proceeds — using the tri-state semantics (pass / fail / pending-or-unregistered) from Resolved at planning. [user]
- **R3:** It waits for automated-reviewer feedback within a ~30-minute patience window anchored to the last push, reporting `AWAITING_REVIEW` until reviews arrive or the window elapses; a stale-approval-dismissal loop is detected and reported `NEEDS_HUMAN`, never re-looped. [user]
- **R4:** New valid review threads are resolved via `/flow-next:resolve-pr`; it loops resolve → re-check until no new reviews arrive (convergence). [user]
- **R5:** `resolve-pr` runs autonomously under the fn-59.2 signal convention (`mode:autonomous` arg token primary, `FLOW_AUTONOMOUS=1` env secondary; question-suppression branches only, never Ralph/receipt paths) — its needs-human cases report `NEEDS_HUMAN` instead of blocking on a question, it ends with the machine-readable `RESOLVE_PR_VERDICT=…` terminal line land gates on, and its Forbidden auto-invocation line carries the confined land exception. [paraphrase]
- **R6:** Once CI is green, threads are addressed, and the configured review signal (`land.reviewSignal`: silence-convergence default, formal approve, or named reviewer) is satisfied, it merges via explicit `gh pr merge --squash --match-head-commit` — autonomously, never `--auto`. [user]
- **R7:** Merge conflicts are handled (rebase / resolve attempt) or reported `BLOCKED`. [user]
- **R8:** After merge it discovers and follows the project's own release instructions if present; otherwise it stops at merge. [user]
- **R9:** It is opt-in and isolated — a separate skill; projects that don't run it are unaffected, and it touches only PRs it authored (or a configured filter). [user]
- **R10:** Per-PR verdicts are `MERGED | RELEASED | FIXING_CI | AWAITING_REVIEW | RESOLVING | BLOCKED | NEEDS_HUMAN`, and every tick ends with the single terminal `LAND_VERDICT=…` line defined in Resolved at planning (worst-severity priority rule; `NO_WORK` when no authored PRs), as the LAST line of the response. [paraphrase]
- **R11:** Docs + flow-next.dev updated — new skill page, both navbars, changelog, command reference, version bump; the auto-merge override is documented (CLAUDE.md exception note included). [user]
- **R12:** Before merging, it flips the build-loop's draft PR to ready (`gh pr ready`, idempotent — non-draft PRs skip) once the merge gate is satisfied. [inferred]
- **R13:** After a successful merge it closes the spec (`flowctl spec close`), flips the linked tracker issue via the opt-in `land.merged` touchpoint, and a tick that re-enters on a merged-but-unclosed spec resumes close → tracker → release idempotently. [inferred]
- **R14:** CI-fix attempts are bounded per PR (`land.ciFixBudget`, default 3, ledger under the git common dir); on exhaustion the PR is durably labeled `flow-next:needs-human`, reported `NEEDS_HUMAN`, and skipped on subsequent ticks. [inferred]
- **R15:** The `land.*` config surface (`release`, `patienceMinutes`, `reviewSignal`, `automatedReviewers`, `ciFixBudget`) ships with seeded flowctl defaults so `config get` returns values on fresh repos. [inferred]
- **R16:** Branch hygiene: dirty-tree refusal at tick start, per-PR checkout restored to the original branch + clean-tree assertion between PRs and at tick end. [inferred]
- **R17:** `/flow-next:land --dry-run` reports discovery + per-PR gate classification (CI tri-state read, review-signal state, would-be action) and the aggregated terminal line WITHOUT any mutation — no checkout, no push, no label, no merge, no ledger write. [inferred]

## Boundaries
<!-- scope: business -->

- Auto-merge here **intentionally overrides** the standing "no `gh pr merge` from skills" rule — confined to this one opt-in skill; the build-loop and every other skill keep the no-auto-merge rule. [user]
- **No generic release engine** — the loop follows the project's own release process or stops; it never invents versioning / publish steps. [user]
- It does **not** author PRs (the build-loop does) — it only babysits existing ones. [paraphrase]
- Choosing / authoring specs, planning, implementing — out of scope (build-loop). [paraphrase]
- Human PR review is not required (automated reviewers gate merge); but with no automated reviewer configured it must not merge unreviewed. [user]
- Parallel-PR throughput tuning (max PRs per tick) is out of scope for v1 — process all discovered PRs serially. [inferred]

## Decision Context
<!-- scope: both -->

`/loop` cadence vs `/goal`: babysitting waits on external events (CI, reviewers) over hours, so it's cadence-driven, not drain-to-completion — hence a separate skill from the build-loop. Fully autonomous + opt-in + isolated is the user's framing: a separate skill means no risk to non-users, and that isolation is what licenses the auto-merge override confined here. Release can't generalize — every project releases differently and flow-next has no generic release primitive — so "follow the project's instructions or stop" is the only honest contract. Explicit merge over `gh pr merge --auto` because land itself is the gate: `--auto` delegates gating to branch protection, which insta-merges on unprotected repos. Silence-convergence as the default review signal because the dominant bot reviewers (Codex et al.) never file formal APPROVEs. Depends on fn-59 for the PR-authoring convention it babysits. [strategy:Ralph autonomous mode]

## Strategy Alignment
<!-- STRATEGY.md populated 2026-05-16 -->

- Extends the **Ralph autonomous mode** track to the PR → merge → release tail, gated on CI + an approving automated review (quality-first). [strategy:Ralph autonomous mode]
- No conflict with *"Not working on: built-in CI runners"* — it reads the user's CI (`gh pr checks`) and fixes code; it provides no CI runner. [strategy]
- No conflict with *"Not working on: SaaS"* — merge / release via `gh` + the project's own scripts, all in-repo. [strategy]
- **Cross-platform parity** — sync-codex mirror for the Codex variant. [strategy:Cross-platform parity]

## Quick commands

```bash
# Skill + mirror validation (after fn-60.1 / fn-60.3)
bash -n scripts/sync-codex.sh && ./scripts/sync-codex.sh
# Config defaults seeded (after fn-60.2)
.flow/bin/flowctl config get land.reviewSignal --json   # → "silence", not null
# Full test suite
for f in plugins/flow-next/tests/test_*.py; do python3 "$f" || echo "FAIL: $f"; done
```

## Early proof point

Task fn-60.1 validates the core approach: that gh state (all-checks `gh pr checks --json bucket` buckets, `reviewDecision`, unresolved threads, `mergeStateStatus`) is sufficient to drive the full gate decision tree for one real PR tick. If the silence-convergence signal proves undetectable or gh state is too racy to gate on, re-evaluate the merge-gate design (e.g. require a named reviewer) before fn-60.2/.3.

## Requirement coverage

| Req | Description | Task(s) | Gap justification |
|-----|-------------|---------|-------------------|
| R1 | /flow-next:land skill, cadence babysitter | fn-60.1 | — |
| R2 | CI tri-state fix loop | fn-60.1 | — |
| R3 | Patience window + stale-approval detection | fn-60.1 | — |
| R4 | resolve-pr dispatch until convergence | fn-60.1, fn-60.2 | — |
| R5 | resolve-pr autonomous mode + carve-out | fn-60.2 | — |
| R6 | Gated explicit merge | fn-60.1 | — |
| R7 | Conflict handling | fn-60.1 | — |
| R8 | Release-follow | fn-60.1 | — |
| R9 | Opt-in + isolated | fn-60.1 | — |
| R10 | Verdict grammar + terminal line | fn-60.1 | — |
| R11 | Docs + site + version | fn-60.3 | — |
| R12 | Draft → ready flip | fn-60.1 | — |
| R13 | Spec close + tracker + re-entry | fn-60.1 | — |
| R14 | Bounded CI fixes + durable label | fn-60.1 | — |
| R15 | land.* config seeded defaults | fn-60.2 | — |
| R16 | Branch hygiene | fn-60.1 | — |
| R17 | --dry-run classification mode | fn-60.1 | — |
