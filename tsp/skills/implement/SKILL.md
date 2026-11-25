---
name: implement
description: Turn a TSP node into shipped code using tsp_implement_prepare for routing and the workflow-write tools for loop closure. Triggers on "implement <node-id>", "/tsp:implement", or any request to ship the work governed by a TSP node.
---

# tsp:implement

First follow [Worker lifecycle](../../docs/application-sessions.md).

TSP is the source of truth: stop when the plan says the node is blocked
or under-refined. The skill assumes only the MCP server's tools.

## Capability

`implement <node-id>` — pick a node, build a routing decision, then
either run a single-pass implementation (atomic node) or walk the
subtree's atomic leaves in dependency order (non-atomic node), with
status writes showing on the canvas live.

## Inputs

- `node_id`: the TSP node id to implement (required).
- `plan_id`: optional; defaults to the session's
  prior `tsp_context_set` value.
- `agent_session_id`: a stable id for this implementation session (the
  harness generates one if not supplied; reuse it across the loop).

## Required MCP tools

- `tsp_context_get` — resolve session-scoped plan default.
- `tsp_implement_prepare` — build the brief and routing decision.
- `tsp_attempt_progress` — record a short milestone with a fresh idempotency key.
- `tsp_workflow_record_result` — record each node's result on the caller's attempt,
  including its summary, touched files, checks, tests and code pointer. This also
  retains the node-status projection until the status-write cutover.
- `tsp_session_renew` — renew every ~2 minutes while coding or testing
  ([why](../../docs/application-sessions.md#staying-visible-while-you-work)).
- `tsp_session_note_create` — exactly one per implementation session,
  after the last `record_result`, written as `handoff` writes it.

For non-atomic nodes the skill calls `tsp_implement_prepare` again
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
1. tsp_context_get
   - If no plan_id passed and no context set, ask the user once and
     call tsp_context_set so subsequent reads use it.
2. tsp_implement_prepare(node_id=<node>, include_subtree=true,
                         require_acceptance_criteria=true)
   # plan_id is optional everywhere once tsp_context_set has run.
3. Branch on routing:
   - blocked            -> show blockers and stop.
   - needs_refinement   -> show missing prerequisites and stop.
   - atomic_single_pass -> run the atomic flow.
   - subtree_sequential_leaves -> run the non-atomic flow.
```

### Atomic flow

```text
1. Keep the prepare brief in session context; the TSP node is the plan artifact.
   Report milestones with tsp_attempt_progress(milestone=<short line>,
                                               idempotency_key=<fresh key>).
   Pass node_ids only as State writes says.
2. Read referenced files; make edits using harness tools.
3. Run tests covering the change.
4. tsp_workflow_record_result(
     node_id=<node>, agent_session_id=<session>,
     result_status="complete" if AC satisfied and tests pass else
                   "in_progress" or "blocked",
     summary=<one-paragraph summary>,
     touched_files=<edited paths>,
     acceptance_results=[{criterion_id: "ac-1", status: ...}, ...],
     tests_run=<test ids or commands>,
     blockers=<external blockers>,
     next_steps=<follow-ups for the next session>,
   )
   # acceptance_results link by the stable criterion_id (ac-1,
   # ac-2, …) shown alongside acceptance_criteria on reads — never
   # re-type criterion text. new_status is the committed status;
   # transition_note explains implicit transitions. Retroactive
   # recording needs only this one call.
5. tsp_session_note_create(agent_session_id=<session>, node_ids=<every
   node recorded>, ...) once, after record_result: the note `handoff`
   step 4 defines (../handoff/SKILL.md, ../../docs/skill-behavior-spec.md):
   what was done, decisions taken with the person, what is next.
```

### Non-atomic flow

```text
For each leaf in routing.leaf_queue (in order):
  1. tsp_implement_prepare(node_id=<leaf>, include_subtree=false,
                           require_acceptance_criteria=true).
  2. If leaf routes to blocked / needs_refinement, stop and surface
     why; do NOT skip ahead.
  3. Report each milestone with tsp_attempt_progress(
       milestone=<short line>, idempotency_key=<fresh key>).
     Pass node_ids only as State writes says.
  4. Implement the leaf locally; run leaf-relevant tests.
  5. tsp_workflow_record_result(node_id=<leaf>, agent_session_id=<session>, ...).

After every leaf completes, re-run tsp_implement_prepare on the
parent node. Report on the parent only when it is inside your attempt scope;
otherwise leave the parent rollup to the plan and summarize the leaf results.
For an in-scope parent, decide:
  - If parent acceptance criteria are now satisfied:
      tsp_workflow_record_result(node_id=<parent>, result_status="complete", ...).
  - Otherwise:
      tsp_workflow_record_result(node_id=<parent>, result_status="in_progress", ...)
      and call out unfinished AC.
Then step 5 of the atomic flow, once for the whole run.
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

The session handle authorizes every write to the caller's attempt. A Build seed
carries a claim; pass it to tsp_session_register. A non-null `attempt` in the
response marks a bound session: stay inside its scope and omit `node_ids` on
every progress call (any you pass must equal the whole scope as a set). A
hand-run session (`attempt=null`) passes every node it will work on in
`node_ids` on its first progress only, which opens its attempt. Milestones
append to the attempt; `record_result` maps complete to finished, in_progress
to partial, and blocked to blocked, then updates `Node.status` after the fenced
report commits. Retry a result with the same idempotency key.

There is no exclusive leaf claim, active-run lease or waiting-for-person hold.
Independent attempts may cover the same node. Continue advances the attempt's
agent epoch and fences the previous agent. Finish with one session note, then
follow the worker lifecycle when the worker shuts down.
