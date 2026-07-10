---
name: implement
description: Turn a TSP node into shipped code using tsp.implement.prepare for routing and the workflow-write tools for loop closure. Triggers on "implement <node-id>", "/tsp:implement", or any request to ship the work governed by a TSP node.
---

# tsp:implement

Turn a TSP node or subtree into actionable code work. TSP is the
source of truth: this skill creates an implementation brief, routes
atomic vs. non-atomic work, records progress back to TSP, and stops
when the plan says the node is blocked or under-refined.

This skill is harness-portable. It assumes only the MCP server's
tools are available; nothing about Claude Code, slash commands, or
plugin orchestration is baked into the tool contract.

## Capability

`implement <node-id>` — pick a node, build a routing decision, then
either run a single-pass implementation (atomic node) or walk the
subtree's atomic leaves in dependency order (non-atomic node), with
status writes flowing back to the canvas live through the plan's
Y.js room.

## Inputs

- `node_id`: the TSP node id to implement (required).
- `plan_id` and `branch_name`: optional; default to the session's
  prior `tsp.context.set` value, then to `branch_name="main"`.
- `session_id`: a stable id for this implementation session (the
  harness generates one if not supplied; reuse it across the loop).

## Required MCP tools

- `tsp.context.get` — resolve session-scoped plan and branch defaults.
- `tsp.implement.prepare` — build the brief and routing decision.
- `tsp.workflow.record_start` — claim the active session before work
  begins. This lights the live-session indicator on the canvas and makes
  closing the session your responsibility.
- `tsp.workflow.record_result` — **required** to close the session: write
  the terminal status (and optional in-progress checkpoints) the moment
  work finishes OR is abandoned. It is not an optional courtesy. Skipping
  it leaves the node falsely shown as live and stuck `in_progress` until a
  server-side inactivity timeout force-expires the session.

For non-atomic nodes the skill calls `tsp.implement.prepare` again
per leaf so each leaf gets its own brief and routing pass.

## Local filesystem responsibilities

The MCP server has no filesystem access. The harness:

1. Reads files referenced in the brief or surfaced by `code_refs`.
2. Edits files using its own write tools.
3. Runs tests using its own shell tool and parses the result.
4. Reports `touched_files`, `acceptance_results`, `tests_run`, and
   `summary` back through `record_result`.

## Tool sequence

```text
1. tsp.context.get
   - If no plan_id passed and no context set, ask the user once and
     call tsp.context.set so subsequent reads use it.
2. tsp.implement.prepare(selector=<node>, include_subtree=true,
                         require_acceptance_criteria=true)
3. Branch on routing:
   - blocked            -> show blockers and stop.
   - needs_refinement   -> show missing prerequisites and stop.
   - atomic_single_pass -> run the atomic flow.
   - subtree_sequential_leaves -> run the non-atomic flow.
```

### Atomic flow

```text
1. tsp.workflow.record_start(node, session_id, harness, summary).
   Keep the prepare brief in session context; the TSP node is the
   plan artifact, so never persist the brief as a repo file.
2. Read referenced files; make edits using harness tools.
3. Run tests covering the change.
4. tsp.workflow.record_result(
     node, session_id,
     result_status="complete" if AC satisfied and tests pass else
                   "in_progress" or "blocked",
     summary=<one-paragraph summary>,
     touched_files=<edited paths>,
     acceptance_results=[{criterion: ..., status: ...}],
     tests_run=<test ids or commands>,
     blockers=<external blockers>,
     next_steps=<follow-ups for the next session>,
   )
```

### Non-atomic flow

```text
For each leaf in routing.leaf_queue (in order):
  1. tsp.implement.prepare(selector=<leaf>, include_subtree=false,
                           require_acceptance_criteria=true).
  2. If leaf routes to blocked / needs_refinement, stop and surface
     why; do NOT skip ahead.
  3. tsp.workflow.record_start(leaf, session_id, harness, summary).
  4. Implement the leaf locally; run leaf-relevant tests.
  5. tsp.workflow.record_result(leaf, session_id, ...).

After every leaf completes, re-run tsp.implement.prepare on the
parent node and decide:
  - If parent acceptance criteria are now satisfied:
      tsp.workflow.record_result(parent, session_id, "complete", ...).
  - Otherwise:
      tsp.workflow.record_result(parent, session_id, "in_progress", ...)
      and call out unfinished AC.
```

## Blocking conditions

| Condition                                           | Skill behavior                                                                                        |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Routing is `blocked`                                | Render the blockers list (node ids, titles, statuses) and exit; do not start implementation.          |
| Routing is `needs_refinement`                       | Show the missing prerequisites; recommend running plan-side refinement before implementation.         |
| Subtree exceeds `max_leaf_tasks`                    | Surface the `subtree_too_large` warning; recommend a narrower target node or running next-actionable. |
| Atomic node with sparse code refs                   | Proceed but flag lower confidence in the brief; harness should search the repo for likely files.      |
| Acceptance criteria missing on best-effort override | Record `acceptance_results` as `unclear` for un-checkable items so handoff stays honest.              |

## Output format

The skill returns a short status report to the user:

```text
✅ atomic_single_pass complete
   node: <node_id> — <title>
   touched: <count> files
   tests: <count> run, <count> passed
   AC: <satisfied>/<total>
   next: <one-line summary or "—">
```

Or, for non-atomic flows, a per-leaf summary plus the final parent
status. Failures (blocked/needs_refinement) render the brief excerpt
and the specific missing-prereq list.

## State writes

Every `record_start` claims `Node.status = "in_progress"` and stores
the active session id; every `record_result` either keeps the status
in progress, transitions to `blocked`, or transitions to `complete`,
and clears the active session for terminal outcomes. Status writes
flow through `PlanRepository` so the canvas reflects the change
immediately via the plan's Y.js room.

A claimed session is a promise to close it. The active session id drives
the canvas live-session indicator, so every `record_start` must be paired
with a `record_result` on completion or abandonment — otherwise the node
keeps showing as live. As a safety net for crashed or cut-off sessions, a
server-side reaper auto-expires any session with no activity past an
inactivity TTL (clearing the indicator and stamping an expiry marker so
the auto-closed session stays distinguishable from a clean close), but
that is a backstop, not a substitute for closing the session yourself.
