# Harness-portable skill behavior spec

This document describes the TSP coding-agent skills as a contract
that any MCP-aware harness can implement. The plugin in this repo
ships that contract in Claude Code, Cursor, and Codex as equal
packagings; Cline and other MCP-aware harnesses can re-implement
these skills without changing the underlying MCP tool contracts.

The spec is organized so each skill has the same section structure:
capability, inputs, required MCP tools, local filesystem
responsibilities, blocking conditions, output format, state writes.
That symmetry is intentional: a port to a new harness shouldn't have
to invent new shapes; it just translates the orchestration into the
new harness's native execution model.

For the canonical living source for each skill, see
`../skills/<name>/SKILL.md`.

## Common premises

- Tools are named the way the server lists them, with underscores (for example `tsp_node_get`). A client may put a prefix before each name; match a tool by the end of its name.
- The MCP server has NO filesystem access. Every skill that needs to
  read or write files does so locally inside the harness, then ships
  bounded payloads to MCP.
- Agents work on the plan itself, and every skill below reads and writes
  that one plan.
- Every worker follows [Worker session lifecycle](./application-sessions.md):
  register independently, pass the active handle in ordinary tool payloads,
  renew or resume its lease, and close at worker shutdown. Workers sharing
  OAuth credentials retain independent defaults across transport reconnections.
  Renewal, not planning activity, keeps a worker's lease and live cursor: a
  skill that spends minutes in local code or tests renews about every two
  minutes rather than issuing reads or notes to stay visible.
- Strict scope additivity: there is no implicit inheritance
  between scopes.

## Skills

### `implement` (load-bearing)

Capability: turn a TSP node or subtree into shipped code. Routes
atomic vs. non-atomic work via `tsp_implement_prepare`; runs the
local edit / test loop; closes per-leaf and per-parent results back
onto the caller's attempt.

Required MCP tools: `tsp_context_get`, `tsp_implement_prepare`,
`tsp_attempt_progress`, `tsp_workflow_record_result`.

Blocking conditions: routing returns `blocked` (show blockers, exit),
`needs_refinement` (show missing prerequisites, exit), or
`subtree_too_large` (recommend narrower target).

State writes: a Build seed carries a claim for `tsp_session_register`; a
non-null `attempt` in the response identifies a bound session. Omit `node_ids`
on every `tsp_attempt_progress`, including the first milestone. For a hand-run
session registered with `attempt=null`, pass all nodes it will work on in
`node_ids` on its first progress only; this opens its attempt and appends a
milestone. Supplied node ids on a bound session must equal its entire scope
as a set; a mismatch is refused with the bound nodes named.
`tsp_attempt_progress` appends milestones and
`record_result` records each node's report on the caller's attempt, retaining
the `Node.status` projection. The session handle authorizes the write; no run
id or exclusive leaf claim is needed. Close the application session at worker
shutdown after reporting and leaving the session note.

### `explain`

Capability: explain a topic against the plan. Names the governing
nodes, surfaces tree context, lists relationships, calls out gaps.

Required MCP tools: `tsp_context_get`,
`tsp_plan_semantic_search` (keyword-backed),
`tsp_node_execution_context`, optionally `tsp_edges_list`.

Blocking conditions: no hits → fall back to `tsp_nodes_search`; only
coarse parent hits → explain at parent level.

State writes: none.

### `next-actionable`

Capability: recommend the next atomic leaves to work on. Ranked by
downstream critical-path length, then by status (`ready` before
`draft`), then by depth.

Required MCP tools: `tsp_context_get`, `tsp_plan_next_actionable`,
`tsp_node_execution_context` (for the primary recommendation).

Blocking conditions: no candidates → show `excluded_counts`; cycle
detected → render involved nodes and recommend DAG repair.

State writes: none.

### `align`

Capability: detect drift between code and plan. The harness fetches
the node's intent, scope, and acceptance criteria through free reads,
reads the code locally, and judges each criterion itself (`satisfied`
/ `missing` / `unclear` / `contradicted`, bound by meaning; removed
diff lines never count as `satisfied` evidence). It renders a
structured drift report (verdict + findings + per-criterion checks).
No metered operation is ever invoked: `tsp_alignment_assess` remains
a free deterministic keyword check.

Required MCP tools: `tsp_context_get`, `tsp_node_execution_context`.
Optionally `tsp_workflow_record_result` to record acceptance evidence
on aligned verdicts.

Blocking conditions: missing AC → alignment can only compare
intent/scope, confidence is low, recommend plan refinement; no likely
files found → ask the user or keyword-search the repo from node
title/scope terms.

