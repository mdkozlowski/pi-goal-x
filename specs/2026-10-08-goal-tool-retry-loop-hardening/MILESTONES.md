# Milestones: Goal tool retry-loop hardening

## 2026-10-08 — Retry-loop behavior implemented

Optional `get_goal.task_id` values can arrive as empty or whitespace-only strings, or as `"-"` for non-task detail sections. Passing these placeholders as task filters causes detail-page validation to reject an otherwise valid request.

Some adapters send a non-empty `update_goal_task.updates` batch together with redundant top-level single-task fields. Rejecting the combination prevents a valid batch from running and can lead the model to repeat the same failing call.

An empty batch falls back to single-task fields when both are present. A malformed non-empty batch is rejected before mutation and does not fall back. The call renderer now handles optional `updates` values safely. Regression coverage replaces the old assertion that mixed forms must fail.

### Validation

- Focused goal tests: 36 passed, 0 failed.
- `node scripts/run-unit-tests.mjs`: 1,057 passed, 0 failed across 44 suites.
- `node scripts/run-unit-tests.mjs integration`: 31 passed, 0 failed.
- `node_modules/.bin/tsc --noEmit`: passed.
- ESLint on the four changed TypeScript files: passed.
- `git diff --check`: passed.
- LSP diagnostics found no errors. They report two existing unused-import hints in `extensions/goal-core-tools.ts` for `Theme` and `findTaskInTree`; those imports were not part of this change.
