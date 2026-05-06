# /flow-next:diagnose — disciplined six-phase debugging skill

## Conversation Evidence

> user: "capture an epic for diagnose"
> user (follow-up, paraphrased): the skill must read as flow-next-native — composing against existing memory / glossary / decisions / doc-aware infrastructure, not a generic methodology bolted on.
> user (later turn): not convinced this should be in the Ralph loop — pushed back on the original Ralph-safe stance; diagnose's user-checkpoint at Phase 3 is load-bearing, and Ralph already has a bug-fix path through review-iteration + auto-capture on `NEEDS_WORK → SHIP`. Ralph-block diagnose like capture / prospect / strategy.

The skill described below is flow-next-native. Each phase composes against existing flow-next infrastructure (memory tracks, glossary, decisions, doc-aware skill conventions, Ralph-block pattern matching capture / prospect / strategy, canonical-Claude + sync-codex.sh cross-platform rule). The methodology itself — feedback-loop discipline, falsifiable hypotheses, tagged debug logs, correct-seam regression-test rule, post-mortem capture — is well-trodden in the senior-engineer post-mortem literature; the load-bearing flow-next contribution is making it a *phase contract* the host agent can't skip by accident, plus the integration surface with `flowctl memory add` + `GLOSSARY.md` + `knowledge/decisions/`.

## Goal & Context

<!-- Source-tag breakdown: ~20% [user] / ~50% [paraphrase] / ~30% [inferred] -->

flow-next currently has no first-class debugging skill. When a user (or a Ralph worker mid-task) hits a hard bug or a performance regression, the host agent reaches for ad-hoc reasoning: read code, guess, edit, hope. The most under-applied debugging principle — *build a fast deterministic feedback loop before reasoning about the bug* — is the one most often skipped.

`/flow-next:diagnose` packages senior-engineer debugging discipline into a slash-command skill that the host agent reads + executes. The phases enforce: refuse to leave Phase 1 without a deterministic loop; generate ranked falsifiable hypotheses before testing any; instrument with tagged logs that vacate cleanly; only write regression tests when a correct seam exists; close the loop with a bug-track memory entry that the next debugger can search.

The skill is **flow-next-native** — not generic methodology bolted on, but one that integrates against existing flow-next infrastructure:

- `GLOSSARY.md` for code-area mental model when investigating an unfamiliar module.
- `knowledge/decisions/` for ADR-equivalent prior architectural choices in the area being touched (cited *during* investigation, not just at the end).
- `flowctl memory add --track bug --category <c>` for post-mortem capture, mirroring Ralph's existing `NEEDS_WORK → SHIP` auto-capture pattern.
- `flowctl memory add --track knowledge --category architecture-patterns` when Phase 5's correct-seam check surfaces a structural finding. flow-next deliberately does NOT spawn a separate architecture-improvement skill — the memory primitive already exists, and skill proliferation costs more than it pays. Future epics can build a consumer skill if accumulated entries justify it.
- Ralph-block guard matching `/flow-next:capture`, `/flow-next:prospect`, `/flow-next:strategy` — `FLOW_RALPH=1` or `REVIEW_RECEIPT_PATH` set → exit 2 with stderr message. Ralph's existing bug-fix loop (worker → review backend → `NEEDS_WORK` re-spawn → auto-capture on `SHIP`) is the autonomous-mode debugging path; diagnose is human-only because Phase 3's user-checkpoint is load-bearing.
- Canonical-Claude-native + `sync-codex.sh` rewrite pattern for cross-platform parity with Codex / Droid.

This epic is a **minor bump** — one new skill, one new slash command, no flowctl schema changes (memory subsystem already supports the categories in use).

## Architecture & Data Models

<!-- Source-tag breakdown: ~50% [paraphrase] / ~50% [inferred] -->

