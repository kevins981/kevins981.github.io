# CRMArena Trace Format (Trajectory JSON)

## File structure

CRMArena trajectories appear as single JSON objects with the following top-level fields:

- `task_id`: Integer task identifier.
- `task_type`: Task category string (e.g., `monthly_trend_analysis`).
- `gt_answer`: Ground-truth short answer string used for evaluation.
- `reward`: Numeric score for the run (commonly `0` for failure, `1` for success).
- `agent_info`: Metadata about the run (token usage, termination reason, number of turns).
- `traj`: Ordered list of events representing the agent-environment interaction.

## `agent_info` conventions

Common fields inside `agent_info`:

- `num_turns`: Number of assistant turns (not necessarily equal to number of `traj` entries).
- `end_reason`: How the run terminated, typically:
  - `source`: e.g., `agent`
  - `message`: e.g., `Submit action`
  - `content`: often `None`
- `usage`: Token and cost accounting, typically arrays per turn:
  - `prompt_tokens`, `completion_tokens`, `total_tokens`, `cost`

## `traj` event schema

Each element of `traj` is a chat-style message object. Typical patterns:

### System message

- `role`: `system`
- `content`: Benchmark instructions, task context (including “Today's date”), and domain definitions (quarters, seasons, time periods).

### User message

- `role`: `user`
- `content`: Natural-language task request. Often includes constraints like “Return only the month name.”

### Assistant tool-call message

The assistant generally emits tool calls rather than free-form answers:

- `role`: `assistant`
- `content`: often `null`
- `tool_calls`: list of tool call objects:
  - `id`: tool call id string
  - `type`: typically `function`
  - `function.name`: tool name (e.g., `get_start_date`)
  - `function.arguments`: JSON-serialized string of arguments

### Tool result message

Tools respond as separate `role: tool` messages:

- `role`: `tool`
- `tool_call_id`: matches the assistant tool call `id`
- `name`: tool name (matches `function.name`)
- `content`: tool output as a string (may itself encode JSON-like content)

### Final submission

The final answer is submitted via a `respond` tool call:

- `role`: `assistant` with `tool_calls[0].function.name == "respond"`
- `function.arguments`: JSON string like `{"content":"December"}`

The benchmark instructions typically require that `respond` contains only the short answer (no explanation), and uses the literal string `None` when no records match.

# Correctness Condition (CRMArena Benchmark)

## Definition

Correctness is determined solely by the final answer submitted by the agent.

## Task taxonomy (benchmark-specific)

For this benchmark, there is a fixed set of `task_type` values:

- `monthly_trend_analysis`
- `top_issue_identification`
- `handle_time`
- `transfer_count`
- `best_region_identification`

## Golden answer source

The unique golden final answer for each task is provided in the trajectory JSON as `gt_answer`.

## Submission mechanism

The agent submits its final answer by calling the `respond()` tool. The task finishes immediately when `respond()` is called.

## Evaluation implication

Only the content submitted via `respond()` is scored. Intermediate tool calls, intermediate messages, and reasoning are not used to determine correctness.

## Triage workflow heuristic

When triaging a failure, first identify the trajectory’s `task_type`, then consult the corresponding “golden process” section in `003_golden_processes_all_task_types.md` to generate hypotheses about the agent’s intermediate steps.

The “golden process” descriptions are not required or official procedures. Deviations from them are not, by themselves, evidence of failure. They are heuristics that can help localize likely root causes when the final `respond()` value is incorrect.

# Golden Processes (All Task Types)

## Purpose

These are heuristic “golden” trajectories for each CRMArena `task_type`. They are not evaluation rules; they are triage aids for diagnosing why a final `respond(...)` submission is incorrect.

## `best_region_identification`

### Purpose

Identify the single U.S. state (two-letter abbreviation) with the fastest case closure time over a specified time window.

### Golden expected trajectory

1. Determine the date range (`start_date`, `end_date`) from the prompt and “Today’s date”.
2. Fetch cases in range via `get_cases(start_date, end_date)`.
3. Add shipping state to each case via `get_shipping_state(...)`.
4. Calculate average closure time by state via `calculate_region_average_closure_times(...)`.
5. Select the state with the minimum closure time via `find_id_with_min_value(...)`.
6. Submit only the two-letter state abbreviation via `respond(...)`.

### Notes

- The final answer must be exactly the two-letter abbreviation (e.g., `CA`) and nothing else.
- If no records match the requested window/filters, submit `None` via `respond(...)`.

## `transfer_count`

### Purpose

Compare agents by the number of cases they transferred over a specified time window, subject to a minimum “cases handled” threshold.

### Golden expected trajectory

1. Determine the date range (`start_date`, `end_date`).
2. Get number of cases handled by each agent via `get_agent_handled_cases_by_period(...)`.
3. Filter agents by the minimum handled threshold via `get_qualified_agent_ids_by_case_count(...)`.
4. Get number of cases transferred by each qualified agent via `get_agent_transferred_cases_by_period(...)`.
5. Select the maximum transferred value via `find_id_with_max_value(...)`.
6. Submit the required final value via `respond(...)` (often the agent id, depending on the prompt).

### Notes

- The final `respond(...)` content must match the prompt’s formatting constraints exactly (e.g., id-only, number-only, or `None`).

## `handle_time`

### Purpose

Identify the agent associated with the highest average handle time over a specified date range, using only cases that were not transferred.

### Golden expected trajectory

1. Determine the date range (`start_date`, `end_date`).
2. Filter to non-transferred cases within the date range via `get_non_transferred_case_ids(...)`.
3. Fetch details for the filtered cases via `get_cases(...)`.
4. Compute average handle time by agent via `calculate_average_handle_time(...)`.
5. Select the maximum average via `find_id_with_max_value(...)`.
6. Submit the required final value via `respond(...)` (often the agent id, depending on the prompt).

### Notes

- Prefer `get_non_transferred_case_ids(...)` over post-hoc filtering so transferred cases are excluded correctly.
- The final `respond(...)` content must match the prompt’s formatting constraints exactly.

## `monthly_trend_analysis`

### Purpose

Identify the month with the highest or lowest number of cases over a specified date range, typically scoped to a specific product via its order items.

### Golden expected trajectory

1. Determine the date range (`start_date`, `end_date`).
2. Get order item ids for the target product via `get_order_item_ids_by_product(...)`.
3. Fetch cases in range filtered to those order items via `get_cases(...)`.
4. Compute number of cases per month via `get_month_to_case_count(...)`.
5. Select the max/min month via `find_id_with_max_value(...)` or `find_id_with_min_value(...)` as required.
6. Submit the required final value via `respond(...)` (often a month name, depending on the prompt).

### Notes

- Some prompts require month name only (no year, no extra text).
- If no records match the requested window/filters, submit `None` via `respond(...)`.

## `top_issue_identification`

### Purpose

Identify the most (or least) frequent issue type for a specified product over a specified date range.

### Golden expected trajectory

1. Determine the date range (`start_date`, `end_date`).
2. Get order item ids for the target product via `get_order_item_ids_by_product(...)`.
3. Get issue-type counts for those order items via `get_issue_counts(...)`.
4. Select the max/min issue via `find_id_with_max_value(...)` or `find_id_with_min_value(...)` as required.
5. Submit the required final value via `respond(...)` (often the issue label/id requested by the prompt).

### Notes

- Follow the prompt’s formatting constraints exactly (e.g., label-only, no explanation).
- If no records match the requested window/filters, submit `None` via `respond(...)`.

# Common Failure Patterns (CRMArena Triage)

## Purpose

This unit lists recurring failure patterns that commonly explain incorrect `respond(...)` submissions. It functions as an index that points to more detailed units when available.

## Pattern catalog

### CF-001: Tool output truncation leads to incomplete aggregates

**Symptom**

A `role: tool` message ends with an observation truncation marker such as:
`<Observation too long and is truncated. Showing the first 10000 characters only.>`

**Why it fails**

The agent performs min/max selection, counts, or averages on a partial set of records, then submits an answer as if it covered the full dataset.

**Typical affected tasks**

Any task requiring aggregation over all matching records (e.g., state with minimum average closure time, most common issue type, month with max/min counts).

**Mitigation**

Treat truncation as incomplete data; change retrieval strategy to obtain the full dataset without truncation.

**See also**

`008_tool_output_truncation_failure_mode.md`