State writes: only when explicitly asked (`record_result` with the
harness-judged `acceptance_results`).

### `impact`

Capability: reverse-map a file / diff / set of symbols to TSP nodes.
Used before refactors and during review.

Required MCP tools: `tsp_context_get`, `tsp_impact_map`. Optionally
`tsp_node_execution_context` for the top candidate.

Blocking conditions: no recorded code_refs anywhere → low-confidence
warning; `NO_CANDIDATES` → ask user for the governing node.

State writes: none.

### `debug`

Capability: from a stack trace / error / test failure, identify
TSP nodes likely governing the failing path.

Required MCP tools: `tsp_context_get`, `tsp_debug_map`. Optionally
`tsp_node_execution_context`.

Blocking conditions: unparseable stack frames → fall back to
keyword search over `error_text` (the server does this automatically
and warns).

State writes: none.

### `handoff`

Capability: end-of-session loop closure. Per active node, judge the
acceptance criteria locally against the session's artifacts, then
`record_result`; finally `session_note.create` covering the whole
session. No metered operation is ever invoked.

Required MCP tools: `tsp_context_get`, then
`tsp_workflow_record_result` and `tsp_session_note_create`.

Blocking conditions: no reliable mapping between local state and a
TSP node → skip per-node `record_result`, create session note only.

Status-setting policy:

| Local state                                             | Status written    |
| ------------------------------------------------------- | ----------------- |
| All AC satisfied + tests pass                           | `complete`        |
| Work started but AC incomplete, no external blocker     | `in_progress`     |
| Work cannot continue due to missing dependency/decision | `blocked`         |
| No reliable node mapping                                | session note only |

Tool sequence (detailed):

1. `tsp_context_get` — resolve plan; ask the user once if unset.
2. Compose the local-state summary (harness-side): touched files,
   tests run + results, current diff, active node ids (in this attempt's scope
   with no terminal result yet), unresolved blockers and
   next steps.
3. For each active node:
    1. Judge each acceptance criterion locally against the session's artifacts (diff, tests run, files read): `satisfied` / `missing` / `unclear` / `contradicted`, bound by meaning rather than vocabulary; removed diff lines never count as `satisfied` evidence. Compose `acceptance_results` entries directly as `{criterion_id, status, note}` — positional `criterion_id` (`ac-1`, `ac-2`, …), a short evidence string in `note`, never re-typed criterion text (the server resolves the id to the canonical wording).
    2. `tsp_workflow_record_result(node_id, session_handle, idempotency_key, result_status, summary, touched_files, acceptance_results, tests_run, blockers, next_steps)` — mint `idempotency_key` once per result and resend it on retry (a recorded key replays, never records twice; `result_in_flight` means the first call is still landing, so retry the same call shortly); the response's `new_status` is what the status rules actually committed; `transition_note` states any implicit transition and `incomplete_dependencies` names blocking node ids. The session handle routes this result to the caller's attempt; no run id or prior `record_start` is needed.
4. `tsp_session_note_create(plan_id?, agent_session_id, node_ids, summary, next_steps, blockers, local_context)` — one note covering the whole session and every active node id. The note says what was done and the decisions taken with the person (`summary`), and what is next (`next_steps`, `blockers`). `implement` closes with this same note, once, after its last `record_result`.
5. Show the final handoff summary plus the recommended next command.

Extended blocking conditions:

| Condition                                                  | Behavior                                                                                              |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Multiple active nodes                                      | Loop `record_result` per node; one `session_note.create` covers all of them.                          |
| The attempt credential is refused or superseded            | Stop and tell the person; Continue gives the next agent a new epoch on the same attempt.              |
| `record_result` fails for one node                         | Continue with the others; surface the failed node id in the summary so the user can retry manually.   |
| Acceptance criteria not judgeable (no diff/tests to weigh) | Record from harness checks alone; note the limitation in the session note's `local_context`.          |
| No active nodes touched this session                       | Skip per-node `record_result`; still create a session note so the canvas captures the session intent. |

State writes: node `status` through `record_result`; supplementary
fields and session note in TSP.

### `plan-author`

