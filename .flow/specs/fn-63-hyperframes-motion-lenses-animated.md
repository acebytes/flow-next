# HyperFrames motion lenses: animated spec/PR/pipeline videos

## Conversation Evidence

> user: "make a stub spec for adding beautiful, informative, helpful hyperframes animations (research it if you dont know what it is) -- this would be for flow.next"
> [context: HyperFrames = heygen-com/hyperframes (Apache-2.0) — write HTML with seekable animations (CSS keyframes, GSAP, WAAPI, requestAnimationFrame); the CLI captures the page frame-by-frame in a headless browser and stitches a deterministic MP4. Non-interactive by default, built for agents/CI: same input, same frames, same output. Sources: github.com/heygen-com/hyperframes, hyperframes.video.]
> [context: follows fn-62 (optional HTML artifact mode) — same render-lens philosophy, same opt-in/detect-on-PATH dependency shape (lavish/clawpatch pattern). Notably: GitHub PRs and Linear accept .mp4 attachments — video closes the "GitHub can't render committed HTML" display gap from fn-62.]

## Goal & Context

<!-- Source: 40% [user] / 60% [paraphrase] -->

STUB — not ready; refine via /flow-next:interview before blessing.

Extend fn-62's render-lens family with a **motion lens**: beautiful, informative, helpful animations rendered as deterministic MP4s from the same flow state, via HyperFrames (HTML-with-seekable-animations → headless frame capture → video). The artifact-mode disclosure file gains motion guidance; opted-in skills can emit short videos where motion genuinely informs — a spec's task DAG building up stage by stage with the critical path traced, a PR's churn map animating from base to head, the pipeline narrative (capture → spec → plan → work → review → PR) animated for flow-next.dev / STRATEGY-pipeline use. Because GitHub PRs and Linear accept video attachments, the motion lens can live where the HTML lens cannot be displayed — an animated PR summary embedded directly in the PR body.

## Boundaries

- Builds ON fn-62's artifact mode (config gate, disclosure file, design system, self-containment discipline) — never lands before it. [paraphrase]
- HyperFrames is an optional dep, detect-on-PATH, never required — exactly the lavish/clawpatch shape; absence is invisible. [paraphrase]
- Deterministic HTML rendering, NOT diffusion/AI video — no model calls, reproducible in CI. [paraphrase]
- Motion must inform (reveal sequence, causality, change-over-time), never decorate — "beautiful, informative, helpful" is the bar; a gimmick animation is a defect. [user]/[paraphrase]
- Markdown stays the source of truth; a video is a render lens like any other — regenerable, never parsed back. [paraphrase]

## Open Questions

- Which surfaces earn motion first: spec DAG build-up, PR churn animation, docs-site pipeline hero, release-notes videos? Pick 1-2 for v1.
- Embed targets + size budget: GitHub PR attachment limits (~10MB), Linear attachments, flow-next.dev hosting — does a 15-30s 1080p render fit?
- Dependency weight: HyperFrames needs Node + a headless browser — acceptable as opt-in? Render time per video in interactive vs autonomous flows?
- Animation stack guidance in the disclosure file: CSS keyframes only (zero extra deps) vs allowing GSAP/WAAPI for seekability quality?
- Does the motion lens share fn-62's `.flow/` artifact location and linking rules, or do videos stay out of git (size) with regenerate-on-demand only?
- Autonomous use: may pilot/Ralph render videos (deterministic, non-blocking) or interactive-only at first?

## Acceptance Criteria

- **R1:** STUB — to be defined at interview/planning. flow-next gains an opt-in HyperFrames motion lens producing deterministic, informative MP4 renders of flow state (spec/PR/pipeline), layered on fn-62's artifact mode with the same opt-in + detect-on-PATH discipline. [user]/[paraphrase]
