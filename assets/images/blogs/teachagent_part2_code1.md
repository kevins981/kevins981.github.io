# Purpose and Scope

This knowledge base supports *triage of failure reasons* for traces produced by an airline assistant agent. “Triage” here means: determine what went wrong (or why the agent transferred/refused) in a given failed trace, using a consistent taxonomy and a repeatable workflow.

## Inputs and Outputs

### Inputs

The primary input is a single agent trace: an ordered sequence of messages/events capturing the interaction between user, assistant, and tools.

### Outputs

The output of triage is a structured diagnosis consisting of the **root cause** of the failure within the trajectory. 
For strategies to accurately identify the failure root cause, check `006_Tool_Call_First_Triage_Strategy.md`

### Key Artifact Directories

1. `traces/`: failure agent traces that are the primary focus for triage.
2. `agent_tools/`: definitions of tools available to the agent under test; used to triage tool-usage mistakes.

### Evaluation Rules

When present, `info.task.actions` defines the minimum required set of tool functions the agent must call to be considered correct. The agent may call additional tools, but calling fewer than this minimum set is incorrect.

When an agent calls additional tools beyond `info.task.actions`, those extra actions are judged by their effect on the airline system’s end state: correctness depends on whether the final state matches the end state produced by executing only the expected (“golden”) actions.

# Trace Model and Artifacts

This section defines the structural units of evidence used during triage. A “trace” is treated as an immutable record of interaction events.

## Core Entities

### Trace

A trace is an ordered sequence of events representing one end-to-end attempt to complete a user request.

In this corpus, the ordered event list is stored in the `traj` field; `traj` is the agent’s trajectory for the attempt.

## Benchmark Setup (Airline Agent Workload)

Each test case is an airline customer-service scenario evaluated via a multi-turn conversation between:

1. A **target agent** (the airline assistant being tested).
2. A **simulated user** produced by a separate LLM.

At the start of the test case, the simulated-user LLM is conditioned with the task-level instruction found in `info.task.instruction`. The subsequent conversation in `traj` is then generated turn-by-turn.

### Reading Intent vs. Oracle Actions

During triage, use both:

- `info.task.instruction` to understand what the simulated user is trying to achieve in the dialogue, and
- `info.task.actions` to understand what the benchmark expects the target agent to do (the required minimum set of tool calls).

However, the recommended *workflow order* is to start from `info.task.actions` and the realized tool calls, because many failures are detectable as a first mismatch between (a) missing required calls, (b) incorrect tool arguments, or (c) extra state-mutating calls that perturb end state. See `006_Tool_Call_First_Triage_Strategy.md`.

A common and intentional benchmark pattern is that the dialogue may include user requests that appear to go beyond the oracle actions. In such cases, interpret the extra requests as disallowed/out-of-policy for the scenario unless the oracle includes corresponding actions.

### Event

An event is a single step in the trace. Events are categorized by *role*:

1. **System**: policy and run context provided to the assistant.
2. **User**: user utterances (including follow-ups and corrections).
3. **Assistant**: the agent’s natural-language responses and tool-calling decisions.
4. **Tool**: tool execution results and system-level notifications (e.g., transfer completion).

### Outcome Signal

Traces typically contain an external outcome signal (for example, a numeric reward or pass/fail label). In this corpus, the `reward` field is authoritative for success classification: `reward = 1` indicates the task was completed successfully, and `reward = 0` indicates failure. Triage uses the outcome signal to decide whether a trace is considered a failure case, but does not treat it as sufficient evidence for *why* the trace failed.

### Expected Actions (Task Oracle)

Some traces also include a task specification that defines a minimum required set of agent actions. When present, `info.task.actions` is the authoritative list of tool functions that must be called for the trace to be considered correct. This list is a minimum set: the agent may call additional tools, but omitting any required action is sufficient evidence that the attempt is incomplete.

In this project’s evaluation setup, `info.task.actions` is treated as the golden reference for which tool calls are permitted. If the user conversation requests additional actions not reflected in `info.task.actions`, the agent is not expected to perform those extra actions; tool calls made to satisfy them are treated as incorrect.

#### Empty Oracle Action List Implies No State Mutation

In this benchmark, the oracle (`info.task.actions`) is always correct. If `info.task.actions = []`, the agent should not call any **state-changing** tools. Any state mutation performed via tools should be treated as an oracle violation and a likely root cause of `reward = 0` outcomes.

#### State-Changing vs. Read-Only Tools

Correctness checking depends on whether extra tool calls mutate system state. During triage:

- Treat tools that modify reservations, tickets, payments, or baggage as **state-changing**.
- Treat lookup/search tools as **read-only**.
- Treat `transfer_to_human_agents` as **state-changing** (it changes the execution path by escalating the case).

