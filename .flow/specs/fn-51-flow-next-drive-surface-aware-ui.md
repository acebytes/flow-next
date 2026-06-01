## Conversation Evidence

> user: "which browser driving tool does ray recommend in his skill, i think he uses the build-in codex one too, not just agent-browser like we have here."
> user: "i agree with all of this, the ladder for browser tools so flow-next can support as many as possible, i assume this would mean a rewrite of our browser skill that should probably be renamed to flow-next:browser or something so it doesnt collide with codex browser skills etc."
> user: "wonder if we should not call it browser but another name that would also cover non browser apps that could be ran/automated using computer use, ie. non electron desktop apps, or should that be separate"
> user: "surely the only thing that changes in terms of skill scope is a conditional that says something like if this is a browser app then x, otherwise y"
> user: (name decision) "flow-next-drive"
> [context: RayFernando1337/rayfernando-skills browser-playbook.md + computer-use-playbook.md (Apache-2.0). Driver ladder: cursor-ide-browser -> chrome-devtools-mcp -> browser-use -> Playwright -> Codex Computer Use -> manual. Computer Use is "a fallback for reach, not the default driver"; the only way to reach native macOS/Electron apps; graceful-degradation table: native + CU unavailable -> drive the Electron/Tauri dev-server URL in a browser.]
> [context: flow-next ships skills/browser/ (frontmatter name: browser), hardwired to the agent-browser CLI; sync-codex.sh renames it to agent-browser in the Codex mirror (codex/skills/agent-browser/) -- colliding with the user's global agent-browser skill and Codex-native browser skills.]

## Goal & Context
<!-- scope: business -->
<!-- Source-tag breakdown: 55% [user] / 30% [paraphrase] / 15% [inferred] -->

flow-next ships a `browser` skill hardwired to a single driver (Vercel's agent-browser CLI), and named so it collides: its Codex-mirror rename to `agent-browser` clashes with the user's global `agent-browser` skill and with Codex-native browser skills. The skill should **drive any UI surface — a web/browser app OR a native desktop app (Electron / Tauri / native macOS) — not just browsers**, detecting the surface, picking the right driver for it, and degrading gracefully where a richer driver is absent. Renamed `flow-next-drive` to fix the collision and signal the broader reach. This also unblocks a future `/flow-next:qa` skill, which needs a real driver ladder to drive live apps. The ladder + surface structure borrows from Ray Fernando's running-bug-review-board (Apache-2.0 — credited in CHANGELOG + skill).

## Architecture & Data Models
<!-- scope: technical -->

The skill is restructured from "agent-browser command reference" into a **surface-aware driver ladder**. A top-level step detects the target surface and dispatches on a conditional (browser/web app -> web driver path; native desktop -> Computer Use path). Both paths share one **universal flow** (`observe / navigate -> snapshot -> act on fresh refs -> capture evidence -> release`); only the actuation and the per-surface reference differ. Layout: lean entry point (surface detection + universal flow + the ladder), then per-driver / per-surface reference files. The existing references (commands, advanced, auth, snapshot-refs, session-management, proxy, debugging) fold into the web-surface / agent-browser rung's reference; surface-detection, the universal flow, and the ladder itself are net-new. Canonical files use Claude-native tool names; `sync-codex.sh` rewrites for the Codex mirror (single source of truth).

## Edge Cases & Constraints
<!-- scope: technical -->

- Most execution environments (cloud VMs, Linux, CI) lack Codex Computer Use — it must never be a hard dependency, and a pass must still succeed with whatever driver is present.
- Native desktop with no Computer Use: drive the Electron/Tauri dev-server URL via the web ladder; for a true native macOS app with no CU, surface the limitation rather than failing silently.
- agent-browser is the only driver assumed present; every other rung (chrome-devtools-mcp, cursor-ide-browser, Playwright, Computer Use) is detected and optional.

## Acceptance Criteria
<!-- scope: both -->

- **R1:** Canonical skill renamed `browser` -> `flow-next-drive` (directory, frontmatter `name`, and surfaced invocation `flow-next:flow-next-drive`), matching the `flow-next-*` naming convention used by every other plugin skill. [user]/[paraphrase]
- **R2:** Codex mirror is also named `flow-next-drive` (drop the `agent-browser` rename in `sync-codex.sh`), resolving the collision with the user's global `agent-browser` skill and with Codex-native browser skills. [paraphrase]
- **R3:** Surface detection + conditional dispatch — the skill detects the target surface (web/browser app vs native desktop: Electron / Tauri / native macOS) and branches to the matching driver path. The universal flow (`observe -> act -> verify -> capture`) is shared across surfaces; only actuation + the per-surface reference differ. [user]
- **R4:** Web surface -> driver ladder, in priority order: agent-browser (default rung) -> chrome-devtools-mcp (auto-wait + attach-to-real-signed-in-Chrome) -> Playwright -> cursor-ide-browser MCP -> manual screenshot relay. [paraphrase]
- **R5:** Native-desktop surface -> Codex Computer Use (macOS); graceful degradation when CU is absent — drive the Electron/Tauri dev-server URL via the web ladder, and for a true native macOS app with no CU, document the limitation rather than fail. agent-browser stays the only assumed-present driver; Computer Use is never a hard dependency. [user]/[paraphrase]
- **R6:** Existing agent-browser references (commands, advanced, auth, snapshot-refs, session-management, proxy, debugging) fold into the web-surface / agent-browser rung's per-driver reference — no capability regression for current agent-browser users. [paraphrase]
- **R7:** Cross-platform parity preserved — canonical uses Claude-native tool names; `sync-codex.sh` is updated for the new skill name + surface/ladder content; the Codex mirror is regenerated. [strategy:Cross-platform parity]
- **R8:** Plugin version bumped (skill change, not docs-only per CLAUDE.md). Docs updated across THREE surfaces: (a) **repo** — CHANGELOG (crediting rayfernando-skills), root README, doc index `plugins/flow-next/docs/README.md`, CLAUDE.md, `.flow/usage.md`, and anywhere the old `browser` skill is referenced (platforms/skill lists); (b) **flow-next.dev** (`~/work/flow-next.dev`) — the browser/driver skill page + any guide / workflow page that references the skill by its old `browser` name (the rename touches them) + changelog entry, run the `pnpm build` gate; (c) **mickel.tech** (`~/work/mickel.tech`) — the flow-next app page, **maintainer-only (Gordon updates post-merge; contributor PRs skip it)**. [inferred]
- **R9:** The rename is surfaced as a user-facing change — migration/uninstall notes cover the old `browser` / `agent-browser` skill names so existing references don't silently break. [inferred]

## Boundaries
<!-- scope: business -->

- iOS / iPadOS app driving is OUT of scope — defer to the community iOS simulator skills (as Ray's skill does); never spin a simulator for a web-only app.
- Full native-desktop QA *workflow* (scenario authoring, bug filing, verdict) is downstream `/flow-next:qa`; this skill provides driver/actuation + the surface conditional only.
- NOT the tracker-sync bridge (separate spec, captured next).
- No MCP server (chrome-devtools-mcp, cursor-ide-browser) or Computer Use becomes a hard install dependency — drivers are detected and optional; agent-browser is the only assumed-present one.
- NOT reimplementing what a driver already does (e.g. don't reinvent Playwright or Computer Use) — the skill orchestrates drivers, it doesn't replace them.

## Decision Context
<!-- scope: both -->

The surface change is deliberately minimal — a top-level conditional (browser app -> web ladder; else -> Computer Use), not a separate skill or a heavy native subsystem. The Computer Use rung already reaches native apps, so the marginal cost of covering native desktop is the detection branch + a per-surface reference. agent-browser stays the **default rung** because it's what flow-next already ships, it's CDP-based and headless-safe, and it needs no extra install. Computer Use sits at the native branch (and as a web human-fidelity escalation) rather than the default because most execution environments (cloud VMs, Linux, CI) lack it — making it a dependency would break the common case. The ladder mirrors Ray Fernando's proven structure rather than inventing one. The rename is bundled with the rewrite so shipping a surface-aware ladder under the colliding `browser` / `agent-browser` name doesn't re-introduce the collision the user flagged.

## Requirement coverage

| R-ID | Task |
|------|------|
| R1 | fn-51.M (TBD — populate via /flow-next:plan) |
| R2 | fn-51.M (TBD — populate via /flow-next:plan) |
| R3 | fn-51.M (TBD — populate via /flow-next:plan) |
| R4 | fn-51.M (TBD — populate via /flow-next:plan) |
| R5 | fn-51.M (TBD — populate via /flow-next:plan) |
| R6 | fn-51.M (TBD — populate via /flow-next:plan) |
| R7 | fn-51.M (TBD — populate via /flow-next:plan) |
| R8 | fn-51.M (TBD — populate via /flow-next:plan) |
| R9 | fn-51.M (TBD — populate via /flow-next:plan) |
