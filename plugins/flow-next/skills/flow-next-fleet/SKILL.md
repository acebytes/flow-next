---
name: flow-next-fleet
description: Parallel-worktree orchestrator over multiple ready specs. Inspects flow-next state, classifies each ready spec by stage (work / plan-review / spec-completion-review / make-pr), groups conflicting units into waves by file-overlap, presents a plan for human approval, then launches one short-lived worktree per unit with a fleet-runner subagent that chains the dispatched stage skill into /flow-next:make-pr and /flow-next:resolve-pr. STUB specs (empty Approach/Acceptance) and ambiguous states route to a "needs human" queue instead of auto-dispatch. Human-gated, never autonomous. Triggers on /flow-next:fleet, optionally with --dry-run, --work-only, --max-concurrent <N>, --worktree-root <path>, --review=<backend>.
user-invocable: false
allowed-tools: Read, Bash, Grep, Glob, Write, Edit, Skill, Task, AskUserQuestion
---

# /flow-next:fleet — parallel-worktree orchestrator over ready specs

A single invocation: inspect the board, classify every ready spec, group conflicting units into safe waves, ask for approval, launch each unit in its own short-lived worktree with a fleet-runner subagent that chains `/flow-next:work` (or another stage skill) into `/flow-next:make-pr` and `/flow-next:resolve-pr`. STUB specs and ambiguous states never auto-dispatch — they surface in a "needs human" queue at the end.

Fleet is the **human-gated parallel** counterpart to `/flow-next:pilot`. Pilot is a single-tick autonomous conductor (one spec, one stage, driven by `/loop` or `/goal`); fleet fans the same stage skills out across many specs at once, behind an explicit approval gate. They are alternative drivers and **never nest** — fleet refuses to run inside pilot or Ralph.

Per CLAUDE.md, the per-worktree chain stops at `/flow-next:resolve-pr`. **Fleet does not merge** — `/flow-next:land` is the sole skill allowed to call `gh pr merge`, and it stays in the user's hands. Each worktree's branch is left on origin with its PR open + reviews resolved; the user clicks merge.

## Preamble

**CRITICAL: flowctl is BUNDLED — NOT installed globally.** `which flowctl` will fail (expected). Define once; subsequent blocks (here and in `workflow.md`) use `$FLOWCTL`. Subagents that run in fresh context fall back to the repo-local copy:

```bash
FLOWCTL="${DROID_PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/scripts/flowctl"
[ -x "$FLOWCTL" ] || FLOWCTL=".flow/bin/flowctl"
WORKTREE_KIT="${DROID_PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/skills/flow-next-worktree-kit/scripts/worktree.sh"
```

## Pre-check: local setup version

One-line nag when the local setup lags the plugin, same pattern as `/flow-next:plan` and `/flow-next:pilot`:

```bash
SETUP_VER=$(jq -r '.setup_version // empty' .flow/meta.json 2>/dev/null)
PLUGIN_JSON="${DROID_PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/.claude-plugin/plugin.json"
PLUGIN_VER=$(jq -r '.version' "$PLUGIN_JSON" 2>/dev/null || echo "unknown")
if [[ -n "$SETUP_VER" && "$PLUGIN_VER" != "unknown" && "$SETUP_VER" != "$PLUGIN_VER" ]]; then
  echo "Plugin updated to v${PLUGIN_VER}. Run /flow-next:setup to refresh local scripts (current: v${SETUP_VER})." >&2
fi
```

Continue regardless.

## Hard guards (before anything else)

```bash
if [[ -n "${FLOW_RALPH:-}" || -n "${REVIEW_RECEIPT_PATH:-}" || -n "${PILOT_DRY_RUN:-}" ]]; then
  echo "Fleet, pilot, and Ralph are alternative drivers — never nest them." >&2
  exit 1
fi

REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
if git -C "$REPO_ROOT" status --porcelain | grep -v '^.. \.flow/' >/dev/null; then
  echo "Refusing: dirty non-.flow working tree in the main worktree. Fleet runs against a clean base." >&2
  git -C "$REPO_ROOT" status --porcelain | grep -v '^.. \.flow/' >&2
  exit 1
fi
```

Fleet runs in the main worktree as orchestrator; each dispatched unit runs in its own short-lived worktree under `<worktree-root>/<slug>`. The main worktree stays on its current branch (typically `main`) throughout the fleet's lifecycle. Worktrees inherit from `$FLEET_WORKTREE_ROOT` (default `.worktrees/`), which is configurable per-invocation via `--worktree-root`.

## Mode detection

Parse `$ARGUMENTS` for flags. Unknown flags warn to stderr and are ignored. Default is full polymorphic dispatch with the platform's default review backend.