When the task-level instruction or simulated user dialogue appears to request actions beyond `info.task.actions`, interpret the mismatch as intentional: the extra requests are typically disallowed or out-of-policy for the scenario, and the agent is expected to refuse/avoid them while completing the oracle-required subset.

#### Instruction–Action Mismatch Implies Disallowed Actions

In this benchmark, treat both `info.task.instruction` (task-level intent) and `info.task.actions` (golden minimum tool calls) as authoritative parts of the test case.

When they appear to conflict—e.g., the instruction or simulated user dialogue asks for **A, B, and C**, but `info.task.actions` contains only **C**—interpret this as intentional:

- The agent **must not perform** A/B (they are typically disallowed, illegal, unsafe, or out-of-policy in the scenario).
- The agent is expected to refuse/deflect A/B while still completing the allowed subset (C).

**Triage implication:** Do not mark A/B as “missing required tool calls” when A/B are not present in `info.task.actions`. Prefer diagnosing other failure sources (wrong arguments, end-state perturbation, premature transfer, etc.) if `reward = 0` despite completing the oracle action set.

### Correctness Checking (End-State Equivalence)

Correctness is not determined solely by matching tool calls to the expected actions list. When an agent calls additional tools beyond `info.task.actions`, those extra actions are evaluated by their effect on the system’s final state. Conceptually, the benchmark compares the end state produced by the agent’s full action sequence to the end state produced by executing only the expected (“golden”) actions. Additional actions are “incorrect” if they perturb the end state so that it no longer matches the golden end state.

## Benchmark Artifacts and Limits

This section captures systematic benchmark limitations that can produce “failures” which are not actionable for agent changes. These items may still be recorded during triage for completeness and trend analysis.

### Simulated-User Drift (LLM User Contradicts Task Instruction)

#### Definition

In this benchmark, the “user” is simulated by an LLM. The simulation is conditioned on a task-level instruction, but later simulated user turns may contradict that instruction. When the agent follows the later contradictory user message, evaluation can mark the trace incorrect because the golden action remains aligned to the task-level instruction.

#### Implication for Triage

These cases should still be recorded as observed failures, but they are typically not actionable: the root cause is inconsistency in the simulated user’s behavior rather than a missing capability or a straightforward policy/logic bug in the agent.

<meta-info>
* **Authority:** Treat the task-level instruction as the authoritative user intent; later simulated user turns may be inconsistent with it.
* **Triage Labeling:** When a later simulated user turn contradicts the task-level instruction, treat the incident as an LLM user failure (simulator failure), not an agent failure.
* **Non-Actionability:** Do not expect the agent to detect or resolve these contradictions; treat them as out of scope for agent fixes.
* **Triage Policy:** Record simulated-user drift cases, but treat them as largely non-actionable for agent improvements unless the project explicitly changes evaluation or modifies the simulator to avoid contradictions.
</meta-info>

# Common Failure Patterns

This section defines recurring failure patterns observed during trace triage. A “failure pattern” is a reusable diagnosis template: a characteristic causal chain that explains why a trace failed, independent of the specific user request.

## Using This Section

Apply patterns by first localizing the earliest mismatch between the agent’s realized tool calls and `info.task.actions` (when present). This reduces ambiguity about “what went wrong” before attributing “why it went wrong.” See `006_Tool_Call_First_Triage_Strategy.md`.

## Pattern: Premature Human Transfer (In-Scope Task)

### Definition

The agent transfers to a human despite the request being in scope and progressable with available tools/constraints. This excludes correct transfers (out of scope, blocked), and includes transfers that omit the blocking reason and any in-scope alternatives.

### Diagnostic Criteria

This pattern applies when all of the following are true:

1. The user request is within the domain scope described by the trace’s system/policy context.
2. The trace contains a human-transfer action (for example, a transfer tool call) as the terminal action or an early action that prevents progress.
3. No evidence in the trace shows an actual blocking constraint (for example, the user refuses to confirm an update, required identifiers are missing and cannot be obtained, or the request is explicitly disallowed).

If the agent believes a blocking constraint exists, this pattern still applies when the trace does not communicate that constraint to the user prior to transferring.

### Example (Trace 10)

In Trace 10, the user asks to remove a passenger (“Ethan”) from reservation `H9ZU1C`. The agent issues a transfer to a human agent immediately, without attempting an in-scope next step such as enumerating the intended modification and requesting explicit confirmation to proceed.

## Pattern: Golden Mismatch (Incorrect Tool Arguments)

### Definition

The agent calls the correct tool, but chooses the wrong entity/option via its arguments (wrong flight/passenger/date/fare/reservation). The tool result is real but wrong versus the expected (“golden”) target; this is distinct from fabricating non-tool facts.

### Diagnostic Criteria

This pattern applies when all of the following are true:

