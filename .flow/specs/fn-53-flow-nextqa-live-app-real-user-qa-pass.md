## Conversation Evidence

> user: "2nd would be the actual QA stuff. interesting for me here, which browser driving tool does ray recommend in his skill, i think he uses the build-in codex one too, not just agent-browser like we have here. then the other interesting part is that we would derive the test cases from the spec right?"
> [context: Ray Fernando running-bug-review-board (Apache-2.0) — real-user QA pass forbidden from marking PASS by reading source; structured P0/P1/P2 bug reports (persona, steps, expected vs actual, evidence); YES/NO ship verdict; but it must RECONSTRUCT app intent from README/landing/phase-docs (a whole discovering-the-app.md reference). flow-next already HAS the spec (AC, R-IDs, boundaries, decision context) — a structural advantage.]
> [context: flow-next review surface today is all static — impl-review, spec-completion-review, quality-auditor, code-review. Nothing drives the LIVE app like a real user. Memory already has a bug track (track: bug); receipts already gate Ralph state transitions; make-pr already renders an R-ID coverage table.]

> **Browser-driver question resolved (planning research, fn-53):** BRB does **not** default to agent-browser — its ladder is cursor-ide-browser → chrome-devtools-mcp → browser-use → Playwright → Codex Computer Use (macOS fallback). fn-51 flow-next-drive leads with **agent-browser** then chrome-devtools-mcp → Playwright → cursor-ide-browser → manual, with Computer Use reserved for the native (non-CDP) surface. **No driver gap:** every capability BRB's QA discipline needs (viewport emulation, console capture, storage clear, real-Chrome attach for auth) is reachable on fn-51's ladder. Therefore QA borrows BRB's **discipline** and inherits fn-51's **ladder** — it must NOT copy BRB's browser-playbook prose (which names drivers fn-51 doesn't lead with). flow-next-drive's SKILL.md already credits BRB (Apache-2.0); R12 mirrors that wording.

## Goal & Context
<!-- scope: business -->
<!-- Source-tag breakdown: 55% [user] / 30% [paraphrase] / 15% [inferred] -->