The loop handles both `--flag=value` and space-separated `--flag value` via a `PREV` token holder (avoiding bash positional-parameter parsing, which the host's argument interpolation corrupts inside skill code blocks — same pattern as `/flow-next:pilot`):

```bash
RAW_ARGS="$ARGUMENTS"
FLEET_DRY_RUN=0
FLEET_WORK_ONLY=0
FLEET_MAX_CONCURRENT=""
FLEET_WORKTREE_ROOT=".worktrees"
FLEET_REVIEW=""

PREV=""
for ARG in $RAW_ARGS; do
  case "$PREV" in
    --max-concurrent) FLEET_MAX_CONCURRENT="$ARG"; PREV=""; continue ;;
    --worktree-root)  FLEET_WORKTREE_ROOT="$ARG";  PREV=""; continue ;;
    --review)         FLEET_REVIEW="$ARG";          PREV=""; continue ;;
  esac
  case "$ARG" in
    --max-concurrent|--worktree-root|--review) PREV="$ARG" ;;
    --max-concurrent=*) FLEET_MAX_CONCURRENT="${ARG#--max-concurrent=}" ;;
    --worktree-root=*)  FLEET_WORKTREE_ROOT="${ARG#--worktree-root=}" ;;
    --review=*)         FLEET_REVIEW="${ARG#--review=}" ;;
    --dry-run)   FLEET_DRY_RUN=1 ;;
    --work-only) FLEET_WORK_ONLY=1 ;;
    -*) echo "Unknown flag: $ARG (ignored by /flow-next:fleet)" >&2 ;;
    *)  echo "Unknown argument: $ARG (ignored by /flow-next:fleet)" >&2 ;;
  esac
done
[[ -n "$PREV" ]] && echo "Flag $PREV given without a value (ignored by /flow-next:fleet)" >&2

# Default concurrency: min(cpu - 2, 4) — bounded by Workflow tool's default and conservative even on small machines.
if [[ -z "$FLEET_MAX_CONCURRENT" ]]; then
  CPU="$(getconf _NPROCESSORS_ONLN 2>/dev/null || echo 4)"
  FLEET_MAX_CONCURRENT=$(( CPU - 2 < 4 ? CPU - 2 : 4 ))
  [[ "$FLEET_MAX_CONCURRENT" -lt 1 ]] && FLEET_MAX_CONCURRENT=1
fi

export FLEET_DRY_RUN FLEET_WORK_ONLY FLEET_MAX_CONCURRENT FLEET_WORKTREE_ROOT FLEET_REVIEW
```

## The classifier (read this before the workflow)

Fleet's per-spec classifier reuses `/flow-next:pilot`'s table verbatim, with **one addition** (STUB detection) that pre-empts the `plan` row. First match wins:

| Condition | Classification | Per-spec action |
|---|---|---|
| spec body contains `**Status: STUB.**` marker OR Approach + Acceptance sections both empty | `needs-interview` | **Notification queue** (no worktree; surfaces in final report) |
| 0 tasks AND spec body is filled | `plan` | Auto-dispatch `/flow-next:plan` per worktree (skipped under `--work-only`) |
| tasks exist AND `plan_review_status != "ship"` AND review backend configured | `plan-review` | Auto-dispatch `/flow-next:plan-review` (skipped under `--work-only`) |
| any task is `todo` or `blocked` | `work` | Auto-dispatch `/flow-next:work` — ONE worktree per unblocked task from `flowctl ready --spec <id>` |
| in-progress-only stale claim | `needs-human` | Notification queue (work's ready-driven loop cannot resume an in-progress claim) |
| all tasks done AND `completion_review_status != "ship"` AND review backend configured | `spec-completion-review` | Auto-dispatch `/flow-next:work` which routes through `/flow-next:spec-completion-review` per the work skill's Phase 3g contract |
| all tasks done AND completion ship-or-ungated AND no PR exists | `make-pr` | Auto-dispatch the per-worktree chain starting at `/flow-next:make-pr` |
| all tasks done AND completion ship AND open PR exists | `defer-to-land` | Notification queue with the open-PR URL — `/flow-next:land` is the user's call, not fleet's |
| all tasks done AND completion ship AND closed-unmerged OR merged-but-spec-still-open | `needs-human` | Notification queue with the inconsistency description |

The STUB detection rule diverges from pilot's behavior: pilot would dispatch `/plan` against an empty spec on the assumption that the Overview alone is enough; fleet is more conservative and routes STUBs to a human-interview queue, because under polymorphic parallel dispatch a bad auto-plan would spawn many wasteful task specs that compound at PR time. The `**Status: STUB.**` marker is the convention introduced by the fn-10 / fn-11 sibling specs.

## Forbidden

- **Calling `gh pr merge` anywhere in the dispatch path.** `/flow-next:land` is the sole skill licensed to merge, and fleet does not invoke it. Per-worktree chains stop at `/flow-next:resolve-pr`.
- **Dispatching skills outside the stage set `{plan, plan-review, work, spec-completion-review, make-pr, resolve-pr}`.** Capture, interview, QA, prospect, audit, sync, prime, and land are never fleet stages. Interview surfaces as a notification, never a dispatch.
- **Nesting inside pilot or Ralph.** Both are alternative autonomous drivers; fleet is the parallel human-gated counterpart. Refuse via the hard guard at the top.
- **Running with a dirty main worktree.** Fleet runs from the main worktree as orchestrator; dispatched units run in their own worktrees. The orchestrator's branch must stay clean for the duration so the manifest write doesn't race a user edit.
- **Auto-dispatching STUB specs to `/flow-next:plan`.** Empty Approach/Acceptance sections need a human interview first; fleet routes them to the notification queue and stops.
- **Launching before approval.** The `AskUserQuestion` approval gate in Phase 6 is mandatory — fleet never creates a worktree or dispatches a subagent before the user explicitly approves the plan.
- **Asking questions during dispatch.** All questions happen in Phase 3 (interview, conditional) and Phase 6 (approval). After approval, the orchestrator runs to completion without further prompts — subagents in worktrees never ask either; they report `needs-human` to the manifest.

## Workflow

Execute [workflow.md](workflow.md) in order:

1. **inspect** — enumerate all open specs, fetch full spec + task JSON, identify ready candidates.
2. **classify** — apply the classifier table above; produce a typed candidate list.
3. **triage** — split into auto-dispatch units vs. notification-queue items.
4. **detect file overlap** — build wave groups (same algorithm as `/flow-next:resolve-pr` Phase 5).
5. **interview (conditional)** — ask the user only if something is genuinely ambiguous (overlap, scope, concurrency cap higher than default). Skip when unambiguous.
6. **approval** — present the wave plan via `AskUserQuestion`. No launches without explicit user approval.
7. **launch** — per wave, create `<worktree-root>/<slug>` via worktree-kit, dispatch one `fleet-runner` subagent per unit with `Task`. Multiple subagents in the same wave run in parallel via a single message of `Task` calls.
8. **monitor** — wait for subagents to return. Each subagent's prompt instructs it to run the chain inside its worktree, write its result to the manifest, and return a structured summary.
9. **report** — print the final summary table grouped by outcome: PR opened (with URL), failed-needs-triage, no-op, notification-queue (needs human interview / land / triage).

## Manifest

Fleet maintains a manifest at `$(git rev-parse --git-common-dir)/flow-next/fleet-manifest.json` (alongside pilot's strikes ledger). The git-common-dir is shared across worktrees and not swept into commits by `git add -A`.

Schema:

```jsonc
{
  "launched_at": "2026-06-23T...Z",
  "main_worktree": "<repo-root>",
  "config": {
    "max_concurrent": 4,
    "worktree_root": ".worktrees",
    "review": "codex",
    "work_only": false
  },
  "units": [
    {
      "slug": "fn-5-…sched.7-work",
      "skill": "flow-next:work",
      "target_kind": "task",       // task | spec
      "target": "fn-5-…sched.7",
      "spec": "fn-5-…sched",
      "files": ["src/openctl/engine/queue.py", "..."],
      "wave": 1,
      "branch": "fleet/fn-5-…sched.7",
      "worktree": ".worktrees/fn-5-…sched.7-work",
      "agent_id": "a1b2c3...",
      "status": "running | done | failed | needs-human | stalled",
      "pr_number": 9,              // set by make-pr step
      "pr_url": "https://...",
      "outcome_reason": "<one line>"
    }
  ],
  "notifications": [
    {"spec": "fn-10-…gates", "kind": "needs-interview",
     "reason": "STUB spec — Approach + Acceptance sections empty",
     "suggested_command": "/flow-next:interview fn-10-…gates"}
  ]
}
```

The manifest is the orchestrator's source of truth across phases. Subagents write their own unit-row outcome via `flowctl` (no plumbing yet) or a `jq` atomic write under the manifest lock (file-lock via `flock` on the manifest path). On reentry, the orchestrator can re-read the manifest to surface ongoing runs to the user.

## Cleanup

The orchestrator does NOT auto-remove successful worktrees by default. PRs stay open after `/flow-next:resolve-pr` returns; the user merges when ready, then can run `<worktree-kit-path> cleanup` (or the bundled `scripts/fleet-cleanup` helper) to prune merged-PR worktrees in one pass. This mirrors the safe-by-default posture of every existing skill: nothing destructive happens without explicit user action.

A future invocation of `/flow-next:fleet` reads the existing manifest first — if there are still-running or merged-but-not-cleaned units, it surfaces them for the user before classifying fresh work. (Fleet does NOT auto-resume a stalled run; resumes are an explicit `--resume` invocation, deferred to a follow-up.)

## Unattended runs — not supported in v1

Fleet is human-gated by design. Phase 6 always blocks on `AskUserQuestion`. There is no `--auto` flag, no autonomous-mode environment variable, no driver-loop integration. For unattended autonomous parallel work, use `/flow-next:pilot` driven by `/loop` or `/goal` (single-spec, single-stage per tick); fleet is the explicit "I want to see and approve the whole batch" alternative.
