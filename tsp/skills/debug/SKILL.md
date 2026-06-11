---
name: debug
description: Identify which TSP nodes likely govern a failing path from a stack trace, error message, or test failure. Triggers on "debug <stack-trace>", "/tsp:debug", "which node owns this failure", or any incident-investigation request.
---

# tsp:debug

Identify which TSP nodes likely govern a failing path. `debug` is an
impact variant optimized for errors, stack traces, test failures, and
runtime symptoms. The MCP server has no filesystem access; the
harness extracts frames and symbols locally and sends bounded
content to `tsp.debug.map`.

V1 backing reuses the same `ImpactMappingService` as `tsp.impact.map`.
Stack frames are parsed for paths and symbols, then fed into the
shared keyword + recorded-code-ref + edge-traversal pipeline. When
frames are unparseable, the service falls back to keyword search over
the error text.

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
- `plan_id` and `branch_name`: optional; default to session context.

## Required MCP tools

- `tsp.context.get` — resolve session context.
- `tsp.debug.map` — the reverse mapping.
- `tsp.node.execution_context` (optional) — pull intent + AC for the
  top candidate so the rendered debug plan stays grounded in plan
  state.

## Local filesystem responsibilities

The harness:

1. Captures the error text (verbatim, up to 40k chars).
2. Extracts stack frames as raw strings; the server parses them.
3. Optionally reads code at file paths the frames reference and
   includes excerpts as `related_artifacts` (cap: 20 entries).

## Tool sequence

```text
1. tsp.context.get
2. (Optional) Local: read related files and capture excerpts.
3. tsp.debug.map(
     selector=<plan>,
     error_text=<error>,
     stack_frames=<list of frame strings>,
     related_artifacts=<bounded list>,
     max_candidates=10,
   )
4. (Optional, top candidate): tsp.node.execution_context(selector=<top_node>)
5. Render the candidate list with rationale, likely failure mode, and
   suggested debug steps.
```

## Stack-frame parsing

The server's parser handles common conventions:

- Python: `File "path/to/file.py", line N, in symbol`
- Node/JS: `at symbol (path:line:col)`
- Generic: any path containing a known file extension followed by
  `:line` or `:line:col`.

Symbols and paths are deduplicated and used to seed keyword search
against node text. When no frames parse, the server emits a warning
and falls back to keyword search over `error_text`.

## Output format

```text
🐞 Debug map for: <one-line error summary>
   plan: <plan_id>@<branch>

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

| Sparse data                    | Fallback                                                              |
| ------------------------------ | --------------------------------------------------------------------- |
| No stack frames                | Search over the error text via keyword matching against node text.    |
| No code refs                   | Use path terms and node titles via keyword.                           |
| Frames present but unparseable | Service warns; falls back to error-text keyword search automatically. |
| Multiple equally likely nodes  | Return read order based on dependency direction and edge types.       |

## State writes

This skill never writes plan state. It surfaces information; the user
or `/tsp:implement` writes status separately.