```
User invokes /flow-next:diagnose [bug context]
         │
         ▼
    Skill workflow runs in host agent (no subprocess dispatch)
         │
         ├─ Phase 0: Pre-flight (Ralph-block guard; exit 2 if FLOW_RALPH=1
         │           or REVIEW_RECEIPT_PATH set)
         │
         ├─ Phase 1: Build a feedback loop (THE skill)
         │     10 ranked techniques: failing test → curl → CLI fixture →
         │     headless browser → trace replay → throwaway harness →
         │     fuzz/property → bisection → differential → HITL bash
         │     Refuse to advance without one. Iterate on the loop itself
         │     (faster / sharper / more deterministic).
         │
         ├─ Phase 2: Reproduce
         │     Run the loop. Confirm the failure mode matches the
         │     reported bug (not a nearby bug that happens to be visible).
         │
         ├─ Phase 3: Hypothesise
         │     Generate 3-5 ranked falsifiable hypotheses.
         │     Show them to the user via AskUserQuestion BEFORE testing —
         │     cheap checkpoint, big time saver. Don't block if user is
         │     AFK; proceed with agent ranking.
         │
         ├─ Phase 4: Instrument
         │     Each probe maps to a Phase 3 prediction.
         │     Tag every debug log with a unique prefix [DEBUG-<hash>] so
         │     cleanup is a single grep.
         │     Perf branch: measure-first (baseline + bisect), not "log
         │     everything and grep".
         │
         ├─ Phase 5: Fix + regression test
         │     Write the regression test BEFORE the fix — but only if a
         │     correct seam exists.
         │     If no correct seam exists, that itself is the finding.
         │     Note it for Phase 6's architecture-pattern write.
         │
         └─ Phase 6: Cleanup + post-mortem
               Re-run Phase 1 loop, kill [DEBUG-*] logs,
               state the winning hypothesis in the commit / PR message,
               write a bug-track memory entry,
               write an architecture-pattern entry if Phase 5 surfaced
                 a correct-seam-absent finding.
```

**Doc-aware integration (flow-next-specific):** Phases 1 and 4 cite `GLOSSARY.md` for code-area mental model and `knowledge/decisions/` for prior architectural choices in the area being touched. Default when investigating unfamiliar code; not a hard requirement at every phase. This matches how `/flow-next:interview` and `/flow-next:plan` consume the same docs read-only.

**Bundled resource:** A `hitl-loop.template.sh` bash script lives in the skill directory with two helpers — `step "<instruction>"` and `capture VAR "<question>"` — driving a structured human-in-the-loop reproduction when only a human can click. Captured values are emitted as `KEY=VALUE` for the agent to parse from the script's stdout.

**Memory write contract (Phase 6) — flow-next-specific:**

- Always: `flowctl memory add --track bug --category <c> --title "..." --module <m> --tags <auto>` with the winning hypothesis + minimal repro in the body. Mirrors the auto-capture path Ralph already takes on `NEEDS_WORK → SHIP`.
- When Phase 5 found no correct seam: additional `flowctl memory add --track knowledge --category architecture-patterns --title "..." --module <m>` capturing the structural finding. Future debuggers in the same module find it via `memory-scout`.

## API Contracts

<!-- Source-tag breakdown: 100% [inferred] — no user-stated API requirements -->

Slash command surface:

```
/flow-next:diagnose [<short bug description>]
```

Interactive-only — no autofix variant. Diagnose is **Ralph-blocked** (matches `/flow-next:capture`, `/flow-next:prospect`, `/flow-next:strategy`): when `FLOW_RALPH=1` or `REVIEW_RECEIPT_PATH` is set, the skill exits 2 with stderr `Error: /flow-next:diagnose requires a user at the terminal (Phase 3 user-checkpoint is load-bearing); not compatible with Ralph mode. Ralph's existing review-iteration loop is the autonomous bug-fix path.`

The user-checkpoint at Phase 3 (rank 3-5 falsifiable hypotheses → user picks before testing) is the load-bearing discipline. Without it, the methodology degrades to "agent guesses harder", which is no better than the ad-hoc reasoning diagnose exists to replace.

flowctl plumbing already exists; no new flowctl subcommands, no new memory categories, no schema changes.

## Edge Cases & Constraints

<!-- Source-tag breakdown: ~40% [paraphrase] / ~60% [inferred] -->

