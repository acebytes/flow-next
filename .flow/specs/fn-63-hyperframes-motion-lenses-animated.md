# fn-63-hyperframes-motion-lenses-animated HyperFrames motion for flow-next.dev: hero, pipeline, artifact clips

## Conversation Evidence

> user: "what is a motion lens, i dont want to use hyperframes as another type of reader lens, i would want to use it to improve the flow-next.dev website wherever you think it would make sense, maybe the spec is wrong"
> user (original stub): "make a stub spec for adding beautiful, informative, helpful hyperframes animations"
> [context: REWRITE 2026-06-12 — the original stub framed HyperFrames as a "motion lens" (deterministic MP4 renders of flow state as a third review surface). The user rejected that framing: review surfaces shipped in 2.0.0 (fn-62 HTML render lenses); HyperFrames should improve flow-next.dev itself.]
> [context: HyperFrames = heygen-com/hyperframes (Apache-2.0) — HTML with seekable animations (CSS keyframes, GSAP, WAAPI, rAF) → headless frame capture → deterministic MP4. The full HyperFrames skill pack (core, cli, media, registry + css-animations/gsap/waapi adapters) is installed in the dev environment.]

## Goal & Context

Use HyperFrames to add beautiful, informative, helpful motion to **flow-next.dev** — authored video/animation assets that explain the product better than static images, rendered deterministically from compositions living in the site repo. Candidate surfaces (validate via smoke test before committing scope):
1. **Landing hero** — short, muted, looping pipeline-in-motion clip (intent → spec → task graph materializes → re-anchored workers → reviewed PR → ship). [paraphrase]
2. **strategy/pipeline page** — the two static mermaid diagrams become one authored, captioned walkthrough animation of the canonical pipeline. [inferred]
3. **Visual-aids pages** — the fn-62 artifacts ARE HTML (HyperFrames' native input): render the spec lens DAG drawing itself and the PR checklist filling into short clips that convey interactivity better than the current static screenshots. [inferred]
4. **Release clips** — a 2.0.0-style announcement short, reusable for social (manual posting per repo rules). [inferred]

## Boundaries

- NOT a flow-next plugin feature and NOT a render lens — no flowctl config, no skill touchpoints, no artifact contract changes; deliverables live in the flow-next.dev repo. [user]
- HyperFrames is a site-repo dev dependency only (compositions + `npx hyperframes render` in the site dev loop); the flow-next plugin's zero-dep contract is untouched. [paraphrase]
- Motion must inform (sequence, causality, change-over-time), never decorate; `prefers-reduced-motion` falls back to the existing static images; no autoplay-with-sound. [user]/[paraphrase]
- Deterministic renders — same composition, same MP4; committed or build-rendered (decide at interview). [paraphrase]
- NOT per-skill-page videos (24 low-value units) — a few high-leverage moments only. [inferred]
- Aesthetic lineage: reuse the fn-62 instrument-panel house style (palette, typography) so site motion and render lenses read as one family. [inferred]

## Open Questions

- Which surfaces make v1: hero + pipeline page only, or also visual-aids clips + release short?
- Asset strategy: commit rendered MP4s (size? LFS?) vs render in CI/build; size budget per clip (hero must stay light — target <2-3MB, dimensions/fps?).
- Animation stack per composition: CSS keyframes (zero deps, fully seekable) vs GSAP/WAAPI where stagger/orchestration needs it — adapter choice per the installed skills.
- Hero loop length + storyboard: what is the 8-15s version of the pipeline story?
- Do the artifact clips screen-capture the real fn-62 artifacts (authentic) or re-author them as HyperFrames compositions (controllable timing)?
- Video element strategy on Astro/Starlight: poster images, lazy-load, `prefers-reduced-motion` swap mechanics.

## Acceptance Criteria

- [x] **R1:** (shipped without formal planning — built directly at user direction, 2026-06-12) flow-next.dev gains deterministic HyperFrames-authored motion at a small number of high-leverage surfaces (landing hero, pipeline page; candidates: visual-aids clips, release short), with reduced-motion fallbacks and explicit size budgets; the flow-next plugin itself is unchanged. [user]/[paraphrase]

## Shipped (2026-06-12 — closed against delivered work)

Built conversationally at user direction ("i just want you to make some cool videos that improve our site") rather than through interview/plan — the spec closes against the delivered outcome:

- **Landing hero loop — SHIPPED.** 64.5s seamless loop, 11 beats covering every stage (intent → spec w/ provenance chips → tracker sync → plan + plan-review → re-anchored workers → adversarial review + receipts → render lenses → PR-as-cognitive-aid → pilot/land/ralph autonomy → ship → close). Two user-driven iterations: v2 (12s→55s, "too fast / not detailed enough"), v2.1 (≥1.5s settled hold before every transition; close holds 3.25s). Live on flow-next.dev.
- **Pipeline page walkthrough — SHIPPED.** 30s captioned explainer (8 beats incl. the pilot/land orbit visualization) embedded above the mermaid on /strategy/pipeline/.
- **Artifact clips for visual-aids pages — NOT shipped.** Static dogfood screenshots (fn-62) already serve those pages; revisit only if demand appears.
- **Release/social short — NOT shipped.** Deferred; the composition patterns now exist to make one cheaply.

**Open questions → answers-by-doing:** surfaces v1 = hero + pipeline page · assets = rendered MP4s committed to site `public/videos/` (hero 4.9MB, walkthrough 3.0MB), compositions + manifests + lockfiles in site `motion/` (raw renders gitignored; reproducible via `npx hyperframes render`) · stack = GSAP timelines via HyperFrames (deterministic, seekable) · embeds = autoplay/muted/loop hero band + controls-gated docs video, both with `prefers-reduced-motion` poster fallbacks · quality gates = hyperframes lint/validate (WCAG AA)/inspect/animation-map + loop-endpoint PSNR.

**Evidence (flow-next.dev repo):** 314e7e5 (videos + embeds + band), 3bf4977 (hero v2), 7f4ed44 (hero v2.1 settle holds), a1d7ca2 + c285dc4 (reproducibility housekeeping). Design contract: site `motion/design.md` (instrument-panel house style shared with the fn-62 render lenses).
