# TSP plugin for Claude Code

Tree-Structured Planning (TSP) is a canvas for planning software systems that coding agents can read from and write back to. This is the installable Claude Code plugin: it bundles seven skills with a connection to the hosted TSP MCP server, so a single install gives Claude Code a working loop against your plans, route implementation work off a node, record progress back to the canvas, reverse-map a diff to the nodes that govern it, and hand a session off cleanly when you stop.

The skills are written so their behavior ports to any MCP-aware harness (Cursor, Codex, Cline). Claude Code is the first packaging; the underlying tool contracts are the same everywhere.

> This repository is a published mirror. The canonical source lives in the private TSP monorepo and this artifact is synced out on each release, so the skills and manifests here are generated, not hand-edited. See [CONTRIBUTING](./CONTRIBUTING.md) before opening a PR.

## What you need first

- [Claude Code](https://code.claude.com) installed.
- A TSP account at [treestructuredplanning.com](https://treestructuredplanning.com). The plugin connects to the hosted MCP server and authenticates you through your browser; nothing is configured by hand and no token is ever pasted in.

## Install

```text
/plugin marketplace add WebifyServices/tsp-plugin
/plugin install tsp@tsp-plugins
```

The first command registers this repo as a Claude Code plugin marketplace. The second installs the `tsp` plugin from it. On first use, Claude Code discovers the authorization server from the MCP server's response and drives the OAuth 2.1 + PKCE sign-in flow for you in the browser.

The MCP server URL is already pinned to production (`https://treestructuredplanning.com/mcp/`), so there is nothing to set. If you are running TSP somewhere else, clone this repo, change the `url` in `tsp/.mcp.json`, and register your clone as the marketplace instead: `/plugin marketplace add /path/to/your/clone`.

## What you get

Seven skills, each triggered by natural language or its slash form:

| Skill             | Trigger                               | What it does                                                                                  |
| ----------------- | ------------------------------------- | --------------------------------------------------------------------------------------------- |
| `implement`       | `/tsp:implement <node-id>`            | Turn a node or subtree into shipped code; route atomic vs. non-atomic work; record results.   |
| `next-actionable` | `/tsp:next-actionable`, "what's next" | Recommend the next node to pick up, ranked by critical path and dependency readiness.         |
| `explain`         | `/tsp:explain <topic>`                | Explain a topic against the plan: which nodes govern it, how they relate, where the gaps are. |
| `align`           | `/tsp:align <node-id>`                | Detect drift between your code and a node's intent, scope, and acceptance criteria.           |
| `impact`          | `/tsp:impact <path>`                  | Reverse-map a file or diff to the nodes that govern it, before a refactor or review.          |
| `debug`           | `/tsp:debug <stack-trace>`            | From an error or failing test, identify which nodes likely govern the failing path.           |
| `handoff`         | `/tsp:handoff`, "wrap up"             | Close a session: write status, touched files, blockers, and next steps back to the canvas.    |

The MCP server has no filesystem access by design. Every skill that needs your code reads it locally inside Claude Code and ships only bounded snippets to the server, which returns structured results the harness renders.

## Learn more

- Product: [treestructuredplanning.com](https://treestructuredplanning.com)
- The skill contracts in [`tsp/skills/`](./tsp/skills/) and the harness-portable spec in [`tsp/docs/skill-behavior-spec.md`](./tsp/docs/skill-behavior-spec.md).

## License

[MIT](./LICENSE).
