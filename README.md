# TSP plugin

Tree-Structured Planning (TSP) is a canvas for planning software systems that coding agents can read from and write back to. This is the installable TSP plugin: one multi-harness package that installs in **Claude Code**, **Cursor**, and **Codex**, bundling seven skills with a connection to the hosted TSP MCP server. A single install gives your agent a working loop against your plans — route implementation work off a node, record progress back to the canvas, reverse-map a diff to the nodes that govern it, and hand a session off cleanly when you stop.

All three harnesses get the identical skills talking to the same hosted MCP server; only the install step and how you invoke a skill differ. The skill contracts are written to port to any MCP-aware harness (Cline and others re-implement against the same spec), so the three supported here are peers, not a primary plus ports.

> This repository is a published mirror. The canonical source lives in the private TSP monorepo and this artifact is synced out on each release, so the skills and manifests here are generated, not hand-edited. See [CONTRIBUTING](./CONTRIBUTING.md) before opening a PR.

## What you need first

- One of [Claude Code](https://code.claude.com), [Cursor](https://cursor.com) (2.5 or later), or [Codex](https://developers.openai.com/codex) — whichever you already use.
- A TSP account at [treestructuredplanning.com](https://treestructuredplanning.com). The plugin connects to the hosted MCP server and authenticates you through your browser: there are no API keys or tokens to paste anywhere. Cursor and Codex still need the `tsp` server pointed at, as their install sections below cover; only the credential handoff is automatic.

The MCP server URL is production (`https://treestructuredplanning.com/mcp/`) everywhere. If you run TSP somewhere else, Cursor and Codex just need the `url` in the config below changed to your endpoint. Claude Code's marketplace install is pinned to this repo's shipped `tsp/.mcp.json`, so pointing it elsewhere means installing from a fork or clone with that file edited, not the marketplace command.

## Install

Pick your harness — each is a quick, one-time setup. The install steps here match the in-app [Connect your agent](https://treestructuredplanning.com/docs/drive-with-your-agent/connect-your-agent) guide.

### Claude Code

Add the marketplace and install the plugin:

```text
/plugin marketplace add WebifyServices/tsp-plugin
/plugin install tsp@tsp-plugins
```

The first command registers this repo as a Claude Code plugin marketplace; the second installs the `tsp` plugin from it. Claude Code may ask you to reload the session so the new skills and the MCP server register. On first use it discovers the authorization server from the MCP server's response and drives the OAuth 2.1 + PKCE browser sign-in for you; the result is stored and refreshed automatically. If the `tsp` server later shows as needing authentication, run `/mcp`, select `tsp`, and choose **Reauthenticate**.

Skills surface as `/tsp:<skill>` slash commands — type `/tsp:` to autocomplete the seven.

### Cursor

Cursor's plugin system (2.5 and later) packages the skills and the MCP connection together, so setup is two steps.

**Install the plugin.** Run `/add-plugin` in Cursor and point it at the `WebifyServices/tsp-plugin` repo, or install **TSP** from the [Cursor Marketplace](https://cursor.com/marketplace). This brings in the seven skills.

**Connect the MCP server.** Add the `tsp` server to `.cursor/mcp.json` in your project (or `~/.cursor/mcp.json` to make it available everywhere):

```json
{
	"mcpServers": {
		"tsp": {
			"url": "https://treestructuredplanning.com/mcp/"
		}
	}
}
```

A bare `url` is all you need: Cursor shows the server as needing sign-in, opens your browser to authorize it, and then stores and refreshes the token on its own. Don't add an `Authorization` header or a client id — a static auth block suppresses the automatic sign-in. If Cursor sits on "needs login" without opening the browser, use the bridge form, which runs the sign-in itself:

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

Skills surface through Cursor's `/` skill menu, and Cursor applies them automatically when they're relevant.

### Codex

Codex installs the plugin from a marketplace, then connects to the same MCP endpoint — three steps.

**Install the plugin.** In Codex, open `/plugins`, add the `WebifyServices/tsp-plugin` marketplace, and install `tsp`. This brings in the seven skills.

**Add the MCP server.** Put the `tsp` server in `~/.codex/config.toml` (or `$CODEX_HOME/config.toml`):

```toml
[mcp_servers.tsp]
url = "https://treestructuredplanning.com/mcp/"
```

Or add it from the terminal in one line: `codex mcp add tsp --url https://treestructuredplanning.com/mcp/`.

**Sign in once** from your terminal:

```bash
codex mcp login tsp
```

That opens your browser to authorize and caches the result; start Codex normally afterward. Adding the block alone won't connect you until you run `codex mcp login tsp` once. If your Codex build can't reach the URL directly, use the bridge form instead (keep only one `tsp` block):

```toml
[mcp_servers.tsp]
command = "npx"
args = ["-y", "mcp-remote", "https://treestructuredplanning.com/mcp/"]
```

If `codex mcp login` errors on an older build, enable Codex's streamable-HTTP client by adding a `[features]` block with `rmcp_client = true`, then retry.

Skills surface through `/skills`.

## What you get

Seven skills, each triggered by natural language or its slash form. However you connect, you get the same set wired to your plans — only the way you reach them differs by harness (Claude Code's `/tsp:` commands, Cursor's `/` skill menu, Codex's `/skills`):

| Skill             | Trigger                               | What it does                                                                                  |
| ----------------- | ------------------------------------- | --------------------------------------------------------------------------------------------- |
| `implement`       | `/tsp:implement <node-id>`            | Turn a node or subtree into shipped code; route atomic vs. non-atomic work; record results.   |
| `next-actionable` | `/tsp:next-actionable`, "what's next" | Recommend the next node to pick up, ranked by critical path and dependency readiness.         |
| `explain`         | `/tsp:explain <topic>`                | Explain a topic against the plan: which nodes govern it, how they relate, where the gaps are. |
| `align`           | `/tsp:align <node-id>`                | Detect drift between your code and a node's intent, scope, and acceptance criteria.           |
| `impact`          | `/tsp:impact <path>`                  | Reverse-map a file or diff to the nodes that govern it, before a refactor or review.          |
| `debug`           | `/tsp:debug <stack-trace>`            | From an error or failing test, identify which nodes likely govern the failing path.           |
| `handoff`         | `/tsp:handoff`, "wrap up"             | Close a session: write status, touched files, blockers, and next steps back to the canvas.    |

The MCP server has no filesystem access by design. Every skill that needs your code reads it locally inside your agent and ships only bounded snippets to the server, which returns structured results the harness renders.

## Learn more

- Product: [treestructuredplanning.com](https://treestructuredplanning.com)
- In-app setup guide: [Connect your agent](https://treestructuredplanning.com/docs/drive-with-your-agent/connect-your-agent)
- The skill contracts in [`tsp/skills/`](./tsp/skills/) and the harness-portable spec in [`tsp/docs/skill-behavior-spec.md`](./tsp/docs/skill-behavior-spec.md).

## License

[MIT](./LICENSE).
