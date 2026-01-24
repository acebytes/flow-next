# fn-21-en1.1 Add --sandbox CLI flag to flowctl codex commands

## Description
Add a `--sandbox` CLI argument to `flowctl codex impl-review` and `flowctl codex plan-review` commands.

**Size:** S
**Files:** `plugins/flow-next/scripts/flowctl.py`

## Approach

- Add `--sandbox` argument to argparse definitions at lines 5686-5710 for BOTH impl-review and plan-review
- Valid CLI values: `read-only`, `workspace-write`, `danger-full-access`, `auto` (default)
- **Resolve `auto` inside flowctl**: use `danger-full-access` if `os.name == 'nt'`, else `read-only`
- Never pass `auto` to Codex CLI - always resolve to a real value first
- Pass resolved sandbox value through to `run_codex_exec()` function

## Key context

- Codex CLI valid sandbox values: `read-only` (default), `workspace-write`, `danger-full-access`
- `auto` is NOT a valid Codex value - flowctl must resolve it before calling codex
- Windows sandbox is experimental and blocks all reads (openai/codex#9062)
- Platform detection: `os.name == 'nt'` for Windows

## Acceptance
- [ ] `flowctl codex impl-review --sandbox <mode>` accepts all four values
- [ ] `flowctl codex plan-review --sandbox <mode>` accepts all four values
- [ ] `auto` resolved to `danger-full-access` on Windows, `read-only` on Unix BEFORE codex exec
- [ ] `auto` is NEVER passed to Codex CLI
- [ ] Invalid sandbox value produces clear error message
- [ ] `flowctl codex impl-review --help` shows sandbox option with valid values
## Done summary
Added --sandbox CLI flag to flowctl codex impl-review and plan-review commands with choices: read-only, workspace-write, danger-full-access, auto (default). Auto mode resolves to danger-full-access on Windows and read-only on Unix before passing to Codex CLI. Also fixed accidentally committed .pyc file and added Python cache patterns to .gitignore.
## Evidence
- Commits: c8b049b, fe5c87aaff8f3e30c3517e0e5622afe40f708dd2
- Tests: python flowctl.py codex impl-review --help, python flowctl.py codex plan-review --help, resolve_codex_sandbox unit test, invalid sandbox value test
- PRs: