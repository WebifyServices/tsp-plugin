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

Closing the loop is required, not optional. The active session id set by
`record_start` drives the canvas live-session indicator, so every claimed
session MUST be released with `record_result` on completion OR
abandonment; leaving it open keeps the node falsely shown as live. A
server-side reaper auto-expires sessions with no activity past an
inactivity TTL as a backstop for crashed or cut-off sessions, but
`handoff` is how you close the loop deliberately.

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

Collected from session state (the skill asks once when ambiguous):

- `session_id`: the stable id reused across `record_start` /
  `record_result` this session.
- `plan_id` and `branch_name`: from `tsp.context.get`.
- Active node ids: nodes `record_start` was called on this session
  with no terminal result yet.
- `previous_active_session_id` per active node: read from
  `NodeWorkflowState.active_session_id` via `tsp.workflow.get_state`,
  used to detect a stale handoff (a prior session that crashed
  without `record_result`). A non-null `expired_at` on the state means
  the server-side reaper already force-closed that prior session for
  inactivity — note it in the handoff so the takeover is explicit.
- A free-text summary, optional `next_steps`, `blockers`, and
  `local_context`.

## Required MCP tools

- `tsp.context.get` — resolve session-scoped plan and branch.
- `tsp.workflow.get_state` — read each active node's
  `NodeWorkflowState` for the stale-session check.
- `tsp.alignment.assess` (optional) — when the harness has snippets
  for an active node, align before writing so the recorded
  `acceptance_results` reflect what was checked.
- `tsp.workflow.record_result` — terminal status per active node.
- `tsp.session_note.create` — one free-text record covering the
  whole session and the affected node ids.

## Local filesystem responsibilities

The MCP server has no filesystem access. The harness:

1. Reads its own session log / commit history / git diff to compose
   the summary, files-changed list, and tests-run list.
2. Optionally reads recorded `code_refs` from earlier
   `tsp.node.execution_context` calls to pick alignment snippets.
3. Sends only bounded, focused content to MCP (alignment snippets
   cap at 20 entries totaling 20k content chars + 40k diff chars).

## Tool sequence

1. `tsp.context.get` (ask once if plan/branch unset).
2. Compose the local-state summary: touched files, tests run, diff
   state, active node ids, unresolved blockers and next steps.
3. For each active node: optionally `tsp.alignment.assess`, then
   `tsp.workflow.record_result` with the status from the policy below
   and the supplementary fields collected.
4. `tsp.session_note.create` once, covering all active node ids.
5. Show the final summary plus the recommended next command.

The alignment output is not wire-compatible with `record_result`
(`AlignmentCriterionResult` vs `AcceptanceCriterionCheck`, which
forbids extra fields), so the harness transforms each entry before
recording. The full step-by-step, the exact transform recipe, and the
extended blocking-condition table live in
`../docs/skill-behavior-spec.md` under `handoff`.

## Status-setting policy

| Local state                                             | Status written                                 |
| ------------------------------------------------------- | ---------------------------------------------- |
| All acceptance criteria satisfied AND tests pass        | `complete`                                     |
| Work started but AC incomplete, no external blocker     | `in_progress`                                  |
| Work cannot continue due to missing dependency/decision | `blocked`                                      |
| No reliable mapping between local state and a TSP node  | Skip `record_result`; create session note only |

The "no reliable mapping" row matters: if the agent worked on
something not governed by a node it called `record_start` on, do NOT
update an unrelated node's status to fake loop closure. Create a
session note that explains the mismatch so the next session resolves
it.

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
frame.

The skill never creates, deletes, or restructures nodes; it never
writes to generation branches (`branch_name="main"` is the V1
default and the only branch workflow writes target).