Capability: author or repair plan structure over the write tools at
the generative system's quality bar, exactly as lean as the
requirements permit (coverage floor: every stated surface gets an
owning node; leanness ceiling: nothing the source does not state).
Tree-first decomposition (one honest specificity step per level,
child count an output of the semantics — never a target, one
partition axis per level, siblings comparable in kind not size,
semantic atomicity); a complete contract on every node (intent
strictly narrower than the parent's, concrete in_scope noun phrases,
out_of_scope never empty — each entry the owning node's exact title,
root non-goals on the root only, observable acceptance criteria);
and a minimal edge set — zero edges is acceptable, far fewer edges
than nodes, one edge per node pair, no parent/ancestor edges, no
transitive depends_on echoes, no sequencing-as-dependency, every
kept edge noted with what flows. Authors the tree first
(`tsp_nodes_create` batches without edges), then one deliberate edge
pass over the finished tree, then a boundary pass over every node's
out_of_scope, a coverage pass back over the source, and self-review
with `tsp_tree_summary` + `tsp_edges_list`.

Required MCP tools: `tsp_context_get/set`, `tsp_plan_create`,
`tsp_nodes_create`, `tsp_node_update`, `tsp_edge_create`,
`tsp_edge_delete`, `tsp_edges_list`, `tsp_tree_summary`. Optionally
`tsp_node_generate` (metered) to delegate a decomposition or
refinement, reviewed by the same rules.

Blocking conditions: source material too thin → ask, never invent
children; plan/node caps or tier errors → surface and narrow scope
explicitly; existing plan violates the standard → offer a repair pass
(edge deletions first) instead of mirroring the pattern.

State writes: plan, node, and edge structure only; no workflow
writes — delivery status stays `draft` until implementation sessions
claim nodes.

### `plan-repo`

Capability: author a plan of an existing repository from its code.
The root is the repository, the first level is the units its
manifests declare, and each node below is one cohesive part of the
code. Each node's contract describes the code as it is today, and its
in_scope entries name the paths they cover. Relationships come from
real imports and interface implementations, under the same edge rules
as `plan-author`. Shows the person the top two levels before writing.
Optionally plans one path under the node that already owns it.

Inputs: an optional path inside the repository.

Required MCP tools: `tsp.context.get/set`, `tsp.plan.create`,
`tsp.nodes.create`, `tsp.node.update`, `tsp.edge.create`,
`tsp.edge.delete`, `tsp.edges.list`, `tsp.tree.summary`.

Local filesystem responsibilities: reads the repository's manifests,
READMEs, source and import statements locally. Sends only plan
fields to MCP.

Blocking conditions: no readable source → say so and create nothing;
two decompositions tie and the layout is mixed → ask before writing;
plan cap or tier errors → surface them and narrow the path.

State writes: plan, node, and edge structure only; no workflow or
attempt writes, so no node is marked complete.

## Tool / scope / tier matrix

| Tool                                    | Tier | Scopes                             |
| --------------------------------------- | ---- | ---------------------------------- |
| `tsp_context_set/get/clear`             | Free | `plans:read`                       |
| `tsp_plans_list`                        | Free | `plans:read`                       |
| `tsp_prd_get`/`tsp_prd_sections`        | Free | `plans:read`                       |
| `tsp_node_get/parent/children/siblings` | Free | `plans:read`                       |
| `tsp_edges_list`                        | Free | `plans:read`                       |
| `tsp_nodes_search`                      | Free | `plans:read`                       |
| `tsp_tree_summary`                      | Free | `plans:read`                       |
| `tsp_node_execution_context`            | Pro  | `plans:read`                       |
| `tsp_subtree_get`                       | Pro  | `plans:read`                       |
| `tsp_node_blockers`                     | Pro  | `plans:read`                       |
| `tsp_plan_next_actionable`              | Pro  | `plans:read`                       |
| `tsp_plan_semantic_search`              | Pro  | `plans:read` + `semantic:read`     |
| `tsp_implement_prepare`                 | Pro  | `plans:read` + `implement:prepare` |
| `tsp_workflow_record_start`             | Pro  | `plans:read` + `workflow:write`    |
| `tsp_workflow_record_result`            | Pro  | `plans:read` + `workflow:write`    |
| `tsp_session_note_create`               | Pro  | `plans:read` + `workflow:write`    |
| `tsp_alignment_assess`                  | Pro  | `plans:read` + `alignment:run`     |
| `tsp_impact_map`                        | Pro  | `plans:read` + `impact:read`       |
| `tsp_debug_map`                         | Pro  | `plans:read` + `impact:read`       |
| `tsp_plan_create` / `tsp_plan_update`   | Pro  | `plans:read` + `plans:write`       |
| `tsp_node_create/update/delete/move`    | Pro  | `plans:read` + `structure:write`   |
| `tsp_nodes_create`                      | Pro  | `plans:read` + `structure:write`   |
| `tsp_edge_create` / `tsp_edge_delete`   | Pro  | `plans:read` + `structure:write`   |
| `tsp_node_generate` (metered)           | Pro  | `plans:read` + `generation:run`    |

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
