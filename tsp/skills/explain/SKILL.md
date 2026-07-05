---
name: explain
description: Return a coherent answer about a TSP plan topic — which nodes govern it, how they relate, where the gaps are. Triggers on "explain <topic>", "/tsp:explain", "what does the plan say about X", or any plan-orientation request.
---

# tsp:explain

Return a coherent answer about a TSP plan topic: which nodes govern
the topic, how they relate, and where the gaps are.
`tsp.plan.semantic_search` surfaces the most relevant nodes for the
topic; enrich the top hits with their execution context to ground
the answer.

## Capability

`explain <topic>` — find the TSP nodes most relevant to a topic and
render a one-screen explanation: short answer, governing nodes, tree
context, semantic relationships, and any obvious gaps.

## Inputs

- `topic`: the question or topic string (required).
- `plan_id` and `branch_name`: optional; default to session context.
- `max_results`: how many top hits to enrich (default 5).

## Required MCP tools

- `tsp.context.get` — resolve session-scoped plan and branch.
- `tsp.plan.semantic_search` — find relevant nodes (`purpose=explain`).
- `tsp.node.execution_context` — pull intent + scope + relationships
  for the top 3-5 hits.
- `tsp.edges.list` (optional) — get explicit relationship detail when
  the search hits aren't already linked by depends_on.

## Local filesystem responsibilities

None. Explain is purely plan-side; the harness reads no local files.

## Tool sequence

```text
1. tsp.context.get
2. tsp.plan.semantic_search(
     selector=<plan>, query=<topic>, purpose="explain",
     max_results=<5..10>,
   )
3. For top 3-5 hits:
   # Semantic search returns hits whose `node` is a NodeRef (id/title).
   # `tsp.node.execution_context` requires a NodeSelector (plan_id +
   # node_id + optional branch_name). Build one per hit before
   # calling — passing the raw NodeRef fails Pydantic validation.
   tsp.node.execution_context(
       selector=NodeSelector(
           plan_id=<plan>, branch_name=<branch>,
           node_id=hit.node.node_id,
       ),
       include_ancestors=true,
       include_dependencies=true,
       include_dependents=true,
   )
4. (Optional) tsp.edges.list(selector=<plan>) for cross-cutting edges.
5. Render the explanation.
```

## Output format

```text
🧭 Explain: <topic>
   plan: <plan_id>@<branch>

Short answer:
  <one-paragraph synthesis grounded in the top hits>

Governing nodes:
  - <node_id> — <title>  [<score>]
       intent: <intent>
       in-scope: <comma-separated>

Tree context:
  - <ancestor chain or parent>
  - <siblings if relevant>

Semantic relationships:
  - depends_on: <list>
  - relates_to: <list>

Open gaps / low-confidence areas:
  - <gap 1>
  - <gap 2>
```

## Blocking conditions

| Condition               | Skill behavior                                                                                 |
| ----------------------- | ---------------------------------------------------------------------------------------------- |
| No hits                 | Fall back to `tsp.nodes.search` with the same terms. Say no strong match found if still empty. |
| Only coarse parent hits | Explain at the parent level and list its children to inspect next.                             |
| Edges sparse            | Say relationship semantics are under-modeled and lean on tree context (parent + ancestors).    |

## State writes

This skill never writes plan state.
