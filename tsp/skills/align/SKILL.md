---
name: align
description: Detect drift between code and a TSP node's intent, scope, and acceptance criteria. Triggers on "align <node-id>", "/tsp:align", "check alignment", "is this code aligned with the plan", or any drift-detection request.
---

# tsp:align

Detect drift between implementation and plan. The MCP server has no
filesystem access; this skill reads files locally, picks focused
snippets, and ships bounded content to `tsp.alignment.assess`. The
server returns a structured drift report (verdict, findings,
acceptance-criterion checks) the harness renders for the user.

V1 alignment runs deterministic preflight only: keyword matching of
acceptance criteria against snippet content plus out-of-scope drift
detection. The future LLM-judge implementation plugs into the same
tool surface; the harness behavior stays identical.

## Capability

`align <node-id>` — pick a TSP node, gather code that implements (or
should implement) it, and ask the server whether the code matches
the documented intent, scope, and acceptance criteria.

## Inputs

- `node_id`: the TSP node to align against (required).
- `plan_id` and `branch_name`: optional; default to session context.
- `strictness`: `low` / `normal` / `high` (default `normal`). Higher
  strictness should make the harness pick more snippets and submit
  the diff alongside.
- `include_diff`: whether to include the current branch diff in the
  request (default true when there's a diff).

## Required MCP tools

- `tsp.context.get` — resolve session context.
- `tsp.node.execution_context` — fetch the node and its
  `code_refs` so the harness knows which files to look at.
- `tsp.alignment.assess` — the drift assessment.
- `tsp.workflow.record_result` (optional, on aligned verdicts) —
  record acceptance evidence directly off the alignment report.

## Local filesystem responsibilities

The harness:

1. Reads `code_refs` from execution context. These are the files the
   plan already knows about.
2. If `code_refs` is sparse, falls back to keyword search across the
   repo using terms from the node's title, intent, and scope.
3. Reads the candidate files and selects focused snippets to send.
4. Optionally captures the current diff (`git diff main...HEAD` or
   the unstaged diff) and includes it.
5. Enforces caps locally before sending: 20 snippets max, 20k content
   chars per snippet, 40k diff chars. Summarizes long files into
   smaller relevant chunks if needed.

## Tool sequence

```text
1. tsp.context.get (and optionally tsp.context.set if missing)
2. tsp.node.execution_context(selector=<node>, include_code_refs=true)
3. Local: read code_refs.path files; select snippets relevant to the
   node's intent / scope / acceptance criteria.
4. Local: read git diff if include_diff=true.
5. tsp.alignment.assess(
     selector=<node>,
     snippets=<bounded list>,
     diff=<bounded diff or None>,
     strictness=<low|normal|high>,
     include_recommendations=true,
   )
6. Render the drift report.
7. (Optional) On aligned verdicts where the user wants to close the
   loop:
     The alignment output's `acceptance_criteria` entries
     (`AlignmentCriterionResult`: `criterion`, `status`, `evidence`)
     are NOT wire-compatible with `record_result.acceptance_results`
     (`AcceptanceCriterionCheck`: `criterion`, `status`, `note`,
     `extra="forbid"`). Transform each entry before calling
     `record_result`:
       - keep `criterion` and `status` as-is
       - collapse `evidence` into a short `note` string (e.g.
         `"; ".join(e.excerpt or e.path for e in evidence[:3])`)
       - drop the raw `evidence` list (extra fields are rejected).
     tsp.workflow.record_result(
       selector=<node>,
       session_id=<this-session>,
       result_status="complete" if all AC satisfied else "in_progress",
       summary=<short summary>,
       acceptance_results=<transformed from alignment_assess output>,
       touched_files=<paths from snippets>,
     )
```

## Drift report rendering

Render the assessment as a section-structured summary the user can
scan in one screen. Lead with verdict + confidence; then per-finding
detail; then per-criterion status.

```text
🧭 Alignment: <verdict> (confidence <0.0-1.0>)
   plan: <plan_id>@<branch>
   node: <node_id> — <title>

Findings:
   ⚠️  warning  out_of_scope   Snippet at <path> mentions out-of-scope item '<...>'.
       evidence: <path>:<line range>
       recommendation: <if any>

Acceptance criteria:
   ✅ satisfied      <criterion>
   ❓ unclear        <criterion>
   ❌ missing        <criterion>
   🚫 contradicted   <criterion>

Plan fields missing: <list or "none">
Warnings: <list or "none">
```

For drifted verdicts, append a recommendation:

```text
Next: run /tsp:implement <node-id> to bring the code back in line, or
edit the plan to update the affected scope/acceptance criteria.
```

## Blocking conditions

| Condition                                 | Skill behavior                                                                                                                         |
| ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Missing acceptance criteria on the node   | Server returns `MISSING_ACCEPTANCE_CRITERIA`; the skill says alignment can only compare intent/scope and recommends plan refinement.   |
| No likely files found                     | Ask the user for the relevant files, or fall back to repo keyword search using node title + scope terms.                               |
| Snippets exceed size limit (20k per file) | Summarize the file locally (extract function bodies relevant to the node's terms) before sending, or skip very-long files with a note. |
| Diff exceeds 40k chars                    | Trim the diff to the file ranges that match the node's `code_refs`; if still over the cap, drop the diff and run snippets-only.        |
| `ASSESSMENT_MODEL_UNAVAILABLE` in V1      | Already happens by design; the warning lists "LLM judge disabled". Lower the user's expectation accordingly.                           |

## State writes

This skill never writes plan state on its own. It only writes via
`tsp.workflow.record_result` when the user explicitly asks to record
the alignment verdict as session evidence.
