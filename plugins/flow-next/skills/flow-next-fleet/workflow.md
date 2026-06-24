# /flow-next:fleet workflow

Execute these phases in order. Each phase gates on the prior one. Stop on error and surface to user — never plow through with bad state.

## Preamble

```bash
FLOWCTL="${DROID_PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/scripts/flowctl"
[ -x "$FLOWCTL" ] || FLOWCTL=".flow/bin/flowctl"
WORKTREE_KIT="${DROID_PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/skills/flow-next-worktree-kit/scripts/worktree.sh"
REPO_ROOT="$(git rev-parse --show-toplevel)"
TODAY="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
MANIFEST_DIR="$(git rev-parse --git-common-dir)/flow-next"
MANIFEST="$MANIFEST_DIR/fleet-manifest.json"
```

`jq`, `git`, `gh`, and `getconf` must be on PATH. `FLEET_DRY_RUN`, `FLEET_WORK_ONLY`, `FLEET_MAX_CONCURRENT`, `FLEET_WORKTREE_ROOT`, and `FLEET_REVIEW` come from SKILL.md Mode Detection.

## Phase 0 — Guards (already-running fleet check)

Hard guards (Ralph/pilot nesting, dirty tree) already ran in SKILL.md. One additional guard here: refuse to start a fresh fleet if the manifest reports running units.

```bash
if [[ -s "$MANIFEST" ]]; then
  RUNNING_COUNT=$(jq -r '[.units[] | select(.status == "running")] | length' "$MANIFEST" 2>/dev/null || echo 0)
  if [[ "$RUNNING_COUNT" -gt 0 ]]; then
    echo "Existing fleet manifest has $RUNNING_COUNT running unit(s):"
    jq -r '.units[] | select(.status == "running") | "  - \(.slug) → \(.skill) on \(.target) (agent \(.agent_id // "?"))"' "$MANIFEST"
    echo ""
    echo "Resolve or wait for those before launching a new fleet."
    echo "To force a fresh run, clear the manifest manually: rm $MANIFEST"
    exit 1
  fi
fi
```

If `$FLEET_DRY_RUN` is set, the manifest is never written through the whole workflow — Phase 9 prints what would have happened.

## Phase 1 — INSPECT

Enumerate all open specs and pull full state for each.

```bash
SPECS_JSON="$($FLOWCTL specs --json)"
OPEN_SPECS="$(jq -r '.specs[] | select(.status == "open") | .id' <<<"$SPECS_JSON")"

# Echo the gross counts up-front so the user sees what fleet is operating on.
TOTAL=$(jq -r '.specs | length' <<<"$SPECS_JSON")
OPEN=$(echo "$OPEN_SPECS" | grep -c . || echo 0)
echo "Inspect: $TOTAL specs total, $OPEN open."
```

For each open spec, fetch the full payload that the classifier needs:

```bash
declare -A SPEC_FULL TASKS_FULL READY_TASKS
for ID in $OPEN_SPECS; do
  SPEC_FULL[$ID]="$($FLOWCTL show "$ID" --json)"
  TASKS_FULL[$ID]="$($FLOWCTL tasks --spec "$ID" --json)"
  READY_TASKS[$ID]="$($FLOWCTL ready --spec "$ID" --json 2>/dev/null || echo '{"tasks":[]}')"
done
```

Resolve the review backend once (used by classification rows that gate on review-configured):

```bash
if [[ -n "$FLEET_REVIEW" ]]; then
  REVIEW_BACKEND="$FLEET_REVIEW"
else
  REVIEW_BACKEND="$($FLOWCTL review-backend 2>/dev/null || echo none)"
fi
case "$REVIEW_BACKEND" in
  none|ASK|"") REVIEW_CONFIGURED=0 ;;
  *) REVIEW_CONFIGURED=1 ;;
esac
```

## Phase 2 — CLASSIFY

Apply the classifier table (see SKILL.md) per spec. First match wins. Produce one record per dispatch unit.

For each spec, derive:

- `SPEC_BODY`: read via `$FLOWCTL cat <id>` (markdown body of the spec file).
- `TASK_COUNT`, `DONE_COUNT`, `TODO_COUNT`, `BLOCKED_COUNT`, `IN_PROGRESS_COUNT`: aggregated from `TASKS_FULL[$ID]`.
- `PLAN_REVIEW`, `COMPLETION_REVIEW`, `DEPENDS_ON`: read from `SPEC_FULL[$ID]`.
- `STUB_MARKER`: true iff `SPEC_BODY` contains `**Status: STUB.**` (case-sensitive); OR Approach + Acceptance sections both empty.

