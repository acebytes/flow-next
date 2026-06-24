---
name: flow-next:fleet
description: Parallel-worktree orchestrator over ready specs — inspects, classifies, gets approval, launches one short-lived worktree per dispatch unit, chains /work → /make-pr → /resolve-pr per worktree
argument-hint: "[--dry-run] [--work-only] [--max-concurrent N] [--worktree-root <path>] [--review=<backend>]"
---

# IMPORTANT: This command MUST invoke the skill `flow-next-fleet`

The ONLY purpose of this command is to call the `flow-next-fleet` skill. You MUST use that skill now.

**Arguments:** $ARGUMENTS

Pass the arguments to the skill. The skill handles inspection, classification, file-overlap wave grouping, approval interview, parallel dispatch, and the final report. Human-gated: no worktree exists and no subagent runs without explicit user approval.
