# fn-21-en1.3 Detect sandbox failures and handle gracefully

## Description
Detect when codex fails due to sandbox restrictions (not actual code issues) and surface a clear error instead of masking as NEEDS_WORK.

**Size:** S-M
**Files:** `plugins/flow-next/scripts/flowctl.py`

## Approach

- Modify `run_codex_exec()` to return expanded tuple: `(stdout, session_id, exit_code, stderr)`
- **Update all call sites** of `run_codex_exec()` in the same file to handle new return signature
- Check for sandbox failure using: (a) `exit_code != 0` AND (b) known patterns in stderr or JSON item errors
- Parse Codex JSON lines for items with `"status": "failed"` and `"aggregated_output"` containing "rejected:"
- Patterns to match (in error context only): "blocked by policy", "rejected:", "sandbox"
- If sandbox failure detected, exit with code 3 (distinct from 0 success, 2 general error)
- Log clear message: "Codex sandbox blocked operations. Try --sandbox danger-full-access or set CODEX_SANDBOX=danger-full-access"
- Do NOT write receipt with NEEDS_WORK verdict for sandbox failures
- **Note**: After this task, user must re-run setup/ralph-init to update copies

## Key context

- Current `run_codex_exec()` signature at line 934
- Call sites to update: `cmd_codex_impl_review()`, `cmd_codex_plan_review()`, and any others using this function
- Codex JSON streaming includes `{"type":"item.completed","item":{"status":"failed","aggregated_output":"...rejected..."}}`
- Exit code 3 allows Ralph to distinguish sandbox config issues from code quality issues
- Current Ralph behavior: non-zero exit codes already trigger retry; exit code 3 treated same way (user fixes config to resolve)

## Acceptance
- [x] `run_codex_exec()` returns `(stdout, session_id, exit_code, stderr)` tuple
- [x] ALL call sites of `run_codex_exec()` updated to handle new return signature
- [x] Sandbox failure detected via exit_code != 0 AND error patterns in stderr/JSON items
- [x] JSON items with `status: failed` and `rejected:` in output identified as sandbox errors
- [x] No false positives when reviewed code contains "blocked by policy" phrase
- [x] Exit code 3 returned for sandbox failures (0=success, 2=error, 3=sandbox)
- [x] Clear error message mentions --sandbox flag AND CODEX_SANDBOX env var
- [x] Receipt NOT written when sandbox failure detected
- [x] Normal NEEDS_WORK verdicts (actual code issues) still work correctly

## Done summary
Verified sandbox failure detection is fully implemented in flowctl codex commands. The run_codex_exec() function returns (stdout, session_id, exit_code, stderr) tuple, is_sandbox_failure() detects sandbox policy blocking via exit code and error patterns in stderr/JSON items, and both impl-review and plan-review call sites exit with code 3 for sandbox failures without writing receipts.
## Evidence
- Commits: 761d297, deb1122
- Tests: git diff main -- plugins/flow-next/scripts/flowctl.py | grep is_sandbox_failure
- Review: Codex impl-review SHIP verdict
- PRs: