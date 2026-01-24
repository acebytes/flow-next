# fn-21-en1.2 Embed full file contents in codex review prompt

## Description
Embed full contents of changed files in codex review prompts for BOTH impl-review and plan-review. This allows codex to review code without filesystem access.

**Size:** M
**Files:** `plugins/flow-next/scripts/flowctl.py`

## Approach

- Create shared helper function `get_embedded_file_contents(file_paths)`
- Call from BOTH `cmd_codex_impl_review()` AND `cmd_codex_plan_review()`
- **File source for impl-review**: Use `get_changed_files(base_branch)` (existing helper at line 658)
- **File source for plan-review**: REQUIRE `--files` argument (comma-separated paths). No automatic parsing.
- Read files as bytes first to detect binary (null byte in first 1KB)
- Decode with `errors="replace"` to handle encoding issues
- **Configurable budget** via `FLOW_CODEX_EMBED_MAX_BYTES` env var (default 0 = unlimited)
- Skip binary files with note in output
- Status message format:
  ```
  [Embedded X of Y files (Z bytes)]
  [Skipped (binary): file1.png, file2.woff]
  [Skipped (budget exhausted): file3.py, file4.py]
  ```
- Add strong injection warning at TOP of embedded section

## Key context

- `get_changed_files(base_branch)` helper - reuse for impl-review
- Plan-review: require explicit `--files` argument, don't auto-parse spec (too brittle)
- Binary detection: check first 1KB for `\x00` byte
- Budget is optional: `FLOW_CODEX_EMBED_MAX_BYTES=50000` for 50KB limit, unset/0 for unlimited

## Acceptance
- [x] Shared helper function used by BOTH impl-review AND plan-review
- [x] impl-review: embeds files from `get_changed_files(base_branch)`
- [x] plan-review: embeds files from REQUIRED `--files` argument (comma-separated)
- [x] plan-review without `--files` produces clear error message
- [x] Files read as bytes first, then decoded with errors="replace"
- [x] Binary files (null byte in first 1KB) skipped
- [x] Status shows: `[Embedded X of Y files (Z bytes)]`, `[Skipped (binary): ...]`
- [x] Strong injection warning at top of embedded section
- [x] Deleted files (in diff but not on disk) handled gracefully with note
- [x] Configurable budget via `FLOW_CODEX_EMBED_MAX_BYTES` env var (default unlimited)
- [x] Budget exhaustion shows `[Skipped (budget exhausted): ...]` status

## Done summary
## Summary
Task fn-21-en1.2 was already implemented in previous commits. The `get_embedded_file_contents()` helper function exists and is used by both impl-review and plan-review. All acceptance criteria are satisfied.

## Changes
No changes needed - implementation was already complete:
- `get_embedded_file_contents()` helper at line 673-859 in flowctl.py
- Called from impl-review (line 5029) using `get_changed_files(base_branch)`
- Called from plan-review (line 5210) using required `--files` argument
- Binary detection, path traversal protection, budget support all present

## Verification
- `--files` argument required for plan-review (argparse `required=True`)
- Plan-review without --files exits with clear error message
- FLOW_CODEX_EMBED_MAX_BYTES env var supported for budget control
- Injection warning present at top of embedded section

## Testing
- Verified plan-review --help shows --files as required
- Verified plan-review without --files produces error message
- Diffed canonical source vs .flow/bin copy - identical
## Evidence
- Commits:
- Tests:
- PRs: