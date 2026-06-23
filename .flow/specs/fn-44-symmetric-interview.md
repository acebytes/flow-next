# Symmetric interview: --scope=business|technical|both flag for /flow-next:interview

## Goal & Context

Implementing agents make better decisions when they have **structured business context** to ground them. Without it, every implementation call — UX phrasing, error message tone, defensive design depth, gold-plating-vs-MVP, performance vs simplicity trade-offs, regulatory hardening — becomes a guess based on indirect signals. The agent's failure modes are over-engineering (when in doubt, build more) and missed-priority calls (defaulting to the wrong dimension of the trade-off).

A spec with rich business context turns those guesses into structured input:

- **Target user / persona** → tone of error messages, jargon density, accessibility level, technical-literacy assumptions
- **Problem framing / why-now** → disambiguates the actual goal (same "add OAuth" words → different architectures depending on whether it's internal SSO or third-party distribution)
- **Success metric** → resolves "fast vs robust vs extensible" trade-offs the same way the PO would
- **MVP boundary** → stops the agent gold-plating; no extensibility hooks the PO explicitly didn't ask for
- **Business constraints** (regulatory / budget / time / competitive) → changes data-handling defaults, infra choices, robustness vs ship-date trade-offs
- **What NOT to build** → explicit non-goals the agent doesn't accidentally cross
- **Prioritization rationale** → consistency when mid-implementation trade-offs surface
- **Business risks** → defensive design choices the agent would otherwise default-skip
- **UX implications** → loading states, edge-case messaging, hidden-vs-user-facing decisions

`docs/teams.md` already documents a "Symmetric interview" pattern where `/flow-next:interview` is run twice on the same spec — first to fill the business layer, then to fill the technical layer. Both passes target the **same** `.flow/specs/<id>.md` file; the handovers are reviewable states of one evolving spec. This matches spec-driven-development best practice (GitHub Spec Kit, Tessl, Anthropic): a single durable spec that grows through layers, vs Kiro's split-file model (`requirements.md / design.md / tasks.md`).

Today the symmetry is convention, not structure. The interview skill ships **one technical-heavy question bank** (`questions.md`). There is **no business question bank** at all. A PO running `/flow-next:interview fn-1` today gets asked about cache strategies and rollback plans — not about problem framing, success metrics, or what-not-to-build.

`/flow-next:capture` does extract some business signal (target user, problem framing) from conversation, but it's light and inference-heavy (`[inferred]` tags) and skips success metrics, MVP boundary, prioritization rationale, business risks. **No surface in flow-next today deeply refines business context.** fn-44 adds that surface.

There is also a **documentation bug** discovered during this spec's scoping pass: `docs/teams.md` and `GLOSSARY.md` consistently use the nouns "business spec" and "full spec" to label handovers #1 and #2. Both read as distinct artefacts, contradicting the one-spec model that the artefact-path columns show. The ASCII flowchart in the Symmetric-interview subsection repeats the noun pattern. This spec fixes the source naming.

This spec makes the symmetric pattern structural by adding `--scope=business|technical|both`. **Default is `technical`** — preserves the current single-pass behavior exactly for solo devs running multi-agent loops (STRATEGY.md's primary audience), no breaking change. Teams adopting the symmetric pattern opt in via explicit `--scope=business` (PO pass) then `--scope=technical` (tech lead pass), or `--scope=both` for a single combined pass. The business pass fills the business sections of the one spec; the technical pass **reads** those business sections as constraint context and fills the technical sections.

Five docs alignments ride with the flag:

1. **Canonical spec template extracted** to `plugins/flow-next/templates/spec.md` (peer to existing `templates/memory/`). Single source of truth for spec structure.
2. **`docs/teams.md` + `GLOSSARY.md` naming fix.** Misleading noun pairs replaced with layer-state phrasing that makes the same-artefact-evolving model unambiguous.
3. **`docs/teams.md` flow chart updates.** Lifecycle map (mermaid) annotated for scoped interview. Symmetric-interview ASCII flowchart updated to use `--scope` flag + non-misleading handover labels.
4. **`STRATEGY.md` "Our approach"** gets a single-sentence anchor stating the single-evolving-spec choice (explicit one-spec-per-feature, vs Kiro's split-file model).
5. **mickel.tech flow-next page** updated to mention structured business-context capture as part of the externally-facing "spec-driven, agent-first" story.

## Architecture & Data Models

**One spec, two passes, scoped section ownership.**

Single artefact: `.flow/specs/<spec-id>.md`. The two interview passes touch different sections of the same file. The flag controls section ownership at write time so neither pass clobbers the other.

Supplementary design docs (e.g., `docs/design/<topic>.md`, `docs/adr/NNN-*.md`) are NOT specs and are out of scope for the "one spec" rule. They are cross-linked from the spec when load-bearing; the spec remains the single source of truth for R-IDs and acceptance.

### Canonical spec template — file location

The template lives at `plugins/flow-next/templates/spec.md`. Structure:

- Frontmatter header explaining purpose + which skills consume the template
- Section-by-section scaffold with one-line guidance per section
- Each section header annotated with scope owner (`<!-- scope: business -->`, `<!-- scope: technical -->`, `<!-- scope: both -->`) as an HTML comment
- Footer with cross-link to docs/teams.md "Symmetric interview" and to CLAUDE.md "Creating a spec"

Consumers reference the template via:
- Bash: `${CLAUDE_PLUGIN_ROOT:-${DROID_PLUGIN_ROOT}}/templates/spec.md`
- Codex: same after sync-codex rewrite
- CLAUDE.md: relative link to the file (avoid re-embedding the section list)

### Canonical spec sections (post-fn-44)

The template encodes this table:

| # | Section | Scope owner | Notes |
|---|---|---|---|
| 1 | `## Goal & Context` | business | Problem, motivation, why-now, target user |
| 2 | `## Architecture & Data Models` | technical | Component boundaries, integration |
| 3 | `## API Contracts` | technical | Endpoints, interfaces, shapes |
| 4 | `## Edge Cases & Constraints` | technical | Mostly technical; biz constraints feed in from Goal & Context |
| 5 | `## Acceptance Criteria` | both (co-authored) | Outcome predicates (biz) + verifiable predicates (tech) — R-IDs append-only across passes |
| 6 | `## Boundaries` | business | What's explicitly out of scope (biz priority decision) |
| 7 | `## Decision Context` | both (conditionally substructured) | **Flat** when only a `--scope=technical` (zero-flag default) pass has run — same shape as 1.0.2. **Substructured** with two H3s (`### Motivation` biz-owned + `### Implementation Tradeoffs` tech-owned) only after a biz pass has run, OR under `--scope=business`/`--scope=both`, OR when an existing spec already has the H3s. This conditional preserves R22 backward-compat for solo devs who never opt into the symmetric pattern. |

Already-existing + new auxiliary sections (skill-conditional, scope-aware):

- `## Strategy Alignment` / `## Strategy Conflicts` (strategy-aware mode)
- `## Glossary Conflicts` (doc-aware mode)
- `## Conversation Evidence` (written by `/flow-next:capture`)
- `## Resolved via Codebase` (interview audit trail — technical-pass scope)
- `## Resolved via Project Docs` (interview audit trail — business-pass scope, NEW per R26)

### Question banks

Two banks under `plugins/flow-next/skills/flow-next-interview/`:

- `questions-business.md` (NEW) — problem framing, target user/persona, success metrics, MVP boundary, business constraints (regulatory / budget / time), what NOT to build, prioritization rationale, business risks, UX implications.
- `questions-technical.md` (renamed from existing `questions.md` for symmetry) — existing technical buckets unchanged.

Both banks share the existing `Pre-Question Taxonomy` and `Interview Guidelines` blocks — hoisted to `questions-shared.md`.

### Flag parsing + pass behavior

Flag parsing lives in SKILL.md, slotting into the existing `--docs / --no-docs / --strategy / --no-strategy` stripping block. After stripping, `$SCOPE` is one of `business | technical | both`. Default when omitted: `technical`.

Technical pass behavior when `$SCOPE == technical`: BEFORE asking questions, read the spec's existing `Goal & Context`, `Boundaries`, `Decision Context (motivation)`, and outcome-AC sections **if populated**. When populated, cite them in the interview opener. When empty (typical solo-dev case: never ran `--scope=business`), proceed silently — the default is technical-only, business-section absence is the normal state.

Business pass behavior when `$SCOPE == business`: do NOT touch Architecture / API Contracts / Edge Cases. Write a placeholder `*Pending technical-scope interview pass.*` line under each technical section so it's visible in the read-back that those sections are intentionally empty AFTER a business pass.

`both` runs business pass first, writes the business sections, then continues seamlessly into technical pass with the just-written business content as input.

Idempotency: re-running any scope on a spec with already-filled sections refines (does not overwrite) the corresponding section. Sections owned by the other scope are left untouched.

## API Contracts

CLI surface — flag added to `/flow-next:interview` skill, with a **thin deterministic flowctl helper layer** (`flowctl scope resolve / bank / write-policy` + `flowctl spec skeleton`) that both SKILL.md and unit tests invoke. This preserves the architectural rule: skill drives the workflow, flowctl provides atomic helpers. Skill never re-implements parse/policy logic inline — it calls the flowctl subcommands.

```
/flow-next:interview <spec-id-or-path> [--scope=business|technical|both]
                                       [--biz | --tech]                   # short aliases
                                       [--docs | --no-docs]               # existing
                                       [--strategy | --no-strategy]       # existing
```

Resolution rules:

- `--scope=technical` is the default. Preserves current single-pass behavior; no breaking change for solo devs.
- `--biz` ⇔ `--scope=business`. `--tech` ⇔ `--scope=technical`. Short aliases for ergonomic typing.
- Conflicting flags (`--biz --tech`, `--scope=business --tech`, `--scope=foo`) error cleanly with explicit message.
- `--scope=technical` keeps the existing `--docs` / `--strategy` autodetect cascade unchanged.
- `--scope=business` does NOT auto-activate doc-awareness (a PO often doesn't have the glossary/strategy context yet); explicit `--strategy --docs` opt-in.

Template file API: read-only. Consumers (skills, CLAUDE.md) reference via path; no programmatic mutation.

## Edge Cases & Constraints

- **Empty spec, default `--scope=technical`** (solo dev fresh-spec): writes technical sections, never touches business sections. This IS the current behavior — preserved as default.
- **Empty spec, `--scope=business` first**: writes business sections, leaves technical sections with `*Pending technical-scope interview pass.*` placeholders.
- **Spec with biz pass done, then `--scope=technical`**: technical pass reads business sections, cites them, asks technical questions only. Handover happens here.
- **Spec with biz pass done, then `--scope=both`**: refine-mode — re-asks only for diffs/additions on the business side, then runs technical pass normally.
- **`--scope=both` interrupted between phases**: business sections written, technical phase aborted. Re-running `--scope=technical` later completes the spec.
- **R-ID continuity across passes**: business pass allocates R1-Rn; technical pass appends Rn+1-Rm to the same `## Acceptance Criteria` section. Never renumber.
- **Doc-aware behavior with scope**: glossary and strategy lookups are scope-agnostic — both passes can hit them. The DOC_AWARE matrix in SKILL.md gets `--scope` as an orthogonal axis, not nested.
- **Section merge contract (scope-aware writes)**: Each pass must preserve unowned content byte-for-byte:
  - Technical pass MUST preserve existing business-section bodies (`Goal & Context` / `Boundaries`) verbatim. For `## Decision Context`: **(a)** when the section is FLAT (no H3 subsections) — default zero-flag-tech case on a fresh/legacy spec — write/refine the flat body in place; do NOT introduce H3 substructure; this preserves R22 1.0.2 shape. **(b)** when `### Motivation` already exists (a biz pass ran earlier, OR an existing spec has the substructure) — preserve `### Motivation` body byte-for-byte; write/refine ONLY the `### Implementation Tradeoffs` subsection. May overwrite `*Pending technical-scope interview pass.*` placeholder strings under tech-owned section headers.
  - Business pass MUST preserve existing technical-section bodies (`Architecture & Data Models` / `API Contracts` / `Edge Cases & Constraints`) verbatim. For `## Decision Context`: **(a)** when the section is FLAT (no H3 subsections) — first biz pass on a tech-already-populated spec — promote the flat body to `### Implementation Tradeoffs` (preserve byte-for-byte) and write the new `### Motivation` as a sibling H3. **(b)** when H3s already exist — preserve `### Implementation Tradeoffs` byte-for-byte; write/refine ONLY `### Motivation`. If a tech section is empty, the biz pass writes the placeholder string under it; if the tech section already has content, leave it untouched.
  - Auxiliary sections (`Strategy Alignment` / `Glossary Conflicts` / `Conversation Evidence` / `Resolved via Codebase`) are preserved across scope changes — neither pass deletes or rewrites them.
  - `## Acceptance Criteria` R-IDs are append-only across passes per fn-29 rules. Never renumber. Never replace existing entries. New criteria from a later pass take the next unused number.
- **Codex mirror**: question banks copy as-is. SKILL.md changes go through `sync-codex.sh` rewrite. The `templates/spec.md` file mirrors via sync if it contains Claude-native references; otherwise it copies as-is. R30 mirror guard validates the new files land in `plugins/flow-next/codex/`.
- **Template drift**: if a skill embeds the section list rather than referencing the template path, that's a defect — `sync-codex.sh` validation gets a small guard (R21) that flags duplicated section lists in skill files.
- **Flow-chart updates as separate task**: README + docs/teams.md mermaid blocks audited individually to avoid mass-edit conflicts; each gets its own task-level change.

## Acceptance Criteria

- **R1**: `--scope=business|technical|both` flag is parsed in `flow-next-interview/SKILL.md`; default `technical`; preserves current single-pass behavior when flag is omitted (zero-breakage backward compat).
- **R2**: `--biz` and `--tech` short aliases resolve to `--scope=business` and `--scope=technical` respectively.
- **R3**: Conflicting flags (`--biz --tech`, `--scope=<invalid>`, `--scope=business --tech`) error cleanly with an explicit "conflicting scope flags" message.
- **R4**: New `questions-business.md` exists with question buckets covering: problem framing, target user/persona, success metrics, MVP boundary, business constraints, what-not-to-build, prioritization rationale, business risks, UX implications.
- **R5**: Existing `questions.md` is renamed to `questions-technical.md`; SKILL.md references the correct file by scope.
- **R6**: `--scope=business` pass writes only the business-owned sections (`Goal & Context`, `Boundaries`, `## Decision Context > ### Motivation`, outcome-AC); leaves technical-owned sections with explicit `*Pending technical-scope interview pass.*` placeholders.
- **R7**: `--scope=technical` pass reads the spec's existing business sections (when populated) BEFORE asking any technical questions; cites them (including `## Decision Context > ### Motivation` if populated) in the interview opener; references them when a technical question is gated by a business constraint. When business sections are absent, proceeds silently with technical-only questions.
- **R8**: `--scope=technical` pass writes only the technical-owned sections; leaves business-owned sections untouched.
- **R9**: `--scope=both` runs business pass first, then technical pass in same skill invocation, with business pass output passed as context to the technical pass.
- **R11**: **Canonical spec template extracted to `plugins/flow-next/templates/spec.md`.** File contains section-by-section scaffold (7 sections per the canonical-sections table above) with scope-owner HTML-comment annotations and one-line guidance per section. Frontmatter explains purpose + consumers.
- **R12**: **`docs/teams.md` naming fix.** Replace "Business spec" / "Business-layer spec" / "Full spec" labels in the handover-objects table (rows 1-2) and stage-by-stage walkthrough headers ([2]/[3]) with phrasing that makes the same-artefact-evolving model unambiguous (e.g., "Spec — business-layer complete" / "Spec — fully complete").
- **R13**: **`GLOSSARY.md` handover-object fix.** The "Handover object" entry — "flow-next defines six: business spec (#1), full spec (#2), ..." — rewritten to make handovers #1 and #2 unambiguously layers of the same spec.
- **R14**: **`docs/teams.md` "Symmetric interview" subsection + ASCII flowchart**: (a) replace `--strategy --docs` framing with `--scope` as canonical control, (b) rewrite the ASCII flowchart to use `--scope=business` / `--scope=technical` and non-misleading handover labels, (c) explicitly state that handovers are reviewable states of ONE `.flow/specs/<id>.md` file, (d) add a one-paragraph note that supplementary design docs are separate artefacts cross-linked from the spec, NOT the spec itself.
- **R15**: **`docs/teams.md` Lifecycle map mermaid flowchart updated** to annotate `/flow-next:interview` as a scoped operation. Visually communicates that one node = two scoped passes.
- **R16**: **`STRATEGY.md` "Our approach"** gets a single-sentence anchor stating the single-evolving-spec choice (explicit one-spec-per-feature, vs Kiro's split-file model). No new track required.
- **R17**: **`CLAUDE.md` "Creating a spec" guide** updated to LINK to `plugins/flow-next/templates/spec.md` rather than embed the section list. The heredoc example becomes a short pointer with one-line summary, full structure lives in the template file. Any skill (capture, interview, plan) that references the spec structure cross-links the template.
- **R18**: **`plugins/flow-next/README.md` audit + update**: any mermaid blocks or spec-structure mentions reviewed for "two specs" misread; updated to single-spec-evolving framing where applicable. Version badge bumps to 1.1.0.
- **R19**: **mickel.tech flow-next page** (`~/work/mickel.tech/app/apps/flow-next/page.tsx`) updated to surface: (a) structured business-context capture as a flow-next capability, (b) the `--scope=business|technical|both` symmetric-interview pattern. Keeps the existing spec-driven framing; adds the new affordances under it. Standalone PR against `gmickel/mickel.tech` (out-of-tree from this repo).
- **R20**: Codex mirror picks up `questions-business.md`, `questions-technical.md`, `questions-shared.md`, updated SKILL.md, and `templates/spec.md` via `sync-codex.sh`; R30 mirror guard passes; no R17/R19 vocabulary violations in the new business question bank.
- **R21**: `sync-codex.sh` validation block gains a drift guard against canonical-section duplication across **all** skill markdown files (SKILL.md, workflow.md, phases.md, steps.md, examples.md, and any other `*.md` under `plugins/flow-next/skills/*/`) — not just SKILL.md. Detection: any skill markdown file containing `^## Goal & Context` followed within 30 lines by `^## Architecture & Data Models` AND `^## API Contracts` errors out (the canonical template at `plugins/flow-next/templates/spec.md` is the only allowed location for the full sequence). Catches re-divergence wherever the duplication might re-appear.
- **R22**: **Backward-compat invariant — tested as deterministic static checks, not interactive-skill markdown diffs.** A user who never passes `--scope` experiences zero behavioral change across all flow-next surfaces. `/flow-next:interview` is interactive (calls `AskUserQuestion`) and cannot be diff-tested against a fixture without a transcript harness — instead the invariant is enforced at the rule-engine level via deterministic unit tests on observable state: (a) zero-flag scope resolution returns `technical`; (b) `questions-technical.md` is the bank loaded when `SCOPE=technical`; `questions-business.md` is NOT loaded; (c) the section-write policy under `SCOPE=technical` with empty biz sections writes ONLY tech-owned sections, writes NO placeholders, writes NO new sections that did not exist in 1.0.2 — specifically, `## Decision Context` stays FLAT (no `### Motivation` / `### Implementation Tradeoffs` H3s introduced) unless the spec already has them or a biz pass has run; (d) capture's business-routing layer, given a conversation-evidence block with zero biz-signal phrases, adds NO content to business-routing destinations (`Boundaries`, outcome-AC, `### Motivation` under `Decision Context`) and fires NO `--scope=business` suggestion. (This is narrower than "whole spec only has Goal & Context" — a conversation can legitimately contain technical requirements that capture routes elsewhere; the invariant is about the BIZ ROUTING LAYER alone.); (e) `flowctl spec create` produces a skeleton byte-for-byte identical to the 1.0.2 output.
- **R23**: Unit tests cover: scope flag parsing (all valid + conflicting forms); biz-pass-writes-biz-sections-only; tech-pass-reads-biz-first-when-present; tech-pass-proceeds-silently-when-biz-absent; both-mode-end-to-end; alias resolution (`--biz` / `--tech`); template file exists at canonical path; CLAUDE.md links to template (not duplicates it); capture routes target-user / MVP-scope / what-not-to-build signals to the right sections (R24); capture suggestion fires only when business signals exist but biz layer is sparse (R25); R22 deterministic invariants — zero-flag scope resolution returns `technical`, `questions-technical.md` is the loaded bank, section-write policy under `SCOPE=technical` writes only tech-owned sections, capture no-biz-signal evidence adds NO content to business-routing destinations (`Boundaries`, outcome-AC, `## Decision Context > ### Motivation`) AND fires no `--scope=business` suggestion, `flowctl spec create` skeleton byte-for-byte parity with 1.0.2 output; section merge contract enforced (auxiliary sections preserved, R-IDs append-only, placeholder-only replacement).
- **R24**: **Capture biz-context routing.** `/flow-next:capture` routes explicit conversation signals to the appropriate business section based on signal type, source-tagged `[user]` or `[paraphrase]`. Nine SIGNAL CATEGORIES (counting unit for R25's sparse heuristic) map onto a smaller set of markdown destinations: Routing destinations: target user / persona → `Goal & Context`; problem framing / why-now → `Goal & Context`; success metrics / definition of done → outcome-AC + `## Decision Context > ### Motivation`; MVP scope / "not doing X yet" → `Boundaries`; business constraints (regulatory, deadlines, budget) → `Goal & Context` or `## Decision Context > ### Motivation`; what-NOT-to-build / non-goals → `Boundaries`; prioritization rationale → `## Decision Context > ### Motivation`; business risks → `Goal & Context` or `## Decision Context > ### Motivation`; UX expectations → `Goal & Context`. Sections that have no conversation signal stay absent — never auto-populated from nothing.
- **R25**: **Capture next-step suggestion.** When `/flow-next:capture` produces a spec that has business-context signals but the business layer is sparse, the read-back footer includes a one-line suggestion: *"This conversation has business-requirements signals; consider `/flow-next:interview --scope=business <spec-id>` to deep-refine the business layer."* **Counting unit for "sparse"**: nine SIGNAL CATEGORIES from R24 (not markdown destinations, which collapse to ~4 distinct sections). Threshold: at least one detected signal category but fewer than three. When fewer than one — no biz signals at all — the suggestion does NOT fire (this preserves R22 — solo dev who never mentioned business context sees zero new prompts). When three or more — biz layer is reasonably filled — also does not fire. Sweet spot is "user said biz things but underspecified". Suggestion is informational only — not a blocking prompt.

- **R26**: **Business pass MUST investigate project documentation before drafting questions.** Symmetric to the technical pass's "Investigate Codebase Before Asking" rule (existing in SKILL.md:226). Before posing any biz question, the `--scope=business` pass reads — in order — `README.md`, `CHANGELOG.md` (or project-equivalent release notes), `STRATEGY.md`, `GLOSSARY.md`, `knowledge/decisions/` (most-recent N entries, where N is bounded — read TOC + first paragraph of each, not full bodies), the `.flow/specs/` index (open specs only), and `docs/` directory if present. Items answered from these sources are logged in a new `## Resolved via Project Docs` spec section (parallel to existing `## Resolved via Codebase`); the user is NOT asked about things the project docs already define. Analogous to the existing "if you find yourself answering a 'should' question via grep, that's the bug" rule — symmetric form: "if you find yourself asking the user a biz question that README/STRATEGY already answers, that's the bug." `## Resolved via Project Docs` joins the auxiliary-section enumeration in the canonical template (skill-conditional, biz-pass-only).
## Boundaries

Out of scope for fn-44:

- A second skill (e.g., `/flow-next:interview-business`). STRATEGY.md: hold the line on slash-command count.
- Splitting `.flow/specs/<id>.md` into multiple files (Kiro model). Flow-next stays single-file.
- **Supplementary design docs (`docs/design/<topic>.md`, ADRs, etc.) are NOT in scope for the "one spec" rule.** Flow-next continues to support them as cross-linked references; this spec only governs what lives inside `.flow/specs/<id>.md`.
- Templating placeholders / variable substitution in `templates/spec.md`. Static markdown scaffold; no `{{var}}` substitution.
- `flowctl spec create` automatic template seeding. The template file exists and is referenced by skills; `flowctl spec create` continues to produce the minimal skeleton it does today.
- Cross-model review at the per-scope handover (`/flow-next:plan-review --scope=business`). Possible follow-on once the scope-aware interview lands.
- Multi-stakeholder workflows (multiple POs, multiple tech leads on one spec). Stays 1:1.
- Default scope flip in a future release. fn-44 commits to `--scope=technical` as default in 1.1.0.
- AI-x-SDLC-Starter-Kit methodology guide update. Cross-link from docs/teams.md exists; the external guide gets a follow-on PR when fn-44 ships, not bundled with fn-44 itself.
- Localization. English only.

## Decision Context

- **Structured business context is load-bearing for the implementing agent.** Every implementation decision the agent makes — UX phrasing, error message tone, defensive design depth, MVP-vs-extensible trade-offs, performance vs simplicity calls, regulatory hardening — is currently a guess based on indirect signals. The business sections turn those guesses into structured input. Nine context dimensions improve every call.
- **One spec, not two** — confirmed by spec-driven-development best practice (GitHub Spec Kit, Tessl, most tooling): single durable spec evolving through layers. Kiro's split-file model is deliberately not adopted.
- **Fix the docs at the source, not just downstream interpretation** — during fn-44 scoping the noun pairs "business spec" + "full spec" in `docs/teams.md` and `GLOSSARY.md` led both an LLM and a human reader to infer "two specs". Patching only the new code while leaving the misleading prose in the canonical glossary would let the same confusion re-emerge. R12 + R13 fix the source.
- **Canonical spec template as a file, not embedded prose** — currently CLAUDE.md embeds the section list as a heredoc example. As section ownership becomes meaningful (post-fn-44 scope flag), duplication becomes drift risk. Extracting to `plugins/flow-next/templates/spec.md` gives one source of truth. R21 adds a sync-codex guard against future re-divergence.
- **Default `--scope=technical`, not `both`** — STRATEGY.md primary audience is "solo developers running multi-agent loops" and they ARE the PO; forcing them through business-layer questions they've already answered in their head is a tax. Current single-pass behavior is technical-heavy; preserving as default = zero-breakage backward compat. Teams opt into the symmetric pattern with `--scope=business` and `--scope=technical`.
- **mickel.tech update bundled in this spec** — the externally-facing flow-next story is "spec-driven, agent-first, zero-dep". Structured business-context capture is exactly that story landing on the marketing page. R19 covers it as a standalone PR against `gmickel/mickel.tech`.
- **Flow charts updated in same spec** — Lifecycle map (mermaid) + Symmetric-interview ASCII flowchart in docs/teams.md both visually reinforce the documented pattern. Without updating them, the visual model contradicts the prose and CLI behavior.
- **Flag, not new skill** — STRATEGY.md key metric: hold the line on slash-command count.
- **Two question banks, not one tagged bank** — maintenance clarity; lets each layer evolve independently; matches the spec template's section-level scope ownership.
- **Technical pass reads business sections first (when populated)** — prevents drift. Technical questions can't silently contradict business constraints if the agent has them in context.
- **Placeholder lines vs empty sections after biz pass** — explicit placeholders signal intentional emptiness after a biz pass. `/flow-next:plan` can refuse to act on a half-spec by checking for these; empty section (no header at all) is the solo-dev fresh-spec state.
- **Why ship docs + mickel.tech updates in this spec, not separately** — the docs currently say one thing (symmetric interview is the pattern), the code does another (no scope guard), the noun naming actively misleads, and the externally-facing page doesn't surface the affordance. Bundle.

## Strategy alignment

Advances STRATEGY.md "Spec-driven team patterns" track: docs/teams.md describes the symmetric interview as the canonical PO→tech-lead handover, but the skill ships without scope guards AND the canonical glossary uses misleading noun pairs AND the externally-facing page doesn't surface the affordance. Aligns with "Solo developers running multi-agent loops" primary audience by keeping `--scope=technical` as default + project-level operator authority (not per-spec) so Ralph autonomous loops never stall on mid-task escalation.

Version bump: 1.0.2 → 1.1.0 (new feature; minor bump per semver).