1. The agent makes at least one tool call whose arguments contain a choice among multiple valid candidates (for example, selecting one itinerary from a search result set).
2. The trace indicates another candidate better matches the task expectation (for example, the golden specifies a different option), and the chosen candidate conflicts with that expectation.
3. The tool response reflects the chosen candidate, and the agent proceeds without detecting or correcting the mismatch.

## Pattern: Missing Required Tool Calls (Incomplete Action Set)

### Definition

The task oracle defines a minimum required set of tool calls in `info.task.actions`. The agent may perform other tool calls, but the attempt is incorrect if it omits any required action.

### Diagnostic Criteria

This pattern applies when all of the following are true:

1. The trace contains `info.task.actions`.
2. One or more required tool names in `info.task.actions[].name` do not appear in the trace’s assistant tool calls.
3. The trace ends without performing the missing required calls (i.e., there is no later correction).

### Notes

This pattern is about *completeness* of tool use, not about whether the agent’s narrative is plausible. In traces where the agent claims an action occurred but does not issue the required tool call, treat the missing call as the primary evidence of failure.

## Pattern: LLM User Failure (Simulated-User Drift / Noncompliance)

### Definition

The “user” is simulated by an LLM conditioned on `info.task.instruction`, but later user turns contradict that instruction (for example, accepting or requesting a disallowed alternative). The agent may act reasonably within the dialogue, yet the benchmark judges against the task-level instruction.

### Diagnostic Criteria

This pattern applies when all of the following are true:

1. `info.task.instruction` specifies constraints or conditional intent (for example, “if and only if X, do Y”).
2. A later simulated-user turn contradicts those constraints (for example, agrees to a non-task alternative proposed by the agent).
3. The agent follows the contradictory user turn, often via a state-mutating tool call.
4. The outcome signal indicates failure (for example, `reward = 0`).

### Notes

Treat the task-level instruction as authoritative. Record this as a simulator/user failure and do not recommend agent-side fixes unless the project explicitly intends agents to ignore user agreement that violates `info.task.instruction`.

### Examples
- The user LLM instruction specifies “cheapest economy,” but the assistant proposes `basic_economy` and the simulated user explicitly confirms “yes” to proceed with `basic_economy` (which the user is not supposed to do). This causes the assistant to perform the wrong actions.

# Triage Strategy
VERY IMPORTANT: Apply this strategy to find the true ROOT CAUSE of agent failures!

## Overview

A high-leverage triage method for this workload is to begin by comparing the agent’s realized tool calls to the oracle (“golden”) tool calls. Many failures reduce to a small number of tool-call discrepancies: missing expected calls, incorrect arguments to expected calls, or extra calls that incorrectly modify system state.

This strategy is intended to localize the earliest actionable failure point and then support a backward-chaining analysis to identify the underlying cause (for example, misunderstanding requirements vs. hallucinating entities).

## Procedure

1. Identify the oracle-required tool calls from `info.task.actions` (names and arguments).
2. Scan the trajectory for tool calls actually invoked by the agent:
   - confirm each oracle tool is present (no missing calls),
   - confirm ordering if ordering is semantically necessary for the task, and
   - validate that arguments match the oracle targets.
3. Check for additional tool calls beyond the oracle:
   - read-only extra calls (for example, searches/lookup) are typically safe,
   - state-mutating extra calls (for example, booking/canceling/updating) are typically disallowed unless present in the oracle, and
   - even if extra tool calls are permitted in principle, treat them as incorrect if they perturb the final system state away from the golden end state.
4. Interpret tool errors before attributing causality:
   - if a state-changing tool call returns an explicit error indicating the action did not execute (for example, insufficient funds), treat it as **non-mutating** and typically not a direct cause of correctness failure,
   - focus correctness analysis on the subset of **successful, state-mutating** tool calls when comparing to `info.task.actions`.
5. Once a mismatch is found, work backwards to determine why the agent made that choice:
   - missing call: did the agent prematurely terminate, transfer, refuse, or believe the task was complete?
   - wrong arguments: did the agent select the wrong candidate from tool results, misread constraints, or invent/hallucinate values?
   - extra state mutation: did the agent follow a non-oracle user request, misinterpret policy, or over-act beyond the task scope?

## Common Mismatch Types (Signatures)

### Missing oracle tool call

**Symptoms**
- One or more `info.task.actions[].name` does not appear as an agent tool call in `traj`.
- The assistant may claim completion in natural language without corresponding tool execution.

### Incorrect arguments to an oracle tool call

**Symptoms**
- The correct tool name is called, but key arguments target the wrong entity (wrong reservation, passenger, flight, date, etc.).
- Tool output reflects the wrong target and the agent does not correct course.

### Extra state-mutating tool call (end-state perturbation)

**Symptoms**
- Tool calls appear that are not in `info.task.actions` and that modify system state (book/cancel/update).
- The final state plausibly diverges from the golden end state that would result from executing only oracle actions.