- **Non-deterministic bugs.** Phase 1 prose addresses these: the goal is not a clean repro but a *higher* reproduction rate. Loop the trigger, parallelise, add stress, narrow timing windows. A 50%-flake bug is debuggable; 1% is not.
- **Genuinely no loop possible.** Phase 1 prose says: stop and say so explicitly, list what was tried, ask the user for environment access / captured artifact / temporary instrumentation. Do NOT proceed to hypothesise without a loop.
- **No correct seam for regression test.** Phase 5 says: that itself is the finding. Document the architectural gap, do NOT write a fake regression test that gives false confidence. Phase 6 captures the finding as a `knowledge/architecture-patterns/` memory entry.
- **User AFK at Phase 3 checkpoint.** Don't block; proceed with agent ranking. Note that the checkpoint was skipped so a human re-reading the post-mortem can see whether the agent's ranking was peer-reviewed.
- **Ralph mode invocation.** Diagnose IS Ralph-blocked. `FLOW_RALPH=1` or `REVIEW_RECEIPT_PATH` set → exit 2. Ralph's existing bug-fix loop (worker → review backend → `NEEDS_WORK` re-spawn → `bug` track auto-capture on `SHIP`) is the autonomous-mode debugging path. Mid-task workers do NOT invoke diagnose; they iterate via the review-backend feedback loop. Diagnose is for humans because Phase 3's user-checkpoint is load-bearing.
- **Cross-platform parity.** Canonical files use Claude-native tool names (`AskUserQuestion`, `Task`); `scripts/sync-codex.sh` rewrites them in the Codex mirror per the repo cross-platform pattern.

## Acceptance Criteria

