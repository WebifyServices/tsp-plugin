---
name: debug
description: Identify which TSP nodes likely govern a failing path from a stack trace, error message, or test failure. Triggers on "debug <stack-trace>", "/tsp:debug", "which node owns this failure", or any incident-investigation request.
---

# tsp:debug

First follow [Worker lifecycle](../../docs/application-sessions.md).

Identify which TSP nodes likely govern a failing path. `debug` is an
impact variant optimized for errors, stack traces, test failures, and
runtime symptoms. The MCP server has no filesystem access; the
harness extracts frames and symbols locally and sends bounded
content to `tsp_debug_map`.

`tsp_debug_map` uses the same mapping as `tsp_impact_map`. Stack
frames are parsed for paths and symbols, then matched by keyword,
recorded code references and edge traversal. When frames are
unparseable, it falls back to keyword search over the error text.

## Capability

`debug <stack-trace-or-error>` — given a stack trace, error message,
or test failure, list TSP nodes most likely governing the failing
path, with debug steps the user can take next.

## Inputs

- `error_text`: the error message or test failure (required).
- `stack_frames`: a list of stack-frame strings (optional but
  recommended).
- `related_artifacts`: file excerpts the harness already has open
  that may relate to the failure (optional).
- `plan_id`: optional; defaults to session context.

## Required MCP tools

- `tsp_context_get` — resolve session context.
- `tsp_debug_map` — the reverse mapping.
- `tsp_node_execution_context` (optional) — pull intent + AC for the
  top candidate so the rendered debug plan stays grounded in plan
  state.

## Local filesystem responsibilities

The harness:

1. Captures the error text (verbatim, up to 40k chars).
2. Extracts stack frames as raw strings; the server parses common
   Python / Node / generic conventions, deduplicates the paths and
   symbols, and when nothing parses it warns and falls back to
   keyword search over `error_text` automatically.
3. Optionally reads code at file paths the frames reference and
   includes excerpts as `related_artifacts` (cap: 20 entries).

## Tool sequence

```text
1. tsp_context_get
2. (Optional) Local: read related files and capture excerpts. Never include credentials, tokens, API keys or .env contents in an excerpt or diff you send; skip those lines.
3. tsp_debug_map(
     plan_id=<plan>,  # optional once tsp_context_set has run
     error_text=<error>,
     stack_frames=<list of frame strings>,
     related_artifacts=<bounded list>,
     max_candidates=10,
   )
4. (Optional, top candidate): tsp_node_execution_context(node_id=<top_node_id>)
5. Render the candidate list with rationale, likely failure mode, and
   suggested debug steps.
```

## Output format

```text
🐞 Debug map for: <one-line error summary>
   plan: <plan_id>

Candidates (top <N>):
  1. <node_id> — <title>  [confidence <0.0-1.0>]
       rationale: <one-line>
       likely failure mode: <hint or "—">
       suggested debug steps:
         - <step 1>
         - <step 2>
  ...

Related contracts:
  - <edge_id>: <from> -> <to> (<type>)

Suggested commands:
  - /tsp:align <node-id>
  - /tsp:impact <path>
```

## Blocking conditions

| Condition                      | Skill behavior                                                        |
| ------------------------------ | --------------------------------------------------------------------- |
| No stack frames                | Search over the error text via keyword matching against node text.    |
| No code refs                   | Use path terms and node titles via keyword.                           |
| Frames present but unparseable | Service warns; falls back to error-text keyword search automatically. |
| Multiple equally likely nodes  | Return read order based on dependency direction and edge types.       |

## State writes

This skill never writes plan state. It surfaces information; the user
or `/tsp:implement` writes status separately.
