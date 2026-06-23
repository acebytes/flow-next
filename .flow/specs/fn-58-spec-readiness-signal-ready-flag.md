## Conversation Evidence

> user: "a new quite light weight skill that can be called by / loop in claude code or / goal in codex that checks what the next spec is that is marked as ready (do we have thats status) or the next spec that has a corresponding status in the tracker software configured in a project"
> user: "if you look at our flow, you are conflating spec with plan i think. in our flow, the brunt of the work goes into the spec. the plan is just the agent planning the actual implementation surely"
> user: "we need a new status i think for users that do not have tracker-sync connected, correct? for those that do we need to determine which state equates to the ready state."
> user: "this will mean touching some skills like capture/interview etc to potentially ask them if the spec is ready and the plan skill to check and warn if not ready (soft block), not sure, think it all through, not too complicated"
> user: "this can all be mostly solved with prompting, not scripting and already works well even without loop and goal"
> user: "remember that the docs and flow-next.dev (~/work/flow-next.dev) will need comprehensive updates too" [verbatim prefix]

## Goal & Context
<!-- Source: 60% [user] / 30% [paraphrase] / 10% [inferred] -->

The spec is flow-next's load-bearing artefact and the point where human judgment concentrates — the plan and everything downstream is mechanical agent work. Today a spec has only `open | done`; there is no signal for "the human considers this spec complete enough to hand to an agent." This spec adds a **readiness** signal: a human-owned gate marking a spec ready for autonomous (or confident manual) execution.

It is the entry gate the forthcoming build-loop will consume, but it stands on its own as backlog hygiene — knowing which specs are blessed vs still-draft, and being nudged away from planning a half-baked spec — useful even with no loop in play. Target users: solo devs and teams on flow-next, with and without a tracker connected.

## Architecture & Data Models
<!-- Source: 70% [paraphrase] / 20% [user] / 10% [inferred] -->

A `ready` boolean on the spec record (default `false`), persisted in the spec JSON sidecar and orthogonal to `status` (`open|done`). Readiness has **one local read path** with two write sources depending on config:

- **No tracker** — the human sets `ready` directly (flowctl command, or the capture/interview prompt).
- **Tracker connected** — tracker-sync projects the configured tracker "ready" state (`tracker.readyState`) onto the same local `ready` flag on each sync; the tracker is authoritative for those users.

flowctl owns the field + set/query plumbing; the skills own *when* to set it and *when* to warn (prompting, not scripting).

### Resolved at planning (research + gap analysis, 2026-06-10)