- **R1:** Skill at `plugins/flow-next/skills/flow-next-diagnose/SKILL.md` (with `workflow.md` / `phases.md` / `references.md` as needed) implements the six-phase methodology — Phase 1 (build a feedback loop) → Phase 2 (reproduce) → Phase 3 (hypothesise) → Phase 4 (instrument) → Phase 5 (fix + regression test) → Phase 6 (cleanup + post-mortem). [paraphrase]
- **R2:** Phase 1 enumerates the ranked feedback-loop techniques (failing test, curl/HTTP script, CLI fixture diff, headless browser, captured-trace replay, throwaway harness, fuzz/property loop, bisection harness, differential loop, HITL bash) and refuses to advance to Phase 2 without a deterministic loop. [paraphrase]
- **R3:** Phase 3 prescribes 3-5 ranked falsifiable hypotheses before any are tested, surfaced via `AskUserQuestion`, proceeding with agent ranking only when the user is AFK (interactive degradation, not a Ralph path — diagnose is Ralph-blocked). [paraphrase]
- **R4:** Phase 4 prescribes the `[DEBUG-<hash>]` tagged-log convention so cleanup is one grep, and separates the perf branch (measure-first, baseline + bisect) from the correctness branch. [paraphrase]
- **R5:** Phase 5 enforces the correct-seam rule — write the regression test before the fix only when a correct seam exists; if no correct seam exists, the absence itself is the finding (architectural). [paraphrase]
- **R6:** Phase 6 cleanup checklist removes `[DEBUG-*]` instrumentation, deletes throwaway prototypes, states the winning hypothesis in the commit / PR message, and writes a bug-track memory entry via `flowctl memory add --track bug --category <c>` (matching Ralph's existing auto-capture pattern on `NEEDS_WORK → SHIP`). [paraphrase]
- **R7:** When Phase 5 surfaces a correct-seam-absent finding, Phase 6 writes the finding as a `knowledge/architecture-patterns/` memory entry via `flowctl memory add --track knowledge --category architecture-patterns`. No separate skill handoff. [paraphrase]
- **R8:** Skill prose cites `GLOSSARY.md` for code-area mental model and `knowledge/decisions/` for prior architectural context when investigating the bug area, matching how `/flow-next:interview` and `/flow-next:plan` consume the same docs read-only. [paraphrase]
- **R9:** Skill bundles `hitl-loop.template.sh` with `step "..."` and `capture VAR "..."` helpers, emitting captured values as `KEY=VALUE` for agent parsing. [paraphrase]
- **R10:** Slash command at `plugins/flow-next/commands/flow-next/diagnose.md` mirrors the existing `capture.md` / `prospect.md` / `strategy.md` shape (Ralph-blocked skills). Skill SKILL.md begins with the Ralph-block guard (exit 2 on `FLOW_RALPH=1` or `REVIEW_RECEIPT_PATH`) before any other phase. No `mode:autofix` token — diagnose is interactive-only. [paraphrase]
- **R11:** Skill ships fully cross-platform per the canonical-Claude-native + sync-codex.sh rewrite pattern: canonical files use `AskUserQuestion` / `Task`; `scripts/sync-codex.sh` adds the `generate_openai_yaml` call in the workflow section; `REQUIRED_OPENAI_YAML_SKILLS` includes `flow-next-diagnose`. [paraphrase]
- **R12:** Documentation updates land alongside the skill: `CHANGELOG.md` entry under the version block; `plugins/flow-next/README.md` skills/commands table + count incremented; `CLAUDE.md` commands list updated. [paraphrase]

## Boundaries

- **Out of scope:** A separate `/flow-next:improve-codebase-architecture` skill. Phase 5/6's architectural-finding capture lands in memory only; a future epic may build a dedicated consumer skill if accumulated entries justify it.
- **Out of scope:** Folding diagnose into `/flow-next:work`. Diagnose is a standalone human-invoked slash command. Ralph workers do NOT invoke diagnose mid-task; they rely on the existing review-iteration loop + `bug` track auto-capture. A future small extension to `/flow-next:work` skill prose may reference Phase 1 (build a feedback loop) discipline as a recommended sub-pattern; that extension is a separate epic.
- **Out of scope:** New flowctl subcommands. Phase 6 uses existing `flowctl memory add` plumbing; no schema changes, no new memory categories.
- **Out of scope:** Auto-discovery / proactive invocation. Diagnose runs only when explicitly invoked via the slash command — same convention as `/flow-next:audit` and `/flow-next:capture`.
- **Out of scope:** Mid-session context-switching / handoff doc mechanism. Diagnose's deliverable is the fixed bug + regression test + memory entry written via `flowctl memory add`. Cross-session context is preserved through memory entries that future debuggers find via `memory-scout`, not through ephemeral handoff documents. [inferred]

## Decision Context

- **Why standalone, not folded into `/flow-next:work`?** Diagnose isn't always task-shaped — sometimes a user is chasing a bug that hasn't been spec'd. Standalone keeps it composable. Ralph workers don't use diagnose; they rely on the review-iteration loop already in `/flow-next:work`. [paraphrase]
- **Why memory entries instead of a separate architecture-improvement skill?** flow-next already has the memory primitive (`knowledge/architecture-patterns/`); writing structural findings there matches the Ralph auto-capture pattern and avoids skill proliferation for a capability that may not need its own surface yet. A future epic can build the consumer skill if accumulated entries justify it. [paraphrase]
- **Why IS diagnose Ralph-blocked, like capture / prospect / strategy?** Phase 3's user-checkpoint (rank 3-5 hypotheses → user picks before testing) is the load-bearing discipline. Without it, the methodology degrades to "agent guesses harder" — no better than ad-hoc reasoning. Ralph already has a bug-fix path: workers iterate via the review-backend feedback loop (`NEEDS_WORK` re-spawn) and Ralph auto-captures `bug` track memory on `NEEDS_WORK → SHIP`. A second autonomous bug path through diagnose would duplicate the iteration mechanism without the human checkpoint that justifies the methodology. Diagnose is for the human in the loop. [paraphrase]
- **Why include the HITL bash template?** Some bugs require a human to click. The template gives the agent a structured way to drive the human (via the bash script's `step` / `capture` helpers) so the loop is still feedback-shaped. Without it, "ask the user to click and report back" devolves into chat-based debugging. [paraphrase]
- **Why six phases, refuse-to-advance discipline?** The methodology captures what disciplined senior engineers do automatically when they hit a hard bug — the bullets are well-trodden in the post-mortem literature. The load-bearing flow-next contribution is making the discipline a *phase contract* the host agent can't skip by accident, plus the integration surface with `flowctl memory add` + `GLOSSARY.md` + `knowledge/decisions/` so each diagnosis leaves durable trail for the next one. [paraphrase]

## Requirement coverage

| R-ID | Task |
|------|------|
| R1  | fn-40.M (TBD — populate via /flow-next:plan) |
| R2  | fn-40.M (TBD) |
| R3  | fn-40.M (TBD) |
| R4  | fn-40.M (TBD) |
| R5  | fn-40.M (TBD) |
| R6  | fn-40.M (TBD) |
| R7  | fn-40.M (TBD) |
| R8  | fn-40.M (TBD) |
| R9  | fn-40.M (TBD) |
| R10 | fn-40.M (TBD) |
| R11 | fn-40.M (TBD) |
| R12 | fn-40.M (TBD) |
