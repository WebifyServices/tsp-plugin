# Harness-portable skill behavior spec

This document describes the V1 TSP coding-agent skills as a contract
that any MCP-aware harness can implement. The Claude Code plugin in
this repo is the first packaging; Cursor, Codex, Cline, and other
harnesses can re-implement these skills without changing the
underlying MCP tool contracts.

The spec is organized so each skill has the same section structure:
capability, inputs, required MCP tools, local filesystem
responsibilities, blocking conditions, output format, state writes.
That symmetry is intentional: a port to a new harness shouldn't have
to invent new shapes; it just translates the orchestration into the
new harness's native execution model.

For the canonical living source for each skill, see
`../skills/<name>/SKILL.md`.

## Common premises

- The MCP server has NO filesystem access. Every skill that needs to
  read or write files does so locally inside the harness, then ships
  bounded payloads to MCP.
- Selectors default to `branch_name="main"` everywhere. V1 is
  branch-naive; `tsp.branches.list` is deferred Future work.
- Session keying is via the `MCP-Session-Id` header, falling back to
  the bearer token id. Two concurrent sessions on one token get
  distinct context windows.
- Strict scope additivity: every tool's gate declares every scope
  its handler transitively touches. There is no implicit inheritance
  between scopes.
- All workflow writes target `Node.status` canonically through
  `PlanRepository`; supplementary fields land on the workflow store
  and are mirrored into the plan's `workflow_state` Y.Map for live
  canvas updates.

## Skills

### `implement` (load-bearing)

Capability: turn a TSP node or subtree into shipped code. Routes
atomic vs. non-atomic work via `tsp.implement.prepare`; runs the
local edit / test loop; closes per-leaf and per-parent results back
into the workflow store.

Required MCP tools: `tsp.context.get`, `tsp.implement.prepare`,
`tsp.workflow.record_start`, `tsp.workflow.record_result`.

Blocking conditions: routing returns `blocked` (show blockers, exit),
`needs_refinement` (show missing prerequisites, exit), or
`subtree_too_large` (recommend narrower target).

State writes: `record_start` claims `in_progress`; `record_result`
writes `complete` / `in_progress` / `blocked` based on AC + tests.

### `explain`

Capability: explain a topic against the plan. Names the governing
nodes, surfaces tree context, lists relationships, calls out gaps.

Required MCP tools: `tsp.context.get`,
`tsp.plan.semantic_search` (V1 keyword-backed),
`tsp.node.execution_context`, optionally `tsp.edges.list`.

Blocking conditions: no hits → fall back to `tsp.nodes.search`; only
coarse parent hits → explain at parent level.

State writes: none.

### `next-actionable`

Capability: recommend the next atomic leaves to work on. Ranked by
downstream critical-path length, then by status (`ready` before
`draft`), then by depth.

Required MCP tools: `tsp.context.get`, `tsp.plan.next_actionable`,
`tsp.node.execution_context` (for the primary recommendation).

Blocking conditions: no candidates → show `excluded_counts`; cycle
detected → render involved nodes and recommend DAG repair.

State writes: none.

### `align`

Capability: detect drift between code and plan. Harness reads files
locally and ships focused snippets; server returns a structured
drift report (verdict + findings + per-criterion checks).

Required MCP tools: `tsp.context.get`, `tsp.node.execution_context`,
`tsp.alignment.assess`. Optionally `tsp.workflow.record_result` to
record acceptance evidence on aligned verdicts.

Blocking conditions: missing AC → `MISSING_ACCEPTANCE_CRITERIA`
(skill recommends plan refinement); snippets exceed caps → summarize
locally; in V1 the LLM judge is disabled and the warning surfaces
that the verdict is from deterministic preflight only.

State writes: only when explicitly asked (`record_result` with the
alignment-derived `acceptance_results`).

### `impact`

Capability: reverse-map a file / diff / set of symbols to TSP nodes.
Used before refactors and during review.

