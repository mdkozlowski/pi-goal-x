# Technical plan: Goal tool retry-loop hardening

## Scope

Keep the `get_goal` placeholder normalization already present in the working tree, and change `update_goal_task` so its schema and executor agree that a non-empty batch takes precedence over redundant single-task fields.

## `get_goal` argument normalization

In `extensions/goal-core-tools.ts`, normalize `task_id` before constructing the `GoalDetailQuery` or applying the summary-form validation:

- `undefined`, `""`, and whitespace-only strings become `undefined`.
- The `"-"` placeholder becomes `undefined` for sections other than `tasks`.
- A non-empty task id for `section="tasks"` remains a task filter. Other non-empty ids retain the existing invalid-section behavior.

Continue passing the normalized id to `goalDetailPage`; do not change pagination or task lookup logic.

## `update_goal_task` schema and dispatch

In `extensions/goal-task-tools.ts`, replace the mutually exclusive `Type.Union` with one `Type.Object` containing optional `task_id`, `status`, `updates`, `evidence`, and `reason` properties. Keep `additionalProperties: false`. The batch array can be empty at schema level so an empty batch can fall back to the single-task form; the executor remains authoritative about whether a batch is usable.

Dispatch in this order:

1. If `updates` is a non-empty array, validate every update and the 100-item limit. Ignore redundant top-level fields. Send the complete list to `GoalService.updateTasks`, which validates and commits it atomically.
2. If `updates` is present but is not an array, reject it.
3. If `updates` is empty and both `task_id` and `status` are present, use the existing single-task path. If either is absent, return a clear validation error.
4. If `updates` is omitted, use the existing single-task path when `task_id` and `status` are present; otherwise return the existing usage error.

A malformed non-empty batch must fail before any mutation and must not fall back to top-level fields. Keep all task status, evidence, skip-reason, focus, ledger, and active-goal checks unchanged.

Update `renderCall` to show a batch count only for a non-empty `updates` array. For an empty or absent batch, render the single-task fields to match executor dispatch. This also prevents optional or malformed `updates` values from causing a renderer error.

## Tests

- Keep the `get_goal` objective tests for empty, whitespace, and dash placeholders.
- Replace the schema-union test with assertions that the single object schema exposes all known properties and allows both `updates` and top-level fields together.
- Add a handler-level regression that sends a valid batch with redundant top-level fields, including `reason: ""`, and verifies that the task is updated and no mutual-exclusion error is returned.
- Test empty-batch fallback to a valid single-task call and an empty batch without single-task fields.
- Preserve atomic rejection tests for invalid batches; remove assertions that mixed forms must fail.

## Validation

Run the focused goal-tool tests and TypeScript check. Run the wider unit suite if focused validation passes. Check the exact changed-file set and `git diff --check` before recording evidence in `MILESTONES.md`.