The STUB check via shell:

```bash
is_stub() {
  local body="$1"
  if grep -qF '**Status: STUB.**' <<<"$body"; then return 0; fi
  # Heuristic 2: extract '## Approach' / '## Acceptance' sections and check non-empty (ignoring whitespace and headers).
  approach=$(awk '/^## Approach/,/^## /' <<<"$body" | sed '1d;$d' | grep -v '^$' | head -1)
  acceptance=$(awk '/^## Acceptance/,/^## /' <<<"$body" | sed '1d;$d' | grep -v '^$' | head -1)
  [[ -z "$approach" && -z "$acceptance" ]]
}
```

Classifier walks the table in order. The first match emits a unit record:

```jsonc
// work-ready unit (one per unblocked task in flowctl ready --spec)
{ "kind": "auto", "slug": "<task-id>-work", "skill": "flow-next:work",
  "target_kind": "task", "target": "<task-id>", "spec": "<spec-id>",
  "files": [],   // populated in Phase 4
  "branch": "fleet/<task-id>-work" }

// plan-review unit (one per spec)
{ "kind": "auto", "slug": "<spec-id>-plan-review", "skill": "flow-next:plan-review",
  "target_kind": "spec", "target": "<spec-id>", "spec": "<spec-id>",
  "files": [],
  "branch": "fleet/<spec-id>-plan-review" }

// notification (no worktree)
{ "kind": "notify", "spec": "<spec-id>",
  "notification_kind": "needs-interview|needs-human|defer-to-land",
  "reason": "<one line>",
  "suggested_command": "/flow-next:interview <spec-id>" }
```

Under `--work-only`, skip the `plan`, `plan-review`, `spec-completion-review`, and `make-pr` rows of the classifier — only emit `work` units.

Echo a compact summary so the user (and dry-run) can see the classification before any dispatch:

```text
Classify: 12 specs open
  fn-5-…sched      → work (2 tasks ready: .7, .8)
  fn-6-…polish     → plan-review (plan_review_status=unknown)
  fn-9-…spine      → plan-review (plan_review_status=needs_work)
  fn-10-…gates     → needs-interview (STUB marker)
  fn-11-…spine     → needs-interview (STUB marker)
  fn-4-…review     → spec-completion-review (all done, completion=ship — close pending)
  …
```

For each spec the classifier skipped (already done, all-blocked-pending-deps, etc.), echo one line with the skip reason.

If `FLEET_DRY_RUN` is set, stop after this phase — print the classification table and exit 0. The manifest is never written.

## Phase 3 — TRIAGE (split auto vs notify)

```jsonc
AUTO_UNITS = [ kind == "auto" ]
NOTIFICATIONS = [ kind == "notify" ]
```

If `AUTO_UNITS` is empty AND `NOTIFICATIONS` is empty → no work to do. Stop with:

```text
Fleet: nothing to dispatch and nothing to surface. All open specs are either done, awaiting other-actor work, or in defer-to-land. Try /flow-next:land for open PRs, or update STRATEGY.md for what to plan next.
```

If `AUTO_UNITS` is empty but there are notifications → skip to Phase 9 with the notification queue.

## Phase 4 — DETECT FILE OVERLAP (wave grouping)

For each `auto` unit, derive an expected file set:

- **work units** — parse the task `.md` spec for an `## Affected as-built code` section (convention from fn-5.7 / fn-5.8 spec rewrites). Fall back to `src/openctl/` (or repo-specific src dir) plus the task's own `.flow/tasks/<id>.{md,json}` if the section is missing.
- **plan-review / spec-completion-review units** — `.flow/{specs,tasks,epics}/<spec-id>*`.
- **plan units** — `.flow/{specs,tasks,epics}/<spec-id>*`.
- **make-pr units** — empty (read-only against the branch; no file edits).

The parser for `## Affected as-built code` (shell):

```bash
parse_affected_files() {
  local task_md="$1"
  awk '
    /^## Affected as-built code/ { in_section=1; next }
    /^## / && in_section { in_section=0 }
    in_section && /^- `/ { gsub(/^- `/, ""); gsub(/`.*$/, ""); print }
  ' "$task_md"
}
```

Wave grouping algorithm (same as `/flow-next:resolve-pr` Phase 5):

```text
waves = []
for unit in AUTO_UNITS (stable order by classifier-emission):
  placed = False
  for wave in waves:
    if all(unit.files ∩ other.files == ∅ for other in wave) and len(wave) < FLEET_MAX_CONCURRENT:
      wave.add(unit); placed = True; break
  if not placed:
    waves.append([unit])