Required MCP tools: `tsp.context.get`, `tsp.impact.map`. Optionally
`tsp.node.execution_context` for the top candidate.

Blocking conditions: no recorded code_refs anywhere → low-confidence
warning; `NO_CANDIDATES` → ask user for the governing node.

State writes: none.

### `debug`

Capability: from a stack trace / error / test failure, identify
TSP nodes likely governing the failing path.

Required MCP tools: `tsp.context.get`, `tsp.debug.map`. Optionally
`tsp.node.execution_context`.

Blocking conditions: unparseable stack frames → fall back to
keyword search over `error_text` (the server does this automatically
and warns).

State writes: none.

### `handoff`

Capability: end-of-session loop closure. Per active node, optionally
align then `record_result`; finally `session_note.create` covering
the whole session.

Required MCP tools: `tsp.context.get`, optionally
`tsp.alignment.assess`, then `tsp.workflow.record_result` and
`tsp.session_note.create`.

Blocking conditions: no reliable mapping between local state and a
TSP node → skip per-node `record_result`, create session note only.

Status-setting policy:

| Local state                                             | Status written    |
| ------------------------------------------------------- | ----------------- |
| All AC satisfied + tests pass                           | `complete`        |
| Work started but AC incomplete, no external blocker     | `in_progress`     |
| Work cannot continue due to missing dependency/decision | `blocked`         |
| No reliable node mapping                                | session note only |

State writes: `Node.status` through `record_result`; supplementary
fields and session note in the workflow store.

## Tool / scope / tier matrix

| Tool                                    | Tier | Scopes                             |
| --------------------------------------- | ---- | ---------------------------------- |
| `tsp.context.set/get/clear`             | Free | `plans:read`                       |
| `tsp.plans.list`                        | Free | `plans:read`                       |
| `tsp.prd.get`/`tsp.prd.sections`        | Free | `plans:read`                       |
| `tsp.node.get/parent/children/siblings` | Free | `plans:read`                       |
| `tsp.edges.list`                        | Free | `plans:read`                       |
| `tsp.nodes.search`                      | Free | `plans:read`                       |
| `tsp.tree.summary`                      | Free | `plans:read`                       |
| `tsp.node.execution_context`            | Pro  | `plans:read`                       |
| `tsp.subtree.get`                       | Pro  | `plans:read`                       |
| `tsp.node.blockers`                     | Pro  | `plans:read`                       |
| `tsp.plan.next_actionable`              | Pro  | `plans:read`                       |
| `tsp.plan.semantic_search`              | Pro  | `plans:read` + `semantic:read`     |
| `tsp.implement.prepare`                 | Pro  | `plans:read` + `implement:prepare` |
| `tsp.workflow.record_start`             | Pro  | `plans:read` + `workflow:write`    |
| `tsp.workflow.record_result`            | Pro  | `plans:read` + `workflow:write`    |
| `tsp.session_note.create`               | Pro  | `plans:read` + `workflow:write`    |
| `tsp.alignment.assess`                  | Pro  | `plans:read` + `alignment:run`     |
| `tsp.impact.map`                        | Pro  | `plans:read` + `impact:read`       |
| `tsp.debug.map`                         | Pro  | `plans:read` + `impact:read`       |

## Porting checklist

To re-implement this spec in another harness:

1. Read every `../skills/<name>/SKILL.md` for the canonical tool
   sequence and behaviors.
2. Wire the harness's MCP client to the same `/mcp` endpoint with
   the same scoped bearer token.
3. Translate "harness file reads / edits / tests" into the
   harness-native equivalents; do NOT introduce filesystem access
   on the MCP server side.
4. Preserve the bounded payload caps when sending `snippets`,
   `diff`, `error_text`, and `artifacts`.
5. Render the documented output formats so the user gets the same
   on-screen experience across harnesses.

The skill markdown files contain no Claude-specific constructs in the
tool contract; their narrative voice is portable. Some output
formats use emoji headers ("📋", "🧭", "🤝"); other harnesses can
strip or replace those without changing the behavior.
