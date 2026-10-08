# Product: Goal tool retry-loop hardening

## Problem

Model adapters can include empty placeholders for optional `get_goal.task_id`. For a request such as `section="objective"`, the empty string is currently passed to detail-page validation as a supplied task id. The tool returns `task_id requires section=tasks.` Repeated calls with the same arguments do not make progress.

Adapters can also provide both the `update_goal_task.updates` batch and redundant top-level single-task fields. The old executor rejects those fields even when the batch is valid, so the model can repeat the same failing call.

## Required behavior

### `get_goal`

- Treat an empty or whitespace-only `task_id` as omitted for every section.
- Treat `task_id="-"` as an omitted placeholder outside `section="tasks"`.
- Preserve normal validation for non-empty task ids and for task-specific lookup requests.
- Return the requested objective, history, or task listing without an error when an omitted-task placeholder is present.

### `update_goal_task`

- Keep one permissive object schema for the known fields so a call may contain both `updates` and top-level single-task fields.
- When `updates` is a non-empty array, validate and apply the batch atomically. Ignore redundant top-level single-task fields, including empty strings.
- When `updates` is omitted or empty, use `task_id` and `status` as the single-task form when both are present.
- Reject a malformed non-empty batch without partially applying it or falling back to the single-task fields.
- Return a clear input error when neither a usable batch nor a complete single-task form is supplied.

These rules prevent repeated identical tool failures. They do not weaken task-transition validation, evidence requirements, skipped-task reasons, or batch atomicity.

## Success criteria

1. `get_goal` returns objective details when `task_id` is `""`, whitespace, or the non-task placeholder `"-"`.
2. A valid non-empty `update_goal_task.updates` batch succeeds when redundant top-level fields are also present.
3. A batch still fails atomically when one update is invalid, even if top-level single-task fields are valid.
4. An empty batch falls back to valid single-task fields; without those fields it returns a clear validation error.
5. Existing single-task, ordered-batch, lifecycle, and detail-pagination behavior remains unchanged.

## Non-goals

- Add a generic retry loop or automatic tool-call retry policy.
- Ignore malformed batch entries or weaken task lifecycle rules.
- Change task lookup semantics for non-empty task ids.
