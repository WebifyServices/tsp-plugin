---
name: next-actionable
description: Recommend the next TSP node to work on, ranked by downstream critical path and dependency readiness. Triggers on "next", "/tsp:next-actionable", "what should I work on", "what's the next thing to ship", or queue/todo questions about the plan.
---

# tsp:next-actionable

The queue / todo capability for a TSP plan. Looks at the dependency
DAG, status of each node, and downstream critical-path length, then
recommends what a coding agent should pick up next. Bounded results,
O(V+E) server-side; never returns the whole plan in one shot.

## Capability

`next-actionable` — list the top N atomic leaves a session can start
right now (no incomplete predecessors, status not complete/blocked),
ranked by how much work they unlock downstream. Returns one primary
recommendation plus alternates.

## Inputs

- `plan_id` and `branch_name`: optional; default to session context.
- `workstream`: optional filter for backend / frontend / etc.
- `include_in_progress`: include nodes currently `in_progress` in
  the candidate set (default false).
- `include_non_atomic`: include non-atomic nodes (default false).
- `max_results`: top-N cap (default 5, max 25).

## Required MCP tools

- `tsp.context.get` — resolve session-scoped plan and branch.
- `tsp.plan.next_actionable` — the recommendation tool.
- `tsp.node.execution_context` — enrich the primary recommendation
  with intent, blockers, and code refs so the user can decide
  immediately.

## Local filesystem responsibilities

None. Recommendations are computed entirely from plan state.

## Tool sequence

```text
1. tsp.context.get
2. tsp.plan.next_actionable(
     selector=<plan>, workstream=<filter or null>,
     include_in_progress=false, include_non_atomic=false,
     max_results=<N>,
   )
3. For the primary recommendation:
   # `primary.node` is a NodeRef. `tsp.node.execution_context` needs
   # a NodeSelector (plan_id + node_id + optional branch_name); build
   # one before calling — passing the raw NodeRef fails validation.
   tsp.node.execution_context(
       selector=NodeSelector(
           plan_id=<plan>, branch_name=<branch>,
           node_id=primary.node.node_id,
       ),
       include_dependencies=true,
       include_code_refs=true,
   )
4. Render the recommendation.
```

## Output format

```text
📋 Next actionable
   plan: <plan_id>@<branch>

▶ Primary:
  <node_id> — <title>
     critical path: <length>
     status: <status>
     intent: <intent>
     in-scope: <comma-separated>
     recommended: /tsp:implement <node_id>

Alternates:
  2. <node_id> — <title>  [cpl=<n>, status=<s>]
  3. <node_id> — <title>  [cpl=<n>, status=<s>]
  ...

Excluded counts:
  - complete: <n>
  - in_progress: <n>
  - non_atomic: <n>
  - has_incomplete_predecessor: <n>
  - workstream_mismatch: <n>
```

## Blocking conditions

| Condition                | Skill behavior                                                                                       |
| ------------------------ | ---------------------------------------------------------------------------------------------------- |
| No candidates            | Show excluded_counts so the user can see why; recommend relaxing filters or refining the plan.       |
| Dependency cycle warning | Server raises `DEPENDENCY_CYCLE_DETECTED`; render involved node ids and recommend repairing the DAG. |
| Sparse statuses          | `tsp.plan.next_actionable` ranks `ready` candidates ahead of `draft` and may return `draft` entries in the tail when `max_results` exceeds the number of `ready` leaves. Surface the lower-confidence `draft` rows as alternates (not the primary) when the top of the list is a draft candidate. |

## State writes

This skill never writes plan state. It surfaces recommendations; the
user (or `/tsp:implement`) starts work separately, which writes
`Node.status` through `record_start`.