```

Echo the wave plan:

```text
Waves:
  1 (3 parallel): fn-5-…sched.7-work, fn-6-…polish-plan-review, fn-9-…spine-plan-review
  2 (1 sequential after wave 1): fn-5-…sched.8-work
```

## Phase 5 — INTERVIEW (conditional)

Ask the user ONLY if one of these conditions holds:

1. **Wave count > 1 OR any wave size > 4** — concurrency is non-trivial; confirm the cap is OK.
2. **`needs-human` notifications include "stale in-progress claim"** — ambiguity about who should resume.
3. **A spec is classified `defer-to-land`** — surface the open PR URL; offer to launch `/flow-next:land` (NOT inside fleet; the user runs it after).

Use `AskUserQuestion` per condition. Skip the interview entirely when nothing is ambiguous. Default behavior: proceed.

Example question shapes (canonical Claude-native tool name):

```text
AskUserQuestion(questions=[{
  "question": "Concurrency: max 4 units per wave; total 7 units across 2 waves. OK?",
  "header": "Concurrency",
  "options": [
    {"label": "OK, proceed", "description": "Launch waves as planned (4 + 3)."},
    {"label": "Cap at 2", "description": "Re-group into 4 waves of 2."},
    {"label": "Run sequentially", "description": "One unit at a time, in classifier order."}
  ]
}])
```

(`sync-codex.sh` rewrites `AskUserQuestion` into the Codex mirror's plain-text numbered-prompt format.)

Apply any user decisions before the approval gate.

## Phase 6 — APPROVAL GATE (mandatory)

Present the full wave plan via `AskUserQuestion`. This is non-skippable. No worktree exists yet; no subagent has been spawned.

```text
AskUserQuestion(questions=[{
  "question": "Launch fleet? Wave 1 (3 parallel): fn-5-…sched.7-work, fn-6-…polish-plan-review, fn-9-…spine-plan-review. Wave 2 (1, after wave 1): fn-5-…sched.8-work. Notifications: 2 STUB specs (fn-10, fn-11) and 1 defer-to-land (fn-4 open-PR).",
  "header": "Approve fleet",
  "options": [
    {"label": "Launch all waves", "description": "Spawn worktrees + dispatch all auto units now. Notification queue surfaces at the end."},
    {"label": "Launch wave 1 only", "description": "Spawn wave 1 only. Re-invoke /flow-next:fleet for wave 2 after wave 1 completes."},
    {"label": "Notifications only, no auto dispatch", "description": "Skip worktree creation. Just print the needs-human queue and exit."},
    {"label": "Cancel", "description": "Do nothing."}
  ]
}])
```

On Cancel, exit 0 with "Fleet aborted before launch." On Notifications-only, jump to Phase 9.

Write the initial manifest atomically at this point (the first manifest write):

```bash
mkdir -p "$MANIFEST_DIR"
TMP="$MANIFEST.tmp.$$"
# Compose the initial manifest with units (status=pending) + notifications.
# (jq one-liner composing from in-shell variables; status="pending" until Phase 7 launches it.)
jq -n '<composed payload>' > "$TMP" && mv "$TMP" "$MANIFEST"
```

## Phase 7 — LAUNCH (per wave)

For each wave, in order:

1. **Create one worktree per unit** via `worktree-kit`:

   ```bash
   for UNIT in "$WAVE_UNITS"; do
     SLUG=$(jq -r .slug <<<"$UNIT")
     BRANCH=$(jq -r .branch <<<"$UNIT")
     bash "$WORKTREE_KIT" create "$SLUG" main
     # worktree-kit places under .worktrees/ by default; if FLEET_WORKTREE_ROOT differs,
     # adjust the worktree-kit invocation accordingly (v1: kit uses .worktrees/; non-default root
     # routes through git worktree add manually).
   done
   ```

   The branch name is `fleet/<slug>` to keep fleet's branches distinguishable from human-created branches and from worktree-kit's auto-naming.

2. **Dispatch fleet-runner subagents in parallel.** Send a single message containing one `Task` call per unit in the wave:

   ```text
   for UNIT in WAVE_UNITS:
     Task(
       subagent_type="fleet-runner",
       description="<slug>",
       prompt="""
         WORKTREE_PATH=<absolute path to .worktrees/<slug>>
         BRANCH=fleet/<slug>
         SKILL=<flow-next:work | flow-next:plan-review | ...>
         TARGET_KIND=<task|spec>
         TARGET=<task-id or spec-id>
         SPEC=<parent spec-id>
         REVIEW_BACKEND=$REVIEW_BACKEND
         MANIFEST=<absolute path>
         SLUG=<slug>

         <runbook from agents/fleet-runner.md is the agent's system prompt;
         this user-message just hands it the configuration.>
       """
     )
   ```

3. **Wait for the wave to complete** before launching the next wave. Each `Task` call returns with a structured summary; the orchestrator collects them all before advancing.

4. **Update the manifest** after each wave: write each unit's `agent_id`, `status`, `pr_number`, `pr_url`, and `outcome_reason` from the subagent's return value.

## Phase 8 — MONITOR

After each `Task` returns, parse its result. Expected shape (the fleet-runner agent emits a final JSON block; see `agents/fleet-runner.md`):

```jsonc
{
  "slug": "<slug>",
  "outcome": "done | failed | needs-human | stalled",
  "branch": "fleet/<slug>",
  "pr_number": 9,
  "pr_url": "https://github.com/.../pull/9",
  "primary_skill_result": "advanced | no-advance | crash",
  "make_pr_result": "ok | skipped | failed",
  "resolve_pr_result": "clean | needs-human | failed",
  "summary": "<one-line description>"
}
```

The orchestrator does NOT abort siblings when one unit fails — each wave's units are independent. Failures are recorded in the manifest with `status=failed` and surface in the Phase 9 report.

If a subagent does not return (the `Task` call errors out), record `status=stalled` with the error message.

## Phase 9 — REPORT

Final output: a single human-readable summary grouped by outcome. Format:

```text
Fleet results
=============

