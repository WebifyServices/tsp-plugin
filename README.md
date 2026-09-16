# TSP plugin

Tree-Structured Planning (TSP) is a canvas for planning software systems that coding agents can read from and write back to. This plugin installs in **Claude Code**, **Cursor**, and **Codex**. One package gives you eight skills plus a connection to the hosted TSP MCP server.

Install once, then your agent can author plans, implement off a node, record progress, reverse-map a diff to the nodes that govern it, and hand a session off when you stop. All three harnesses get the same skills against the same MCP server. Only the install step and how you invoke a skill change. The skill contracts are written to port to any MCP-aware harness (Cline and others re-implement against the same spec), so the three supported here are peers, not a primary plus ports.

> This repository is a published mirror. The canonical source lives in the private TSP monorepo and this artifact is synced out on each release, so the skills and manifests here are generated, not hand-edited. See [CONTRIBUTING](./CONTRIBUTING.md) before opening a PR.

## What you need first

- One of [Claude Code](https://code.claude.com), [Cursor](https://cursor.com) (2.5 and later), or [Codex](https://developers.openai.com/codex), whichever you already use.
- A TSP account at [treestructuredplanning.com](https://treestructuredplanning.com). Sign in through your browser. No API keys or tokens to paste. Cursor and Codex still need the `tsp` server pointed at (see the install sections below). Only the credential handoff is automatic.

The MCP server URL is production (`https://treestructuredplanning.com/mcp/`) everywhere. If you run TSP somewhere else, change the `url` in the Cursor or Codex config below. Claude Code's marketplace install is pinned to this repo's shipped `tsp/.mcp.json`, so pointing it elsewhere means installing from a fork or clone with that file edited, not the marketplace command.

## Install

Pick your harness. Each is a quick, one-time setup. These steps match the in-app [Connect your agent](https://treestructuredplanning.com/docs/drive-with-your-agent/connect-your-agent) guide.

### Claude Code

Add the marketplace and install the plugin:

```text
/plugin marketplace add WebifyServices/tsp-plugin
/plugin install tsp@tsp-plugins
```

The first command registers this repo as a Claude Code plugin marketplace. The second installs the `tsp` plugin from it. These `/plugin` commands are Claude Code's (terminal or IDE). In Claude Desktop or claude.ai, add `https://treestructuredplanning.com/mcp/` as a custom connector instead. Claude Code may ask you to reload the session so the new skills and the MCP server register. On first use it discovers the authorization server from the MCP server's response and drives the OAuth 2.1 + PKCE browser sign-in for you. The result is stored and refreshed automatically. If the `tsp` server later shows as needing authentication, run `/mcp`, select `tsp`, and choose **Reauthenticate**.

Skills surface as `/tsp:<skill>` slash commands. Type `/tsp:` to autocomplete the eight. A finished sign-in is not the whole check: ask Claude to list your TSP plans. A plan list, even an empty one, means the read path is live.

### Cursor

Cursor 2.5 and later. Two steps: install the plugin, then connect.

**Install the plugin.** Run `/add-plugin` and point it at `WebifyServices/tsp-plugin`, or install **TSP** from the [Cursor Marketplace](https://cursor.com/marketplace). This brings in the eight skills.

**Connect.** Add the `tsp` server to `.cursor/mcp.json` in your project, or `~/.cursor/mcp.json` for every project:

```json
{
	"mcpServers": {
		"tsp": {
			"url": "https://treestructuredplanning.com/mcp/"
		}
	}
}
```

A bare `url` is enough. Cursor shows the server as needing sign-in, opens your browser to authorize it, then stores and refreshes the token on its own. Don't add an `Authorization` header or a client id. A static auth block suppresses the automatic sign-in. If Cursor sits on "needs login" without opening the browser, use the bridge form, which runs the sign-in itself:

```json
{
	"mcpServers": {
		"tsp": {
			"command": "npx",
			"args": ["-y", "mcp-remote", "https://treestructuredplanning.com/mcp/"]
		}
	}
}
```

Skills surface through Cursor's `/` skill menu. Cursor applies them automatically when they're relevant. After sign-in, ask Cursor to list your TSP plans.

### Codex

Three steps: install the plugin, add the server, sign in.

**Install.** Open `/plugins`, add the `WebifyServices/tsp-plugin` marketplace, and install `tsp`. This brings in the eight skills.

**Add the server** in `~/.codex/config.toml` (or `$CODEX_HOME/config.toml`):

```toml
[mcp_servers.tsp]
url = "https://treestructuredplanning.com/mcp/"
```

Or: `codex mcp add tsp --url https://treestructuredplanning.com/mcp/`.

**Sign in once:**

```bash
codex mcp login tsp
```

That opens your browser to authorize and caches the result. Start Codex normally afterward. Adding the config alone leaves you unsigned in until you run `codex mcp login tsp` once. If your Codex build can't reach the URL directly, use the bridge form instead (keep only one `tsp` block):

```toml
[mcp_servers.tsp]
command = "npx"
args = ["-y", "mcp-remote", "https://treestructuredplanning.com/mcp/"]
```

If `codex mcp login` errors on an older build, enable Codex's streamable-HTTP client by adding a `[features]` block with `rmcp_client = true`, then retry.

Skills surface through `/skills`. After login, ask Codex to list your TSP plans.

## What you get

Eight skills, each triggered by natural language or its slash form. However you connect, you get the same set wired to your plans. Only the way you reach them differs by harness (Claude Code's `/tsp:` commands, Cursor's `/` skill menu, Codex's `/skills`):

| Skill             | Trigger                               | What it does                                                                                                                      |
| ----------------- | ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `plan-author`     | `/tsp:plan-author`, "create a plan"   | Author or repair plans at generative-grade structure: tree-first decomposition, minimal justified edges, complete node contracts. |
| `implement`       | `/tsp:implement <node-id>`            | Turn a node or subtree into shipped code; route atomic vs. non-atomic work; record results.                                       |
| `next-actionable` | `/tsp:next-actionable`, "what's next" | Recommend the next node to pick up, ranked by critical path and dependency readiness.                                             |
| `explain`         | `/tsp:explain <topic>`                | Explain a topic against the plan: which nodes govern it, how they relate, where the gaps are.                                     |
| `align`           | `/tsp:align <node-id>`                | Detect drift between your code and a node's intent, scope, and acceptance criteria.                                               |
| `impact`          | `/tsp:impact <path>`                  | Reverse-map a file or diff to the nodes that govern it, before a refactor or review.                                              |
| `debug`           | `/tsp:debug <stack-trace>`            | From an error or failing test, identify which nodes likely govern the failing path.                                               |
| `handoff`         | `/tsp:handoff`, "wrap up"             | Close a session: write status, touched files, blockers, and next steps back to the canvas.                                        |

The MCP server has no filesystem access by design. Every skill that needs your code reads it locally inside your agent and ships only bounded snippets to the server, which returns structured results the harness renders.

## Learn more

- Product: [treestructuredplanning.com](https://treestructuredplanning.com)
- In-app setup guide: [Connect your agent](https://treestructuredplanning.com/docs/drive-with-your-agent/connect-your-agent)
- The skill contracts in [`tsp/skills/`](./tsp/skills/) and the harness-portable spec in [`tsp/docs/skill-behavior-spec.md`](./tsp/docs/skill-behavior-spec.md).

## License

[MIT](./LICENSE).