Every flow-next review today is code/spec-level — `impl-review`, `spec-completion-review`, `quality-auditor`, `code-review`. Nothing drives the *running* app like an unforgiving real user. `/flow-next:qa` fills that gap: a live-app QA pass that drives the deployed app (via fn-51 flow-next-drive), files structured P0/P1/P2 findings with evidence, and ends with a YES/NO ship verdict. The differentiator vs spec-less QA tools (Ray Fernando's running-bug-review-board) is that flow-next **derives the test scenarios directly from the spec** — acceptance criteria, R-IDs, boundaries — so the host already encodes intent instead of reconstructing it (BRB spends a whole reference reconstructing what we already have in `.flow/specs/`). The design borrows BRB's proven QA discipline (Apache-2.0 — credited) but stays lean (no 18-reference port). Depends on **fn-51** (the surface-aware driver ladder) landing first.

## Architecture & Data Models
<!-- scope: technical -->

A SKILL drives the QA workflow on the host agent: discover (read the spec) → derive scenarios from AC/R-IDs/boundaries → prepare (target URL, test accounts, session hygiene, device matrix) → execute (drive the live app via fn-51 flow-next-drive, capture evidence) → file findings → verdict. Findings are structured P0/P1/P2 reports with reproduction + evidence; they feed the bug memory track (`track: bug`) and can be promoted to flow specs/tasks for the fix. The pass emits a YES/NO verdict as a proof-of-work receipt (same model that gates Ralph), with an R-ID coverage table (reusing the make-pr pattern) for traceability. Canonical skill files use Claude-native tool names; `sync-codex.sh` rewrites for the Codex mirror.

**Pure-skill architecture (no new judgment in flowctl):** QA is host-agent-driven and reuses existing flowctl plumbing — `flowctl spec export-cognitive-aid` (AC/R-ID payload to derive scenarios from), `flowctl memory add --track bug` (file findings, built-in overlap dedup), and a directly-written receipt JSON. The template to follow is `flow-next-make-pr` (R-ID coverage table + the §0.0 "detect Ralph once, route deterministically" pattern + AskUserQuestion-for-info-only). The **only** additive flowctl change is one new `tracker.perEvent.qa` config leaf, accepting `off|comment` (default `off`; `comment` posts the verdict as a tracker comment — the only verb that makes sense for a QA verdict), for the opt-in verdict-post (R9) — edited in BOTH `plugins/flow-next/scripts/flowctl.py` and `.flow/bin/flowctl.py` (dual-copy invariant). The config-key docs in `plugins/flow-next/docs/flowctl.md#config` are part of the same change.

**Receipt schema (`type: qa_verdict`).** The Ralph guard validates only `verdict ∈ {SHIP,NEEDS_WORK,MAJOR_RETHINK}`, so the four QA outcomes are carried in a separate `qa_outcome` field while `verdict` holds the enum-compatible projection:

```json
{ "type": "qa_verdict", "id": "<spec-id>", "mode": "rp|interactive|ralph",
  "verdict": "SHIP|NEEDS_WORK|MAJOR_RETHINK",
  "qa_outcome": "SHIP|NEEDS_WORK|NA|BLOCKED",
  "blocked_reason": "<set only when qa_outcome=BLOCKED>",
  "na_reason": "<set only when qa_outcome=NA>",
  "open_p0p1": ["<finding ids>"],
  "timestamp": "<iso8601>" }
```

Projection: `SHIP→verdict:SHIP`; `NEEDS_WORK→verdict:NEEDS_WORK`; **`BLOCKED→verdict:NEEDS_WORK`** (could not verify → no ship claim on a QA basis); **`NA→verdict:SHIP`** (no driveable UI → live QA raises no objection), with `na_reason` recording why. Written to the caller-supplied `--receipt`/`REVIEW_RECEIPT_PATH` else `.flow/review-receipts/qa-<spec-id>.json` (committed). Transient evidence (screenshots, console dumps) lives under `.flow/tmp/` (gitignored), referenced by path — never inlined.

**fn-51 consumption contract (not a callable API).** A skill is not a function — QA does NOT "call" flow-next-drive. The host agent **reads fn-51's workflow + references and executes the universal driving flow itself** (`observe → snapshot fresh refs → act → verify → capture`), recording an evidence tuple per scenario: `{driver_rung, target_url, viewport, screenshot_path, console_path}`. fn-51 owns the ladder/actuation prose; QA owns scenario authoring, evidence capture, and the verdict — so there is no brittle skill-as-API seam to block the proof point.

## Quick commands
```bash
# Skill stays under the 500-line cap and the Codex mirror is current
wc -l plugins/flow-next/skills/flow-next-qa/*.md
./scripts/sync-codex.sh && git diff --quiet plugins/flow-next/codex/ && echo "codex mirror in sync"

# Smoke: the new tracker.perEvent.qa config leaf round-trips via the production cmd_* path
python3 -m unittest plugins.flow-next.tests.test_qa_tracker_event -q

# Proof point — drive a running app from a real spec, file findings, emit the verdict receipt
# /flow-next:qa fn-<some-spec>     (requires a live deploy + an fn-51 driver)
```

## Edge Cases & Constraints
<!-- scope: technical -->

- Requires a **live deploy + a driver** — with no running app or no driver available, surface the limitation rather than failing; add nothing to the base flow when unused.
- Test accounts / session hygiene: stale storage and reused `+test` emails silently poison fresh-user flows — borrow BRB's hygiene rules (persona suffixing) lean; ask the user when accounts are undocumented.
- Driving fidelity inherits whatever fn-51 resolves for the surface (web ladder vs Computer Use); QA never re-implements driving.
- **No driveable UI** — a spec whose AC are all backend/CLI/non-UI (like most of flow-next's own specs) yields no scenarios; emit a clean **N/A verdict** (distinct from NO and from BLOCKED), never crash.
- **Verdict outcome matrix** — four distinct outcomes carried in `qa_outcome`: `SHIP` (YES) / `NEEDS_WORK` (NO) / **N/A** (no driveable UI) / **BLOCKED** (no live deploy or no driver). The receipt's `verdict` field is the enum-compatible projection (see Architecture → Receipt schema). Honesty rules: incomplete R-ID coverage = NO (not YES); a single open P0 = NO; BLOCKED ≠ FAIL; the verdict rests on captured **evidence**, never on agent narration.
- **Re-run idempotency** — a second QA pass overwrites the latest receipt; findings dedup against existing bug-memory entries via `memory add`'s built-in overlap check (never `--no-overlap-check`), surfacing "matches existing entry X" instead of re-filing.
- **Driver degraded to manual** — if fn-51 resolves only the terminal manual rung (no auto-capture), treat it as the R13 graceful-surface path (BLOCKED, surfaced), not a hard failure.

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
- **R12:** Docs + version bump + CHANGELOG (crediting rayfernando-skills), in two tiers: (a) **repo docs — same-PR acceptance (contributor-blocking):** a new qa skill reference, doc index `plugins/flow-next/docs/README.md`, root README, CLAUDE.md, `.flow/usage.md`, `teams.md` (QA stage), `flowctl.md` (`tracker.perEvent.qa` config), and the planning-surfaced misses **`platforms.md`** (fn-51 dependency note) + **`ralph.md`** (`qa_verdict` receipt note); (b) **maintainer post-merge — NOT contributor-blocking (Gordon handles after merge):** **flow-next.dev** (`~/work/flow-next.dev`) QA docs page + lifecycle/workflow opt-in QA stage + `pnpm build` gate, and **mickel.tech** (`~/work/mickel.tech`) flow-next app page. [inferred]
- **R13:** Opt-in / graceful — QA requires a live deploy + a driver; with no running app or driver it surfaces the limitation rather than failing, and adds nothing to the base flow when unused. [inferred]

## Boundaries
<!-- scope: business -->

- NOT a code review — drives the live app, not the source; complements `impl-review` / `spec-completion-review`, doesn't replace them.
- NOT the driver layer — that's fn-51 flow-next-drive; QA consumes it.
- Does NOT auto-fix product code — files findings and hands off (BRB's "test, document, file, hand off; don't fix unless asked").
- Does NOT port the full BRB surface (iOS sim, Computer-Use specifics, auth-provider test-account playbooks) — orchestrates, stays lean.
- iOS / native-app QA inherits fn-51's surface support; a full native QA workflow may be a later extension.
- Opt-in; requires a live deploy + driver; zero impact on the base flow when absent.

## Strategy Alignment

Active tracks served by this plan:
- **Spec-driven team patterns** — QA derives test scenarios directly from the spec (AC/R-IDs/boundaries) and feeds a verdict receipt back to `spec-completion-review`; a new spec-grounded lifecycle stage between work and make-pr.
- **Cross-platform parity** — canonical Claude-native tool names + Task/Explore dispatch + AskUserQuestion; `sync-codex.sh` mirrors the skill to Codex (R10).
- **flow-swarm preparation** — the verdict-as-receipt slots in as the QA gate of the future per-spec board-triggered executor (R9).
- **Ralph autonomous mode** — runs autonomously when target URL + accounts are configured, emitting the verdict receipt; not a hard Ralph-block (R11).

## Decision Context
<!-- scope: both — conditionally substructured -->

### Motivation
flow-next's review surface is all static; the live-app real-user pass is the gap. flow-next is a BETTER host than a standalone QA skill because the spec already encodes intent — BRB burns a whole reference reconstructing what we already have in `.flow/specs/`. The loop closes: findings → bug memory / specs; verdict → receipt feeding `spec-completion-review`.

### Implementation Tradeoffs
Borrow BRB's proven discipline (scenario / evidence / taxonomy / shard lessons) but stay lean — no 18-reference port; the ≤500-line skill cap holds. Derive scenarios from the spec for traceability (spec-AC ↔ scenario ↔ finding ↔ R-ID). Depends on fn-51 for driving (the driver ladder is the reason fn-51 ships first). Verdict-as-receipt reuses the existing model; QA runs after work, before / around make-pr. The lean BRB borrow is exactly five things: P0/P1/P2 taxonomy + tie-break · evidence rules (console/screenshot/URL/server-row) · five session-hygiene rules + persona suffixing · write-path-first / one-tab-per-shard caution · YES/NO verdict + paste-ready handoff. Everything else (HTML dashboard, iOS-sim ladder, auth-provider playbooks, the triage-heuristics catalog, tracker adapters) is out of scope — flow-next already has the bug memory track, receipts, the make-pr R-ID table, and fn-52 tracker-sync.

### Planning decisions (resolved defaults — v1 scope; override before/at work)
- **QA is NOT a hard Ralph receipt-gate in v1** — it writes a `type: qa_verdict` receipt but does not extend `ralph-guard.py`'s `parse_receipt_path` (a `qa-*.json` path still validates via the existing verdict-enum check; gating a future board-executor is deferred). Keeps R11's "not a hard Ralph-block" literal and avoids a hook change. [decision]
- **Verdict → spec-completion-review is documented-only in v1** — the QA receipt is shaped to be completion-review-compatible (SHIP/NEEDS_WORK + R-ID coverage), but completion-review does not yet *read* the qa receipt; the integration is a documented future hook, not wired this PR. [decision]
- **Device matrix v1 = viewport-emulation only** — one desktop + one mobile viewport via fn-51's web ladder; true cross-device / real-device testing deferred (inherits fn-51's surface support). [decision]

## Dependencies
- **fn-51-flow-next-drive** (hard dependency, status: done) — the surface-aware driver ladder QA consumes; fn-51's own boundaries explicitly defer the QA workflow (scenario authoring, bug filing, verdict) downstream to this spec. QA never re-implements driving.
- Soft reuse (no `add-dep`): **fn-30** (bug memory `track: bug` schema + `memory add` overlap dedup), **fn-42** (make-pr R-ID coverage-table pattern + receipt model), **fn-22** (completion-review verdict feed — documented-only in v1), **fn-52** (tracker-sync — opt-in `tracker.perEvent.qa` verdict post).

## Early proof point
Task **fn-53.1** validates the core thesis end-to-end: derive ≥1 scenario from a real spec, drive it on a running app via **fn-51 flow-next-drive**, and capture one piece of evidence (screenshot + console). It exercises all three load-bearing assumptions at once — spec-as-intent derivation, fn-51 driver consumption, and evidence capture. **When no live target is reachable**, the proof point is satisfied by a **BLOCKED proof receipt** recording the missing target/driver (the R13 path) — it still proves derivation + fn-51 dispatch + the evidence-tuple plumbing, just without a captured screenshot; only re-evaluate the approach if derivation or the fn-51 handoff itself fails.

## Requirement coverage

| R-ID | Task |
|------|------|
| R1 | fn-53.1 (skill skeleton + forbidden-PASS-by-source), fn-53.2 (execution discipline) |
| R2 | fn-53.1 (derive scenarios from spec AC/R-IDs/boundaries) |
| R3 | fn-53.1 (invoke fn-51 flow-next-drive) |
| R4 | fn-53.2 (structured P0/P1/P2 findings + evidence) |
| R5 | fn-53.2 (feed bug memory track + dedup) |
| R6 | fn-53.2 (YES/NO verdict receipt + R-ID coverage table) |
| R7 | fn-53.3 (prepare phase: accounts, session hygiene, device matrix) |
| R8 | fn-53.3 (BRB lean-borrow reference, credited) |
| R9 | fn-53.4 (lifecycle + opt-in tracker.perEvent.qa) |
| R10 | fn-53.1 (canonical tool names), fn-53.5 (codex registration + sync mirror) |
| R11 | fn-53.4 (Ralph-aware-but-not-blocked, detect-once) |
| R12 | fn-53.6 (three-surface docs + flowctl.md config + CHANGELOG + credit) |
| R13 | fn-53.4 (opt-in / graceful degradation) |

**Task order (serial, merge-safe):** fn-53.1 → fn-53.2 → fn-53.3 → fn-53.4 → fn-53.5 → fn-53.6. fn-53.3 and fn-53.4 are serialized (not parallel) and split into **disjoint reference files** — `references/qa-discipline.md` (.3) vs `references/autonomy.md` (.4) — so the shared `workflow.md` is only edited in the pre-placed disjoint phase sections fn-53.1 lays down. fn-53.5 (codex registration + sync mirror + skill smoke) and fn-53.6 (docs + flowctl.md + CHANGELOG + version bump) split the former single docs/registration task to keep both M-sized.