PR opened + review-clean (N):
  ✓ fn-5-…sched.7-work  → PR #9 (resolve-pr converged in 0 rounds)
  ✓ fn-6-…polish-plan-review → PR #10 (plan-review verdict SHIP, no findings)

Failed (M) — see worktree for triage:
  ✗ fn-9-…spine-plan-review → /flow-next:plan-review crashed at codex round 3 (manifest: .worktrees/fn-9-…spine-plan-review/)

Stalled (K):
  ⏸ fn-5-…sched.8-work → subagent did not return (rate-limit?). Worktree preserved at .worktrees/fn-5-…sched.8-work.

Needs your attention (P):
  • fn-10-…gates STUB — run /flow-next:interview fn-10-…gates to flesh out Approach + Acceptance.
  • fn-11-…spine STUB — same.
  • fn-4-…review defer-to-land — open PR https://...; run /flow-next:land when ready.

Next:
  - Merge clean PRs above when CI is green.
  - Triage failed/stalled worktrees: cd into .worktrees/<slug>/ to inspect.
  - Cleanup merged-PR worktrees: bash <worktree-kit-path> cleanup
```

The manifest at `$MANIFEST` is the durable record of the run. The user can re-read it at any time, and a follow-up `/flow-next:fleet` invocation surfaces any non-terminal units from it.

## Error handling

- **gh rate limit (HTTP 429)** during manifest writes: sleep 10s + retry once. On second failure, surface to user and exit non-zero.
- **worktree-kit refuses to create** (already exists, symlink in path): record the unit as `status=failed` with the kit's stderr; continue with the rest of the wave.
- **Task tool error** (subagent crash, unable to start): record `status=stalled` with the error; continue with siblings.
- **gh missing or unauthenticated** in the orchestrator phase: stop with a clear error before Phase 7 — fleet cannot land worktrees without `gh`. (The runner agent re-checks too; this is defense-in-depth.)

## Safety invariants

- **Never `gh pr merge`** anywhere — neither orchestrator nor runner. Land is the user's call.
- **Never `git push --force`** or any history-rewriting op. Worktree commits are append-only.
- **Never delete a worktree** that the user has not explicitly cleaned up. The orchestrator's `report` phase surfaces the worktree paths; cleanup is opt-in via `worktree-kit cleanup`.
- **Never auto-resume** a stalled or failed unit. The user re-invokes `/flow-next:fleet` with the unit explicitly named (deferred to a follow-up `--resume <slug>` feature).
