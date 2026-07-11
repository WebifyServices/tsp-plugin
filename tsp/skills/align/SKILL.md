---
name: align
description: Detect drift between code and a TSP node's intent, scope, and acceptance criteria. Triggers on "align <node-id>", "/tsp:align", "check alignment", "is this code aligned with the plan", or any drift-detection request.
---

# tsp:align

Detect drift between implementation and plan. The MCP server has no
filesystem access and runs no LLM for alignment: **you are the
judge**. This skill fetches the node's intent, scope, and acceptance
criteria through free MCP reads, reads the code locally, and rules
each criterion itself. It costs the user no TSP credits.

`tsp.alignment.assess` exists as a free deterministic keyword check
(no LLM, no credits) but binds criteria by vocabulary, not meaning —
your own reading of the code is strictly better evidence. A
server-side LLM judge as an explicit, priced second opinion is
post-MVP (#2037); never invoke a metered operation from this skill.

## Capability

`align <node-id>` — pick a TSP node, gather the code that implements
(or should implement) it, and rule whether the code matches the
documented intent, scope, and acceptance criteria.

## Inputs

- `node_id`: the TSP node to align against (required).
- `plan_id` and `branch_name`: optional; default to session context.
- `strictness`: `low` / `normal` / `high` (default `normal`). Higher
  strictness means reading more candidate files and always including
  the diff in what you weigh.

## Required MCP tools

- `tsp.context.get` — resolve session context.
- `tsp.node.execution_context` — fetch the node's intent, scope,
  acceptance criteria, and `code_refs` (the files the plan already
  knows about).
- `tsp.workflow.record_result` (optional, on aligned verdicts) —
  record acceptance evidence.

## Judging procedure

1. `tsp.context.get` (and optionally `tsp.context.set` if missing).
2. `tsp.node.execution_context(node_id=<node>, include_code_refs=true)`.
3. Read the `code_refs` files. If `code_refs` is sparse, keyword-search
   the repo using terms from the node's title, intent, and scope.
4. Read the current diff when relevant (`git diff main...HEAD` or the
   unstaged diff).
5. Rule every acceptance criterion, referenced by its positional
   `criterion_id` (`ac-1` is the first criterion, then `ac-2`, …), with
   exactly one status:
    - `satisfied` — the code observably produces the outcome the
      criterion describes; cite `path:line` evidence.
    - `missing` — nothing implements it.
    - `unclear` — partial or ambiguous implementation.
    - `contradicted` — the code actively conflicts with it.

    Bind criteria to code **by meaning, not vocabulary**: read the
    criterion as an outcome and the code as the mechanism, even when
    they share no words. In a diff, removed (`-`) lines are never
    evidence a criterion is satisfied — a removal may instead support
    `contradicted`.

6. Surface scope findings: code implementing an `out_of_scope` item is
   a `warning`; code conflicting with the node's `intent` is
   `blocking`.
7. Choose a verdict — `aligned` (criteria satisfied, no blocking
   findings), `needs_review` (unclear rulings or warnings), `drifted`
   (missing/contradicted criteria or blocking findings) — plus your
   confidence (0.0–1.0).
8. Render the drift report (format below).
9. Optionally record the verdict (next section) when the user wants to
   close the loop.

## Recording the verdict

`record_result.acceptance_results` links rulings by the positional
`criterion_id` (`ac-1`, `ac-2`, …). Never re-type criterion text; the
server resolves the id to the canonical wording. Each entry accepts
only `criterion_id` / `status` / `note` — put a short evidence string
(e.g. `"src/foo.py:42 — handles the retry"`) in `note`.

```text
tsp.workflow.record_result(
  node_id=<node>,
  session_id=<this-session>,
  result_status="complete" if all AC satisfied else "in_progress",
  summary=<short summary>,
  acceptance_results=[{criterion_id, status, note}, ...],
  touched_files=<paths you judged>,
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
   ⚠️  warning  out_of_scope   <path> implements out-of-scope item '<...>'.
       evidence: <path>:<line range>
       recommendation: <if any>

Acceptance criteria:
   ✅ satisfied      <criterion>
   ❓ unclear        <criterion>
   ❌ missing        <criterion>
   🚫 contradicted   <criterion>
```

For drifted verdicts, append a recommendation:

```text
Next: run /tsp:implement <node-id> to bring the code back in line, or
edit the plan to update the affected scope/acceptance criteria.
```

## Blocking conditions

| Condition                               | Skill behavior                                                                                           |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Missing acceptance criteria on the node | Say alignment can only compare intent/scope; confidence is low; recommend plan refinement.               |
| No likely files found                   | Ask the user for the relevant files, or fall back to repo keyword search using node title + scope terms. |
| Very large surface (many files)         | Prioritize `code_refs` and diff-touched files; say which files you did not read.                         |

## State writes

This skill never writes plan state on its own. It only writes via
`tsp.workflow.record_result` when the user explicitly asks to record
the alignment verdict as session evidence.