- **Lazy on-disk, explicit in JSON output:** the sidecar carries `ready` only after a toggle (absent = false; `spec create` never writes it — maximal R7 invisibility, zero working-tree churn). But every JSON read surface (`show`, `specs`, `list`) emits an explicit `"ready": <bool>` (read-side default false) so consumers (fn-59's selector) always see a stable boolean. Note: `cmd_specs`/`cmd_list` build explicit dicts (no spread) — `ready` must be added to each; `normalize_epic` deliberately does NOT backfill the key (that would defeat lazy purity on the next write).
- **Idempotent toggles:** `spec ready` no-ops (no write, no `updated_at` bump) when already true; `spec unready` no-ops when the key is absent or false. This is what lets capture `--rewrite` call `unready` unconditionally without turning every rewritten draft into a readiness-adopter. `.M` (task) ids are rejected with a clear error; `done` specs are allowed (readiness is status-orthogonal). `cmd_epic_*` aliases + tracker-handle resolution via `resolve_spec_id_arg` (id-grammar lesson).
- **One-way pull:** readiness is tracker→local only. Local `spec ready` is never pushed to the tracker and will be overwritten on the next sync for tracker users. Consequently the capture/interview "mark ready?" prompt is **gated OFF when `tracker.readyState` is configured** (don't invite a local edit sync silently reverts).
- **Projection observability:** the projection emits an `--event`-tagged receipt only when it CHANGES the local flag (silent on no-op echo); configured-name-not-found → warn `noop` receipt + flag untouched + sync continues. Linear match = case-insensitive trimmed state-NAME (names are non-unique and renamable — type alone can't distinguish Todo from Ready); GitHub = label (pre-created at ceremony time, 422-guarded). GitHub semantics: label present on the issue → ready=true; label absent → ready=false (a normal state, not an error); only an unresolvable repo label / invalid config produces the warn-`noop` receipt.
- **Config placement:** `tracker.readyState` at the tracker top level (sibling of `conflictTiebreak`/`staleAfterHours`), NOT under `perTracker` — it's a single scalar.
- **Adoption-gated everywhere (resolves the R5/R7 tension):** the same in-use gate (≥1 ready spec OR `tracker.readyState` set) governs BOTH plan's R6 warning AND the capture/interview mark-ready prompts — non-adopters see zero new prompts or warnings anywhere; adoption enters via `flowctl spec ready`, the tracker ceremony, or prime. Modeled on the STRATEGY husk-vs-presence guard. Plan's question splits by mode: no `readyState` → proceed / mark-ready-then-proceed / abort; `readyState` configured → proceed / abort / update-tracker-state-then-rerun (never offer local mark-ready when the tracker is authoritative). Default = proceed; non-interactive/Ralph auto-proceeds with a log line (net-new detection).
- **Interview never auto-resets** `ready` (it refines in place); only capture `--rewrite` (a full re-authoring) resets — and announces the reset in the post-write summary (one line, no extra confirm).
- **Sequencing note:** fn-54's prompt-optimization passes over capture/interview/plan must baseline AFTER fn-58 lands (annotate fn-54 when its tasks are created).

## API Contracts
<!-- scope: technical -->

- **`flowctl spec ready <id>` / `flowctl spec unready <id>`** — set / clear the flag. [paraphrase]
- **`ready` exposed** in `flowctl specs --json` and `flowctl show <id> --json`; surfaced as a badge in `flowctl list` / `specs` output. [paraphrase]
- **Config key `tracker.readyState`** — records which tracker workflow state maps to readiness, resolved per tracker type (Linear: a workflow-state name; GitHub: a label). [user]

## Edge Cases & Constraints
<!-- scope: technical -->

- Default `false`; pre-existing specs read as not-ready (absent flag). Non-adopters never see readiness surfaced unless they engage it. [inferred]
- For tracker users, each sync re-projects readiness — a local flag edit may be overwritten by the tracker state (tracker authoritative). [paraphrase]
- `capture --rewrite` resets `ready` → `false` (material spec change re-opens the blessing). [inferred]
- Readiness is orthogonal to `status` — a ready spec stays `open` through planning and work. [paraphrase]
- Linear readiness is a state-**name** match layered on the existing `state.type` mapping — a custom "Ready" state typically carries `state.type=unstarted`, which alone cannot distinguish Todo from Ready (same name-override pattern as status-sync). [inferred]
- GitHub issues have no workflow states — for GitHub-tracked projects `tracker.readyState` resolves to a label. [inferred]
- The `ready` flag lives in the committed spec JSON (same placement as `plan_review_status`), so tracker-driven re-projection may produce working-tree changes on sync — accepted, identical to existing status-sync behavior. [inferred]
- Readiness projection rides the fn-57 observability layer (shipped 1.11.0): tracker-sync ops that project `readyState` emit `--event`-tagged receipts, auditable via the read-only `flowctl sync check`. [inferred]
- The capture/interview "mark ready?" prompt (R5) reuses the post-approve consent-question pattern capture gained in fn-57.7 (vocabulary scan → read-back option → post-write) — the precedent is now shipped, not hypothetical. [inferred]

## Acceptance Criteria
<!-- scope: both -->

- **R1:** A spec carries a `ready` boolean (default `false`), persisted in its record and exposed via `flowctl specs --json` / `show --json`. [paraphrase]
- **R2:** A human can mark / unmark a spec ready via flowctl (`spec ready` / `spec unready`), and the state is visible in spec listings as a badge. [user]
- **R3:** For tracker-connected projects, the configured tracker "ready" state projects onto the local `ready` flag on sync, giving readiness a single local read path. [user]
- **R4:** tracker-sync setup asks which tracker workflow state means "ready" and stores it as `tracker.readyState` (a Linear state name or a GitHub label, per tracker type). [user]
- **R5:** capture and interview offer an optional end-of-authoring prompt to mark the spec ready (default: keep draft) — shown only once readiness is adopted in the repo (≥1 ready spec or `tracker.readyState` configured); first adoption enters via `flowctl spec ready`, the tracker ceremony, or prime's coverage report. [user]/[inferred]
- **R6:** plan soft-checks readiness — when a spec isn't ready it warns and offers proceed / abort; never a hard block. [user]
- **R7:** The readiness gate is opt-in and invisible to users who never engage it — no prompts, no warnings, no badge noise anywhere until readiness is adopted; existing specs and non-loop workflows behave exactly as before. [inferred]
- **R8:** Documentation and the flow-next.dev site are updated to cover readiness — architecture spec-lifecycle, GLOSSARY "Ready" term, flowctl command reference, tracker-sync `readyState` mapping; site pages + both navbars + changelog. [user]

## Boundaries
<!-- scope: business -->

- This spec adds the **readiness signal only**. The build-loop and ship-loop skills that consume it are separate, forthcoming specs (this one is their shared dependency). [user]
- No new value is added to the `status` enum — `open|done` is unchanged; readiness is a separate orthogonal flag. [paraphrase]
- No automatic readiness inference — readiness is always an explicit human action or a tracker-state projection, never the agent judging a spec "looks ready." [inferred]
- Parallel execution, PR merge, and release are out of scope (other specs). [paraphrase]
- **Readiness is pull-only for tracker users** — local `spec ready` is not projected out to the tracker and is overwritten on next sync; tracker users set readiness on the board. No outbound readiness push in this spec. [inferred]
- **No `--ready` filter flag** on `specs`/`list` in v1 — `specs --json` + jq covers fn-59's selection; add the flag only if the loop needs it. [inferred]

## Decision Context
<!-- scope: both -->

Readiness is a **flag, not a new `status` value** — `status=open|done` is checked throughout flowctl and stays clean, while readiness is orthogonal and persists through planning and work. The gate is **opt-in / invisible-by-default** so the large body of existing specs and non-loop users are undisturbed. The signal is deliberately **human-owned (or tracker-projected), never agent-inferred** — readiness is the human's gate, the place intent concentrates, because the spec carries the weight and the plan is mechanical agent work. flowctl owns storage; skills own the workflow — the standard skill + thin-plumbing split. [strategy:Ralph autonomous mode]

## Strategy Alignment
<!-- STRATEGY.md populated 2026-05-16 -->

- Aligns with the **Ralph autonomous mode** track — readiness is the human-blessed entry gate that lets a host-driven loop execute a spec unattended while the human keeps control of *direction*. [strategy:Ralph autonomous mode]
- Aligns with the approach principle *"the host agent IS the intelligence; flowctl provides thin atomic helpers"* — readiness is a thin flowctl field; all judgment is skill prompting. [strategy]
- No conflict with *"Not working on: built-in CI runners / SaaS"* — readiness is in-repo metadata only.

## Quick commands

```bash
# Lazy purity: fresh spec has no ready key; JSON still reports false
.flow/bin/flowctl show fn-58 --json | jq .ready          # -> false (explicit)
grep -c '"ready"' .flow/specs/fn-1.json || true           # -> 0 (absent on old sidecars)

# Toggle + badge + idempotency
.flow/bin/flowctl spec ready fn-58 --json && .flow/bin/flowctl specs | grep fn-58   # badge shown
.flow/bin/flowctl spec unready fn-58 --json                                          # reset

# Tests
cd plugins/flow-next && python3 -m unittest discover -s tests -p "test_spec_ready.py" -v
```

## Early proof point

Task fn-58.1 validates the core model (lazy on-disk flag + explicit-false JSON surfaces + idempotent toggles). If lazy-absent + read-default-false proves awkward for any consumer, re-evaluate toward eager-write before wiring tracker projection or skills.

## Requirement coverage

| Req | Description | Task(s) | Gap justification |
|-----|-------------|---------|-------------------|
| R1 | `ready` bool, default false, in record + `specs/show --json` | fn-58.1 | — |
| R2 | `spec ready`/`unready` + listing badge | fn-58.1 | — |
| R3 | Tracker ready-state projects onto local flag (single read path) | fn-58.2 | — |
| R4 | Ceremony asks + stores `tracker.readyState` (Linear name / GitHub label) | fn-58.2 | — |
| R5 | capture + interview optional mark-ready prompt | fn-58.3 | — |
| R6 | plan soft-check (adoption-gated, default proceed) | fn-58.3 | — |
| R7 | Opt-in, invisible to non-adopters | fn-58.1 (lazy write), fn-58.3 (adoption gate) | — |
| R8 | Docs + site + changelog + version | fn-58.4 | — |
