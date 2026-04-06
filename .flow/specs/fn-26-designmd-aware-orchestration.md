# DESIGN.md-aware orchestration

## Overview

When a project has a DESIGN.md file (Google Stitch format), flow-next should detect it and conditionally inject design context at the right points in the pipeline: planning, implementation, readiness assessment, and quality audit.

DESIGN.md is an agent-readable markdown file encoding a project's design system — colors, typography, components, elevation, layout. Introduced by Google Stitch (March 2026), adopted widely (18.6k stars on awesome-design-md). It's the design-side equivalent of AGENTS.md.

**Core principle: "if present, use it."** Never required. Never generated. Never auto-modified. DESIGN.md is upstream design intent — humans/Stitch remain source of truth.

## Constraints (CRITICAL)

- All changes are to .md files in `plugins/flow-next/skills/` and `plugins/flow-next/agents/` only
- Zero flowctl CLI changes
- Zero .flow/ JSON schema changes
- Zero hook changes
- Ralph mode unaffected
- MergeFoundry-compatible (they'll layer deeper runtime validation on top)
- No new agent files — extend existing agents with conditional behavior
- Backend tasks must not pay prompt tax — design injection is frontend-only

## DESIGN.md format (canonical)

Official Google Stitch format (6 sections):

```markdown
## Overview
A calm, professional interface for a healthcare platform.

## Colors
- **Primary** (#2665fd): CTAs, active states, key interactive elements
- **Secondary** (#6074b9): Supporting actions, chips, toggle states

## Typography
- **Headline Font**: Inter
- **Body Font**: Inter

## Elevation
No shadows. Depth via border contrast and surface color variation.

## Components
- **Buttons**: Rounded (8px), primary uses brand blue fill
- **Inputs**: 1px border, surface-variant background

## Do's and Don'ts
- Do use primary color only for the single most important action per screen
- Don't mix rounded and sharp corners in the same view
```

Community extensions (awesome-design-md) add: Layout Principles, Depth & Elevation, Responsive Behavior, Agent Prompt Guide.

**Detection:** `DESIGN.md` at repo root (primary) or `.stitch/DESIGN.md`. Case-insensitive. Validate by checking for 3+ of these section headings (fuzzy match): Overview, Colors/Color Palette, Typography, Elevation/Depth, Components/Component Stylings, Do's and Don'ts, Layout.

**Not a DESIGN.md:** Files named DESIGN.md without hex color codes (#hex pattern) — these are likely architecture design docs common in Go/Rust repos.

## Approach

Five integration points, all prompt-only:

### 1. Scouts detect DESIGN.md during planning
repo-scout adds DESIGN.md to its project docs scan list. If found and valid, reports design tokens summary in findings. Plan skill checks scout output and writes `## Design context` in frontend task specs.

### 2. Worker reads design context in Phase 1.5
If task spec has `## Design context`, worker reads referenced DESIGN.md sections before implementing. Fits naturally into existing investigation targets pattern. Backend tasks without design context: no change.

### 3. Prime checks for DESIGN.md in readiness assessment
New optional criterion DC7 in Pillar 4 (Documentation): "DESIGN.md exists with color + typography + component sections." Informational — doesn't affect score for backend-only projects. docs-gap-scout adds DESIGN.md to its scan.

### 4. Quality-auditor checks design token conformance
New conditional Section 8 (after Performance Red Flags): if DESIGN.md exists and diff contains frontend files, check for hard-coded colors/spacing vs design tokens. Advisory only.

### 5. Flow-gap-analyst adds design alignment check
New conditional Section 6: if DESIGN.md exists, check if specified components exist for the feature, if color/spacing tokens cover the use case, if breakpoints are defined.

## Frontend task detection heuristic

A task is "frontend" if its `## Description` or `**Files:**` field references:
- File extensions: `.jsx`, `.tsx`, `.vue`, `.svelte`, `.css`, `.scss`
- Directories: `components/`, `pages/`, `views/`, `layouts/`, `styles/`, `app/` (Next.js)
- Keywords: button, modal, form, layout, responsive, color, font, card, navigation, theme, UI, component

Backend tasks (`.py`, `.go`, `api/`, `server/`, `controllers/`): no design injection.

## File change map

| File | Changes |
|------|---------|
| `agents/repo-scout.md` | Add DESIGN.md to project docs scan, report design tokens if found |
| `skills/flow-next-plan/steps.md` | Conditional `## Design context` in task spec template for frontend tasks |
| `agents/worker.md` | Read DESIGN.md in Phase 1.5 when design context present |
| `skills/flow-next-prime/pillars.md` | DC7: DESIGN.md criterion (informational) |
| `agents/docs-gap-scout.md` | Add DESIGN.md to doc location scan |
| `agents/quality-auditor.md` | Conditional Section 8: design token conformance (advisory) |
| `agents/flow-gap-analyst.md` | Conditional Section 6: design system alignment |
| `plugins/flow-next/README.md` | Document DESIGN.md awareness in Features section |
| `CHANGELOG.md` | Version entry |

## Quick commands

```bash
# Verify no schema/hook changes
git diff --name-only | grep -v -E '(skills/|agents/|codex/|\.codex-plugin|\.claude-plugin|CHANGELOG|README)' && echo "UNEXPECTED FILES" || echo "OK"

# Smoke test
plugins/flow-next/scripts/smoke_test.sh

# Sync codex
scripts/sync-codex.sh
```

## Acceptance

- [ ] repo-scout detects DESIGN.md and reports design tokens in findings
- [ ] Plan skill writes `## Design context` in task specs for frontend tasks when DESIGN.md exists
- [ ] Plan skill does NOT write design context for backend-only tasks
- [ ] Worker reads DESIGN.md sections in Phase 1.5 when `## Design context` present
- [ ] Worker skips design reading when no design context in spec
- [ ] Prime Pillar 4 has DC7 criterion for DESIGN.md (informational)
- [ ] docs-gap-scout scans for DESIGN.md
- [ ] Quality-auditor conditionally checks design token conformance (advisory)
- [ ] Flow-gap-analyst conditionally checks design system alignment
- [ ] Frontend detection heuristic documented and applied consistently
- [ ] Zero changes to flowctl, .flow/ JSON, hooks
- [ ] Ralph mode unaffected
- [ ] `scripts/sync-codex.sh` runs clean
- [ ] `plugins/flow-next/scripts/smoke_test.sh` passes

## Early proof point

Task .1 validates the core detection + injection path (scout finds DESIGN.md → plan writes design context → worker reads it). If the conditional pattern feels too heavy or the frontend heuristic is unreliable, simplify before proceeding to prime/auditor tasks.

## Requirement coverage

| Req | Description | Task(s) | Gap justification |
|-----|-------------|---------|-------------------|
| R1 | Scout detects DESIGN.md | .1 | — |
| R2 | Plan writes design context in frontend task specs | .1 | — |
| R3 | Worker reads design context in Phase 1.5 | .2 | — |
| R4 | Prime DC7 criterion | .3 | — |
| R5 | docs-gap-scout scans for DESIGN.md | .3 | — |
| R6 | Quality-auditor design conformance | .4 | — |
| R7 | Flow-gap-analyst design alignment | .4 | — |
| R8 | Documentation + version bump + codex sync | .5 | — |
