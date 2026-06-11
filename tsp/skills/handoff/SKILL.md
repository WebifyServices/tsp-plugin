---
name: handoff
description: Close the loop on a coding session by writing status, touched files, blockers, next steps, and a free-text session note back to TSP. Triggers on "handoff", "/tsp:handoff", "wrap up", "finish session", "I'm done for the day", or any explicit end-of-session signal.
---

# tsp:handoff

Loop closure between a coding agent and the canvas. When a session
finishes (work shipped, blocked, or paused), `handoff` summarizes
local state and writes it back to TSP so the next session — human or
agent — picks up from a coherent state. The canvas reflects every
write through the plan's Y.js room within one tick.

This skill is pure orchestration over MCP tools. It owns no backend
files; persistence is handled by the MCP workflow-write tools
(`tsp.workflow.record_result` and `tsp.session_note.create`), which
write through `WorkflowStateService` and broadcast on the plan's
Y.js room.

## Capability

`handoff` — at end of session: per active node, optionally check
alignment, write a result status with the supplementary fields the
agent collected, then create one session note covering the whole
session for free-text handoff context. Output is a one-screen
summary the user can confirm before exit.

## Inputs

The skill collects these from session state (or asks the user once
when ambiguous):

- `session_id`: stable id used during the session for
  `record_start`/`record_result`. The harness reuses whatever id it
  used in `implement` or other writes earlier in the session.
- `plan_id` and `branch_name`: from `tsp.context.get`, default to the
  session's prior `tsp.context.set`.
- Active node ids: nodes the harness called `record_start` on during
  this session and has not yet recorded a terminal result for.
- `previous_active_session_id` per active node: the `session_id` that
  last called `record_start` on this node, read from the workflow
  state row returned by `tsp.workflow.get_state`
  (`NodeWorkflowState.active_session_id`). Used to detect a stale
  handoff where a prior session crashed without `record_result`.
- A free-text summary, optional `next_steps`, `blockers`, and
  `local_context` notes.

## Required MCP tools

- `tsp.context.get` — resolve session-scoped plan and branch.
- `tsp.workflow.get_state` — read each active node's
  `NodeWorkflowState` to recover `previous_active_session_id` (its
  `active_session_id` field) for the stale-session check below.
- `tsp.alignment.assess` (optional) — when the harness has snippets
  for an active node, run alignment before writing the result so the
  recorded `acceptance_results` reflect what was checked.
- `tsp.workflow.record_result` — terminal status per active node.
- `tsp.session_note.create` — one free-text record covering the
  whole session and the affected node ids.

## Local filesystem responsibilities

The MCP server has no filesystem access. The harness:

1. Reads its own session log / commit history / git diff to compose
   the summary, files-changed list, and tests-run list.
2. Optionally reads recorded `code_refs` from earlier
   `tsp.node.execution_context` calls to pick which files to include
   as alignment snippets.
3. Sends only bounded, focused content to MCP (the `record_result`
   inputs cap on the tool side; alignment snippets are capped to 20
   entries totaling 20k content chars + 40k diff chars).

## Tool sequence

```text
1. tsp.context.get
   - If no plan/branch context, ask the user once.
2. Local-state summary (harness-side):
   a. files changed (touched_files)
   b. tests run and their results
   c. current diff state
   d. active node ids (record_started this session, no terminal result yet)
   e. unresolved blockers and next steps
3. For each active node:
   a. If snippets exist for the node:
        tsp.alignment.assess(selector=<node>, snippets, diff?, strictness="normal")
        The alignment output's `acceptance_criteria` entries
        (`AlignmentCriterionResult`: `criterion`, `status`, `evidence`)
        are NOT wire-compatible with `record_result.acceptance_results`
        (`AcceptanceCriterionCheck`: `criterion`, `status`, `note`,
        `extra="forbid"`). The harness must transform each entry:
          - keep `criterion` and `status` as-is
          - collapse `evidence` into a short `note` string (e.g.
            `"; ".join(e.excerpt or e.path for e in evidence[:3])`)
          - drop the raw `evidence` list before passing to
            `record_result` (extra fields are rejected by the model).
   b. tsp.workflow.record_result(
        selector=<node>,
        session_id,
        result_status=<per status-setting policy below>,
        summary=<one paragraph>,
        touched_files=<edited paths>,
        acceptance_results=<from alignment or harness checks>,
        tests_run=<test ids or commands>,
        blockers=<external blockers>,
        next_steps=<follow-ups for the next session>,
      )
4. tsp.session_note.create(
     selector=<plan>,
     session_id,
     node_ids=<active node ids touched in this session>,
     summary=<full free-text handoff summary>,
     next_steps,
     blockers,
     local_context=<extra context the next session needs>,
   )
5. Show the final handoff summary plus the recommended next command.
```

## Status-setting policy

| Local state                                             | Status written                                 |
| ------------------------------------------------------- | ---------------------------------------------- |
| All acceptance criteria satisfied AND tests pass        | `complete`                                     |
| Work started but AC incomplete, no external blocker     | `in_progress`                                  |
| Work cannot continue due to missing dependency/decision | `blocked`                                      |
| No reliable mapping between local state and a TSP node  | Skip `record_result`; create session note only |

The "no reliable mapping" row is important: if the agent worked on
something that isn't governed by a node it called `record_start` on,
do NOT update an unrelated node's status to make the loop look
closed. Create a session note that explains the mismatch so the next
session can resolve it.

## Blocking conditions

| Condition                                                 | Skill behavior                                                                                                                                                              |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Multiple active nodes                                     | Loop over them; `record_result` per node, single `session_note.create` covering all of them.                                                                                |
| `previous_active_session_id` (the active node's `NodeWorkflowState.active_session_id`, read via `tsp.workflow.get_state`) differs from the current `session_id` | Surface the conflict to the user; the prior session crashed without `record_result`. Default to overwrite (soft-warn) but ask before doing so when the user is interactive. |
| `record_result` fails for one node                        | Continue with the others; surface the failed node id in the summary so the user can retry manually.                                                                         |
| Alignment is unavailable (`ASSESSMENT_MODEL_UNAVAILABLE`) | Skip alignment; record results based on harness checks alone. Note the limitation in the session note's local_context.                                                      |
| No active nodes touched this session                      | Skip per-node `record_result`; still create a session note so the canvas captures the session intent.                                                                       |

## Output format

```text
🤝 Handoff complete
   plan: <plan_id>@<branch>
   session: <session_id>
   nodes recorded:
     ✅ <node_id> — complete
     🟡 <node_id> — in_progress
     🚫 <node_id> — blocked
   session note: <note_id> (tsp://...)
   next: <one-line next-recommended-action or "—">
```

When errors occurred during the loop, append:

```text
⚠️  <count> issue(s):
     - <node_id>: record_result failed: <reason>
```

## State writes

This skill only writes to `Node.status` (canonically through
`PlanRepository`) and to `NodeWorkflowState` plus `SessionNote` in
the workflow-state store. All writes flow through the plan's Y.js
room so the canvas reflects the session's results within a single
frame of completing the handoff.

The skill never creates, deletes, or restructures nodes; it never
writes to generation branches (those are owned by the WebSocket
generation session, and `branch_name="main"` is the V1 default and
the only branch workflow writes target).
