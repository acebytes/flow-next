# fn-21-en1.4 Add CODEX_SANDBOX config option to Ralph

## Description
Add CODEX_SANDBOX configuration option to Ralph's config.env template and wire it through to flowctl calls.

**Size:** S
**Files:** 
- `plugins/flow-next/skills/flow-next-ralph-init/templates/config.env`
- `plugins/flow-next/skills/flow-next-ralph-init/templates/ralph.sh`

## Approach

- Add `CODEX_SANDBOX=auto` option in config.env with comment explaining values
- In ralph.sh, export CODEX_SANDBOX so Claude workers and flowctl can read it
- Update flowctl's resolve_codex_sandbox() to support both CLI and env var
- **Priority**: CLI --sandbox > CODEX_SANDBOX env var > platform default
- Place near existing PLAN_REVIEW and WORK_REVIEW options
- Default to `auto` which uses platform detection in flowctl

## Key context

- Config.env pattern: `VARNAME=value` with `# comment` above
- Ralph.sh reads config via `source config.env` at script start (with `set -a` to export)
- **Architecture note**: ralph.sh does NOT call flowctl codex directly; Claude workers do via skills
- The env var approach allows flowctl to read CODEX_SANDBOX regardless of how it's invoked
- CLI flag can override env var for one-off testing
- Follow existing pattern for PLAN_REVIEW/WORK_REVIEW options
## Acceptance
- [x] `CODEX_SANDBOX=auto` added to config.env template with explanatory comment
- [x] ralph.sh exports CODEX_SANDBOX env var for Claude workers and flowctl
- [x] flowctl resolve_codex_sandbox() reads both CLI --sandbox and CODEX_SANDBOX env var
- [x] **Priority order**: CLI --sandbox > CODEX_SANDBOX env var > platform default
- [x] Default value `auto` works on both Windows and Unix
- [x] Comment documents valid values: auto, read-only, workspace-write, danger-full-access
- [x] Invalid CODEX_SANDBOX values produce clean error message (not stack trace)
## Done summary
Added CODEX_SANDBOX config option to Ralph's config.env template with documentation for valid values. Updated ralph.sh to export the env var for Claude workers. The flowctl resolve_codex_sandbox() function reads both CLI --sandbox and CODEX_SANDBOX env var with correct priority order. Additional fix: added stale receipt cleanup on non-sandbox codex failures to prevent false gate satisfaction.
## Evidence
- Commits: 1c86b20, bfbb0ba, 1f3d12d, 73e3dbc, 7d343ee, deb1122
- Tests: manual verification of config.env, ralph.sh, and flowctl.py
- Review: Codex impl-review SHIP verdict after fixes
- PRs: