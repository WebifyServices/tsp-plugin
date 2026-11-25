---
name: handoff
description: Close the loop on a coding session by writing status, touched files, blockers, next steps, and a free-text session note back to TSP. Triggers on "handoff", "/tsp:handoff", "wrap up", "finish session", "I'm done for the day", or any explicit end-of-session signal.
---

# tsp:handoff

First follow [Worker lifecycle](../../docs/application-sessions.md).

Loop closure between a coding agent and the canvas. When a session
finishes (work shipped, blocked, or paused), `handoff` summarizes
local state and writes it back to TSP so the next session — human or
agent — picks up from a coherent state. The canvas shows every write
live.

Closing the loop is required: record each worked node's result on the
caller's attempt and leave one session note. The session handle authorizes
these writes; no leaf claim or active-run lease needs to be released.
Follow the worker lifecycle to close the application session at shutdown.

This skill is pure orchestration over MCP tools. It owns no backend
files; `tsp_workflow_record_result` writes the attempt report and retained
node-status projection, and `tsp_session_note_create` records handoff context.

## Capability

`handoff` — at end of session: per active node, judge the node's
acceptance criteria against what the session actually did (diff,
tests run, files touched), write a result status with the
supplementary fields the agent collected, then create one session
note covering the whole session for free-text handoff context.
Output is a one-screen summary the user can confirm before exit.
The judging is yours, done locally — never invoke a metered
operation from this skill.

## Inputs

Collected from session state (the skill asks once when ambiguous):

- `session_handle`: the current worker's private credential, kept in runtime state.
- `agent_session_id`: the immutable public session id returned by registration.
- `plan_id`: from `tsp_context_get`.
- Active node ids: nodes worked on within the attempt's scope with no terminal
  result yet. If the credential is refused or superseded, stop and tell the
  person; Continue issues the next agent a new epoch on this attempt.
- A free-text summary, optional `next_steps`, `blockers`, and
  `local_context`.

## Required MCP tools

- `tsp_context_get` — resolve session-scoped plan.
- `tsp_node_execution_context` (optional) — re-fetch a node's
  acceptance criteria when they aren't already in session context.
- `tsp_workflow_record_result` — terminal status per active node.
- `tsp_session_note_create` — one free-text record covering the
  whole session and the affected node ids.

## Local filesystem responsibilities

The MCP server has no filesystem access. The harness:

1. Reads its own session log / commit history / git diff to compose
   the summary, files-changed list, and tests-run list.
2. Judges each active node's acceptance criteria against those
   artifacts locally: `satisfied` / `missing` / `unclear` /
   `contradicted` per criterion, bound by meaning rather than
   vocabulary, with removed diff lines never counting as evidence
   of `satisfied`.

## Tool sequence

1. `tsp_context_get` (ask once if the plan is unset).
2. Compose the local-state summary: touched files, tests run, diff
   state, active node ids, unresolved blockers and next steps.
3. For each active node: judge its acceptance criteria locally, then
   `tsp_workflow_record_result` with the status from the policy below,
   `acceptance_results` composed directly as
   `{criterion_id, status, note}` entries (positional ids `ac-1`,
   `ac-2`, …; never re-type criterion text; a short evidence string
   goes in `note`), the supplementary fields collected, and a fresh
   `idempotency_key` (e.g. a UUID) minted once per node's result. If
   the call fails or times out, retry with the SAME key: a result that
   already landed is replayed (`replayed: true`), never recorded twice;
   a `CONFLICT` with `result_in_flight` means the first call is still
   landing, so wait a moment and retry the same call again.
   The session handle routes the result to your attempt; do not pass
   a run id or start a second reporting path.
4. `tsp_session_note_create` once, covering all active node ids.
5. Show the final summary plus the recommended next command.

The full step-by-step and the extended blocking-condition table live
in `../docs/skill-behavior-spec.md` under `handoff`.

## Status-setting policy

| Local state                                             | Status written                                 |
| ------------------------------------------------------- | ---------------------------------------------- |
| All acceptance criteria satisfied AND tests pass        | `complete`                                     |
| Work started but AC incomplete, no external blocker     | `in_progress`                                  |
| Work cannot continue due to missing dependency/decision | `blocked`                                      |
| No reliable mapping between local state and a TSP node  | Skip `record_result`; create session note only |

The "no reliable mapping" row matters: if the agent worked on
something not governed by a node in its attempt scope, do NOT
update an unrelated node's status to fake loop closure. Create a
session note that explains the mismatch so the next session resolves
it.

## Output format

```text
🤝 Handoff complete
   plan: <plan_id>
   session: <agent_session_id>
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

This skill only writes each active node's `status` and workflow
state through `tsp_workflow_record_result`, and one session note
through `tsp_session_note_create`. The canvas reflects the session's
results as soon as they land.

The skill never creates, deletes, or restructures nodes.
