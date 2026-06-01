## Conversation Evidence

> user: "2nd would be the actual QA stuff. interesting for me here, which browser driving tool does ray recommend in his skill, i think he uses the build-in codex one too, not just agent-browser like we have here. then the other interesting part is that we would derive the test cases from the spec right?"
> [context: Ray Fernando running-bug-review-board (Apache-2.0) — real-user QA pass forbidden from marking PASS by reading source; structured P0/P1/P2 bug reports (persona, steps, expected vs actual, evidence); YES/NO ship verdict; but it must RECONSTRUCT app intent from README/landing/phase-docs (a whole discovering-the-app.md reference). flow-next already HAS the spec (AC, R-IDs, boundaries, decision context) — a structural advantage.]
> [context: flow-next review surface today is all static — impl-review, spec-completion-review, quality-auditor, code-review. Nothing drives the LIVE app like a real user. Memory already has a bug track (track: bug); receipts already gate Ralph state transitions; make-pr already renders an R-ID coverage table.]

## Goal & Context
<!-- scope: business -->
<!-- Source-tag breakdown: 55% [user] / 30% [paraphrase] / 15% [inferred] -->

Every flow-next review today is code/spec-level — `impl-review`, `spec-completion-review`, `quality-auditor`, `code-review`. Nothing drives the *running* app like an unforgiving real user. `/flow-next:qa` fills that gap: a live-app QA pass that drives the deployed app (via fn-51 flow-next-drive), files structured P0/P1/P2 findings with evidence, and ends with a YES/NO ship verdict. The differentiator vs spec-less QA tools (Ray Fernando's running-bug-review-board) is that flow-next **derives the test scenarios directly from the spec** — acceptance criteria, R-IDs, boundaries — so the host already encodes intent instead of reconstructing it (BRB spends a whole reference reconstructing what we already have in `.flow/specs/`). The design borrows BRB's proven QA discipline (Apache-2.0 — credited) but stays lean (no 18-reference port). Depends on **fn-51** (the surface-aware driver ladder) landing first.

## Architecture & Data Models
<!-- scope: technical -->

A SKILL drives the QA workflow on the host agent: discover (read the spec) → derive scenarios from AC/R-IDs/boundaries → prepare (target URL, test accounts, session hygiene, device matrix) → execute (drive the live app via fn-51 flow-next-drive, capture evidence) → file findings → verdict. Findings are structured P0/P1/P2 reports with reproduction + evidence; they feed the bug memory track (`track: bug`) and can be promoted to flow specs/tasks for the fix. The pass emits a YES/NO verdict as a proof-of-work receipt (same model that gates Ralph), with an R-ID coverage table (reusing the make-pr pattern) for traceability. Canonical skill files use Claude-native tool names; `sync-codex.sh` rewrites for the Codex mirror.

## Edge Cases & Constraints
<!-- scope: technical -->

- Requires a **live deploy + a driver** — with no running app or no driver available, surface the limitation rather than failing; add nothing to the base flow when unused.
- Test accounts / session hygiene: stale storage and reused `+test` emails silently poison fresh-user flows — borrow BRB's hygiene rules (persona suffixing) lean; ask the user when accounts are undocumented.
- Driving fidelity inherits whatever fn-51 resolves for the surface (web ladder vs Computer Use); QA never re-implements driving.

## Acceptance Criteria
<!-- scope: both -->

- **R1:** New `/flow-next:qa` skill — a live-app real-user QA pass; drives the running app like an unforgiving customer, **forbidden from marking PASS by reading source** (the gap: all current flow-next review is static). [user]/[paraphrase]
- **R2:** Scenarios derived DIRECTLY from the spec — acceptance criteria → test scenarios; R-IDs → coverage / traceability (reuse the make-pr R-ID coverage-table pattern); boundaries → what NOT to test (prevents false bugs); decision context → expected behavior. The spec-as-intent advantage over spec-less QA tools. [user]/[paraphrase]
- **R3:** Drives the live app via **fn-51 flow-next-drive** (surface-aware driver ladder) — depends on fn-51 landing; QA never re-implements driving. [paraphrase]
- **R4:** Findings filed as structured P0/P1/P2 reports (persona, steps-to-reproduce, expected vs actual, evidence: console / screenshots / URL); filed immediately on FAIL — evidence discipline + the P0/P1/P2 taxonomy borrowed from BRB. [paraphrase]
- **R5:** Findings feed the bug memory track (`track: bug`) and can be promoted to flow specs/tasks for the fix; bidirectional traceability spec-AC ↔ scenario ↔ finding ↔ R-ID. [paraphrase]
- **R6:** Pass ends with a YES/NO ship verdict + open P0/P1 list, emitted as a proof-of-work receipt (fits the existing receipt model); the verdict can feed `spec-completion-review` ("does the *live app* satisfy the AC, not just the code"). [user]/[paraphrase]
- **R7:** Prepare phase — target URL / app, test accounts, session hygiene (stale storage, persona suffixing), device matrix (mobile / tablet / desktop); borrowed lean from BRB; ask the user when undocumented. [paraphrase]
- **R8:** Lean borrow from Ray's running-bug-review-board (credited) — scenario derivation, taxonomy, evidence rules, session hygiene, parallel-shard lessons. Do NOT port the full surface (iOS sim, Computer-Use specifics, Clerk/Auth0 test-account playbooks); stay within the ≤500-line skill discipline. [paraphrase]
- **R9:** Lifecycle position — runs after `/flow-next:work`, around / before `make-pr`, as a QA stage; slots in as the qa gate of the future per-spec board-triggered executor; verdict optionally posts to the tracker when fn-52 sync is configured (opt-in). [paraphrase]
- **R10:** Cross-platform — canonical Claude-native tool names + Task/Explore subagent dispatch + AskUserQuestion; `sync-codex.sh` rewrites for the Codex mirror. [strategy:Cross-platform parity]
- **R11:** Runs interactively AND autonomously — autonomous when target URL + test accounts are configured (emits the verdict receipt); asks the user when they're undocumented. Not a hard Ralph-block. [paraphrase]
- **R12:** Three-surface docs + version bump + CHANGELOG (crediting rayfernando-skills): (a) **repo** — a new qa skill reference, doc index `plugins/flow-next/docs/README.md`, root README, CLAUDE.md, `.flow/usage.md`, `teams.md` (QA stage); (b) **flow-next.dev** (`~/work/flow-next.dev`) — a new QA docs page + the lifecycle guide / workflow pages gain an **optional, opt-in** QA stage, run the `pnpm build` gate; (c) **mickel.tech** (`~/work/mickel.tech`) — the flow-next app page, **maintainer-only (Gordon post-merge; contributor PRs skip it)**. [inferred]
- **R13:** Opt-in / graceful — QA requires a live deploy + a driver; with no running app or driver it surfaces the limitation rather than failing, and adds nothing to the base flow when unused. [inferred]

## Boundaries
<!-- scope: business -->

- NOT a code review — drives the live app, not the source; complements `impl-review` / `spec-completion-review`, doesn't replace them.
- NOT the driver layer — that's fn-51 flow-next-drive; QA consumes it.
- Does NOT auto-fix product code — files findings and hands off (BRB's "test, document, file, hand off; don't fix unless asked").
- Does NOT port the full BRB surface (iOS sim, Computer-Use specifics, auth-provider test-account playbooks) — orchestrates, stays lean.
- iOS / native-app QA inherits fn-51's surface support; a full native QA workflow may be a later extension.
- Opt-in; requires a live deploy + driver; zero impact on the base flow when absent.

## Decision Context
<!-- scope: both — conditionally substructured -->

### Motivation
flow-next's review surface is all static; the live-app real-user pass is the gap. flow-next is a BETTER host than a standalone QA skill because the spec already encodes intent — BRB burns a whole reference reconstructing what we already have in `.flow/specs/`. The loop closes: findings → bug memory / specs; verdict → receipt feeding `spec-completion-review`.

### Implementation Tradeoffs
Borrow BRB's proven discipline (scenario / evidence / taxonomy / shard lessons) but stay lean — no 18-reference port; the ≤500-line skill cap holds. Derive scenarios from the spec for traceability (spec-AC ↔ scenario ↔ finding ↔ R-ID). Depends on fn-51 for driving (the driver ladder is the reason fn-51 ships first). Verdict-as-receipt reuses the existing model; QA runs after work, before / around make-pr.

## Requirement coverage

| R-ID | Task |
|------|------|
| R1 | fn-53.M (TBD — populate via /flow-next:plan) |
| R2 | fn-53.M (TBD) |
| R3 | fn-53.M (TBD) |
| R4 | fn-53.M (TBD) |
| R5 | fn-53.M (TBD) |
| R6 | fn-53.M (TBD) |
| R7 | fn-53.M (TBD) |
| R8 | fn-53.M (TBD) |
| R9 | fn-53.M (TBD) |
| R10 | fn-53.M (TBD) |
| R11 | fn-53.M (TBD) |
| R12 | fn-53.M (TBD) |
| R13 | fn-53.M (TBD) |
