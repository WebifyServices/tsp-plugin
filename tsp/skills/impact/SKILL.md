---
name: impact
description: Reverse-map a file or diff to the TSP nodes that govern it. Triggers on "impact <path>", "/tsp:impact", "what nodes does this change affect", before-refactor or before-review intent.
---

# tsp:impact

First follow [Worker lifecycle](../../docs/application-sessions.md).

Reverse-map code artifacts (files, diffs, symbols) to TSP nodes via
`tsp.impact.map`. Useful before refactors, during review, and before
merging work that might cross semantic boundaries.

Ranking draws on three sources, strongest first: paths previously
recorded by `record_result`, keyword overlap with node text, and
one-hop `depends_on` neighbors. The mapping improves over time as
`implement` and `handoff` record touched files; cold-start results
are lower-confidence and lean on keyword + edge traversal.

## Capability

`impact <path>` — given a file path (or diff, or set of symbols), list
the TSP nodes most likely to govern that code, ranked by confidence,
with the in-scope items that apply and the risks of touching them.

## Inputs

- `path`: a file path (required if no diff or symbols are provided).
- `diff`: a unified diff (optional; bounded at 40k chars).
- `symbols`: explicit symbols / function names (optional).
- `mode`: `pre_refactor` / `pre_review` / `general` (default).
- `plan_id` and `branch_name`: optional; default to session context.

## Required MCP tools

- `tsp.context.get` — resolve session context.
- `tsp.impact.map` — the reverse mapping.
- `tsp.node.execution_context` (optional) — fetch full intent and
  acceptance criteria for the top candidate so the harness can render
  them inline.

## Local filesystem responsibilities

The harness:

1. Reads the file at `path` (or the diff for the current branch) and
   captures a bounded excerpt.
2. Optionally extracts symbols (function / class names) using its own
   parser or simple regex.
3. Sends only bounded content to MCP: 20 artifacts max, 20k content
   chars per artifact, 40k diff chars.

## Tool sequence

```text
1. tsp.context.get
2. (Optional) Local: read the file or capture the diff; extract symbols.
3. tsp.impact.map(
     plan_id=<plan>,  # optional once tsp.context.set has run
     artifacts=[CodeArtifactSpec(path=..., content_excerpt=..., symbols=...)],
     mode=<pre_refactor|pre_review|general>,
     max_candidates=10,
     include_dependency_neighbors=true,
   )
4. (Optional, top candidate): tsp.node.execution_context(node_id=<top_node_id>)
5. Render the candidate list with rationale, governing scope, and risks.
```

## Output format

```text
🎯 Impact map for <path or "current diff">
   plan: <plan_id>@<branch>

Candidates (top <N>):
  1. <node_id> — <title>  [<relation>, confidence <0.0-1.0>]
       rationale: <one-line rationale>
       in-scope: <comma-separated scope items or "—">
       risks: <comma-separated risks or "—">
  ...

Dependency neighbors:
  - <edge_id>: <from> -> <to> (<type>)

Suggested checks:
  - /tsp:align <node-id>
  - /tsp:impact current diff
```

For empty results, the skill surfaces the warning verbatim and
recommends a `tsp.nodes.search` query built from the artifact's path
and symbol terms.

## Blocking conditions

| Condition                          | Skill behavior                                                                                                 |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| No code refs recorded anywhere     | Warn that confidence is low; the keyword + edge fallback still produces candidates but the user should verify. |
| Artifact exceeds 20k content cap   | Trim to the function bodies and import lines; fall back to symbols-only if still over.                         |
| `NO_CANDIDATES`                    | Show the warning verbatim and ask the user to identify the governing node manually, then offer to record it.   |
| Multiple equally likely candidates | Render all of them in rank order; pick the top one for the optional execution-context enrichment.              |

## State writes

This skill never writes plan state. It surfaces information; the user
or `/tsp:implement` writes status separately.
