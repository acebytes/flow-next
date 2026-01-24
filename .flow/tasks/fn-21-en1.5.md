# fn-21-en1.5 Update documentation for Windows Codex sandbox

## Description
Update documentation to explain Windows Codex sandbox limitations, new configuration options, and re-run requirement.

**Size:** S
**Files:**
- `plugins/flow-next/docs/flowctl.md`
- `plugins/flow-next/docs/ralph.md`
- `CLAUDE.md`
- `CHANGELOG.md`

## Approach

- flowctl.md: Add --sandbox flag to codex command docs for BOTH impl-review and plan-review, explain Windows behavior and `auto` resolution
- flowctl.md: Document that plan-review now requires `--files` argument
- ralph.md: Document CODEX_SANDBOX config option with valid values
- ralph.md: Document exit code 3 = sandbox config issue. Note: Ralph already retries on non-zero exit codes, so exit code 3 will be retried (user should fix config to resolve)
- CLAUDE.md: Add note in Codex section about Windows requiring `danger-full-access` or `auto`
- CHANGELOG.md: Add entry under next version's Fixed section
- ALL docs: Note that users must re-run `/flow-next:setup` or `/flow-next:ralph-init` after plugin update to get the fix

## Key context

- flowctl.md codex section: lines 488-549
- ralph.md Codex Integration: lines 325-347
- CLAUDE.md Codex section: lines 187-220
- CHANGELOG format: `- **Title** — Description` under `### Fixed`
- Exit code 3: Ralph will retry it (existing behavior), user must fix CODEX_SANDBOX config to resolve

## Acceptance
- [ ] flowctl.md documents `--sandbox` flag for BOTH impl-review and plan-review
- [ ] flowctl.md documents `--files` requirement for plan-review
- [ ] flowctl.md explains `auto` resolution (danger-full-access on Windows, read-only on Unix)
- [ ] flowctl.md documents `FLOW_CODEX_EMBED_MAX_BYTES` env var for embedding budget
- [ ] ralph.md documents CODEX_SANDBOX config option with valid values
- [ ] ralph.md documents FLOW_CODEX_EMBED_MAX_BYTES config option
- [ ] ralph.md documents exit code 3 = sandbox config issue (Ralph retries, user must fix config)
- [ ] ralph.md adds troubleshooting for "blocked by policy" errors
- [ ] CLAUDE.md notes Windows requires `--sandbox danger-full-access` or `auto`
- [ ] ALL docs note: re-run setup/ralph-init after plugin update to get fixes
- [ ] CHANGELOG.md has entry: "Fixed Codex sandbox on Windows blocking all reads"

## Done summary
Updated documentation for Windows Codex sandbox fix: added --sandbox flag docs for flowctl codex commands, documented CODEX_SANDBOX config option for Ralph, added troubleshooting section for "blocked by policy" errors, and updated CHANGELOG.
## Evidence
- Commits: 9b6adcdf4f6574c858a9329e0bf64a26e2403992
- Tests: flowctl codex impl-review fn-21-en1.5 --base 7d343ee --sandbox auto (SHIP verdict)
- PRs: