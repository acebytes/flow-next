# Eval-driven prompt optimization for flow-next skills & agents

## Conversation Evidence

> user: "olelehmann100kMRR/autoresearch-skill clone this … read it … then tell me where it would make sense to apply to the flow-next skills, i am looking for accuracy and increasing efficiency (speed/tokens)"
> user: "lets do it using this skill and whatever else makes sense, start with 1, do this on a new optimization branch"
> user: "at the end of the day i think we can probably tweak a lot of our core skills without losing accuracy"
> user: "i dont [think] this optimize skill should live inside flow-next instead we should have a detailed instruction file inside flow-next on how to do this that points to the other repo … and paths on my local machine, where we spin up such experiments"
> user: "the accuracy-critical ones, need to take into account that the user can override the spec.md etc"
> user: "may take a more complicated repo than flow-next itself, maybe /Users/gordon/work/DocIQ-Sphere or so"
> [context: external methodology = Karpathy "autoresearch" — binary evals, baseline, one mutation, keep-if-better ratchet. Cloned at ~/repos/autoresearch-skill (SKILL.md + eval-guide.md).]

## Goal & Context
<!-- scope: business -->
<!-- Source-tag breakdown: 60% [user] / 30% [paraphrase] / 10% [inferred] -->

flow-next ships ~21 agents (~24k tokens of prompt) and several very large skill prompts (`make-pr` ~31k, `audit`/`capture`/`impl-review` ~14–15k each, `interview` ~12k). The scout family (repo/context/build/testing/security/…) is dispatched in **parallel fan-out** on every `prime`/`plan`/`capture`, so both their **prompts** (input) and their **outputs** (which flow into the planner's context) are paid repeatedly. This spec adopts **eval-driven prompt optimization** (the external "autoresearch" methodology) to make the core skills/agents **leaner and/or more accurate** — measured, not by vibes — and to **prove no accuracy is lost** via a keep/revert ratchet. The methodology stays **external** (`~/repos/autoresearch-skill`); flow-next gets a **how-to doc**, not a vendored skill. Already proven on `repo-scout` (83%→100% on its eval set, ~40–50% leaner output, accuracy held). This spec is the umbrella to roll it across the core set. [user]/[paraphrase]

## Architecture & Data Models
<!-- scope: technical -->

The optimization is an **offline, experimental** loop run by the host agent, never auto-applied to `main`:

- **Per target:** an `opt/<name>` branch + an `optimization/<target>/` harness (repo root, never ships): `test-inputs.md` (3–5 frozen inputs), `evals.md` (3–6 **binary** evals), `results.tsv`, `changelog.md`, `<target>.md.baseline` (revert backup).
- **The loop:** baseline (run as-is, score) → one mutation → re-run → keep-if-score-rose-else-revert. Log every experiment.
- **Run trick:** the prompt-under-test is a **file**; dispatch a read-only `Explore` subagent told to *read that file and follow it* (NOT the registered agent — that runs the installed copy, so edits wouldn't be live). Model held constant; same inputs every round.
- **Two levers:** (1) **Output budget** — cap what the skill emits (relative paths, top-N/section, no code blocks) → downstream context savings; (2) **Prompt trim** — remove instructions/examples that don't move the score → input savings every call.
- **Promotion:** a kept mutation ships via the normal pipeline (apply to canonical → `sync-codex.sh` → version bump → CHANGELOG/docs-site → tag).
- Bridge doc: [`agent_docs/optimizing-skills.md`](agent_docs/optimizing-skills.md).

## API Contracts
<!-- scope: technical -->

No runtime API. The "contract" is the harness layout + scoring procedure in `agent_docs/optimizing-skills.md`. Binary eval shape: `EVAL N: name / Question (yes-no) / Pass / Fail`. `max_score = evals × runs`.

## Edge Cases & Constraints
<!-- scope: technical -->

- **Accuracy guarantee = ratchet + evals.** A mutation is kept only if the score (which **includes** accuracy evals) doesn't drop. The guarantee is exactly as strong as the evals → **every suite must carry ≥2–3 real accuracy evals** (grounded/coverage/correctness/format), not just a token cap.
- **Overfitting / eval-gaming:** ≤6 evals, non-gameable, varied frozen inputs (the eval-guide rules).
- **Cost:** the loop itself is token-heavy (N runs × experiments). It's an offline investment that pays back on high-frequency prompts; not worth it for once-a-release skills. Bulk sweeps are workflow/background-shaped.
- **Side-effectful skills** (`work`, `ralph`, `setup`, live `tracker-sync`/`make-pr`/`resolve-pr`) are poor loop targets (state mutation, external calls) — covered by smoke tests, not this.
- **Test bed = representative external repos, NOT flow-next-on-itself.** Repo-context skills (scouts, `plan`, code-aware `capture`, the reviews) must run against a representative app corpus — primary `~/work/DocIQ-Sphere` (~442k LOC; TS/TSX + Python + XSD schemas + Docker) + ≥1 contrasting repo for variety. flow-next is unconventional (one ~24k-LOC `flowctl.py` + markdown skills); tuning a scout against it **overfits** and never exercises multi-file app structure. flow-next-on-itself is acceptable ONLY for repo-agnostic output-FORMAT mutations (e.g. the repo-scout output-budget win), never for accuracy/coverage evals or prompt-trims.

## Acceptance Criteria
<!-- scope: both -->

- **R1:** `agent_docs/optimizing-skills.md` exists + indexed in CLAUDE.md "Where to look" — covers method, the external-tool pointer (`~/repos/autoresearch-skill`), local conventions (`opt/*` branch, `optimization/<target>/`, `~/work/slop-testbed`), the prompt-as-`Explore`-subagent run trick, the two levers, the accuracy-guard rule, and the spec.md-override caveat. **DONE on `opt/autoresearch-tier1` (pending merge).** [user]
- **R2:** `repo-scout` optimized through the loop with a retained harness + changelog: output ~40–50% leaner, accuracy held (grounded/tags/coverage). **DONE on `opt/autoresearch-tier1` (pending merge).** [paraphrase]
- **R3:** Every eval suite written for this initiative carries **≥2–3 accuracy evals** so the keep/revert ratchet cannot trade accuracy for tokens. [user]
- **R4:** Hot-path **scout family** optimized — `context-scout` (~2.8k) + the prime scouts + plan scouts — via the output-budget mutation, each with a **coverage eval** added; kept mutations applied to canonical + Codex mirror. [paraphrase]
- **R5:** **Accuracy-critical** skills (`capture`, `interview`, `plan`, `impl/plan/completion-review`) optimized with evals that treat **`spec.md` as USER-AUTHORITATIVE**: generators eval fidelity + source-tagging + read-back + **no silent overwrite of a user-edited spec**; consumers eval **grounding in the spec as written, including user edits** (a trim must not make them skim user-edited sections). Reuse `~/work/slop-testbed` for reviews. [user]/[paraphrase]
- **R6:** **Heavy prompts** (`make-pr` ~31k, `audit`, `interview`, `prospect`) trimmed via the prompt-trim lever with strong **behavioral** evals (kept output unchanged), for input-token savings every invocation. [paraphrase]
- **R7:** Each kept mutation is **promoted to ship** via the normal release pipeline (apply canonical → `sync-codex` → version bump → CHANGELOG/docs-site → tag). `opt/*` branches stay experimental until promoted; nothing auto-merges. [inferred]
- **R8:** The autoresearch tool is **NOT vendored** as a flow-next skill/command — it stays external, bridged only by `agent_docs/optimizing-skills.md`. [user]
- **R9:** Repo-context skill optimization (scouts, `plan`, code-aware `capture`, reviews) is evaluated against a **representative external repo corpus** — primary `~/work/DocIQ-Sphere` (~442k LOC, multi-stack) + ≥1 contrasting repo — **not flow-next-on-itself** for accuracy/coverage evals or prompt-trims; flow-next-on-itself is permitted only for repo-agnostic output-format mutations. The `~/work/slop-testbed` clean-vs-slop repo remains the frozen eval suite for the review skills. [user]

## Boundaries
<!-- scope: business -->

- **NOT** an `/flow-next:optimize-skill` skill or command — external tool + how-to doc only. [user]
- **NOT** a one-shot — an ongoing program; this spec is the umbrella + conventions, each target is its own `opt/*` experiment.
- Optimization runs are **offline/experimental** — never auto-applied to `main`; a mutation reaches users only through a normal release.
- Scope is **prompt quality** (input/output tokens + accuracy), **not** flowctl plumbing (smoke-test-covered).
- A token-trim must **never** reduce a skill's respect for the user-authoritative `spec.md`.
- Bulk sweeps that fan out many subagent runs are an explicit opt-in (workflow/background), not silent.

## Decision Context
<!-- scope: both — conditionally substructured -->

### Motivation
The host-agent loop's cost is dominated by repeatedly-paid prompts + scout outputs; tightening them is a standing efficiency win, and the review/capture skills' accuracy gates real quality. Eval-driven optimization makes both **measurable** and makes "tweak without losing accuracy" a **property** (the ratchet) rather than a hope.

### Implementation Tradeoffs
External-not-vendored (keeps flow-next's surface clean; the tool is a meta-dev-aid, not a user feature — same instinct as not bloating with commands/skills). Eval-driven over vibes (reproducible; the ratchet is the accuracy guarantee, contingent on real accuracy evals — hence R3). The run trick (Explore reads the prompt file) over invoking the registered agent (edits-are-live). `spec.md`-user-authoritative reshapes accuracy evals: measure **fidelity + respect-for-override**, never "is the skill's output the right spec" (R5). Bulk = background Workflow; accuracy-critical = interactive, human-in-loop on eval design.

## Strategy Alignment

Serves token efficiency + accuracy of the core agent loop without changing the architecture (better prompts, same "host agent IS the intelligence" + zero-dep model). Complements, doesn't replace, the existing static review surface (impl/plan/completion-review) — it optimizes those prompts too.

## Requirement coverage

| R-ID | Task |
|------|------|
| R1 | DONE (opt/autoresearch-tier1) — verify on merge |
| R2 | DONE (opt/autoresearch-tier1) — verify on merge |
| R3 | fn-54.M (TBD — populate via /flow-next:plan) |
| R4 | fn-54.M (TBD) |
| R5 | fn-54.M (TBD) |
| R6 | fn-54.M (TBD) |
| R7 | fn-54.M (TBD) |
| R8 | fn-54.M (TBD) |
| R9 | fn-54.M (TBD — representative-repo corpus, DocIQ-Sphere primary) |
