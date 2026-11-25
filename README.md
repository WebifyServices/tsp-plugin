# TSP plugin

Tree-Structured Planning (TSP) is a canvas for planning software systems that coding agents can read from and write back to. This plugin installs in **Claude Code**, **Cursor**, and **Codex**. One package gives you the TSP skills, a `connect` command and a connection to the hosted TSP MCP server.

Install once, then your agent can author plans, implement off a node, record progress, reverse-map a diff to the nodes that govern it, and hand a session off when you stop. All three harnesses get the same skills against the same MCP server. Only the install step and how you invoke a skill change. The skill contracts are written to port to any MCP-aware harness (Cline and others re-implement against the same spec), so the three supported here are peers, not a primary plus ports.

> This repository is a published mirror, updated on each release, so the skills and manifests here are generated, not hand-edited. See [CONTRIBUTING](./CONTRIBUTING.md) before opening a PR.

The optional [TSP gateway](https://www.npmjs.com/package/@treestructuredplanning/gateway) lets the canvas launch, stop, and resume Claude Code and Codex sessions on a paired computer. In Claude Code, `/tsp:connect` sets it up in one command (see [Connect this computer for Build](#connect-this-computer-for-build)). Keep the plugin installed for the launched agent’s TSP tools. The gateway never changes your harness permissions and never answers approval prompts.

## What you need first

- One of [Claude Code](https://code.claude.com), [Cursor](https://cursor.com) (2.5 and later), or [Codex](https://developers.openai.com/codex), whichever you already use.
- A TSP account at [treestructuredplanning.com](https://treestructuredplanning.com). You sign in to TSP once, in your browser, and your harness's section below gives that step. No API keys or tokens to paste.

## Install

Pick your harness and read only its section: each one installs, signs in and checks the connection on its own. Installing the plugin doesn't sign you in. Until you sign in, your agent starts without TSP's tools and may not say so. These steps match the in-app [Connect your agent](https://treestructuredplanning.com/docs/drive-with-your-agent/connect-your-agent) guide.

### Claude Code

**Install the plugin:**

```text
/plugin marketplace add WebifyServices/tsp-plugin
/plugin install tsp@tsp-plugins
```

The first command registers this repo as a Claude Code plugin marketplace. The second installs the `tsp` plugin from it. Reload if prompted, so the skills and the MCP server register.

**Sign in once.** Run `/mcp`, select the TSP server, and approve in the browser it opens (OAuth 2.1 + PKCE). From a shell, this does the same:

```bash
claude mcp login plugin:tsp:tsp
```

The token is stored and refreshed automatically.

**Check it.**

1. `claude mcp list` shows **Connected** next to the TSP server. **Needs authentication** means you haven't signed in yet.
2. Type `/tsp:` to autocomplete the TSP skills and the `connect` command.
3. Ask Claude to list your TSP plans. A plan list, even an empty one, means the read path is live.

**If something goes wrong.**

- **It needs authentication again.** Run `/mcp`, select the TSP server, and choose **Re-authenticate**.
- **You use Claude Desktop or claude.ai.** The `/plugin` and `/mcp` commands are Claude Code's (terminal or IDE). In Claude Desktop or claude.ai, add `https://treestructuredplanning.com/mcp/` as a custom connector instead.

### Cursor

Cursor 2.5 and later.

**Install the plugin.** Run `/add-plugin` and point it at `WebifyServices/tsp-plugin`. This brings in the TSP skills.

**Add the server.** Add the `tsp` server to `.cursor/mcp.json` in your project, or `~/.cursor/mcp.json` for every project:

```json
{
	"mcpServers": {
		"tsp": {
			"url": "https://treestructuredplanning.com/mcp/"
		}
	}
}
```

A bare `url` is enough. Don't add an `Authorization` header or a client id. A static auth block suppresses the automatic sign-in.

**Sign in once.** Cursor shows the server as needing sign-in, opens your browser to authorize it, then stores and refreshes the token on its own. From a terminal, Cursor's CLI does the same:

```bash
agent mcp login tsp
```

**Check it.**

1. The TSP skills appear in Cursor's `/` skill menu. Cursor also applies them automatically when they're relevant.
2. Ask Cursor to list your TSP plans. A plan list, even an empty one, means the read path is live.

**If something goes wrong.**

- **Cursor sits on "needs login" without opening the browser.** Use the bridge form, which runs the sign-in itself:

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

### Codex

**Install the plugin.** Open `/plugins`, add the `WebifyServices/tsp-plugin` marketplace, and install `tsp`. This brings in the TSP skills and the TSP server, so don't also add a `[mcp_servers.tsp]` block to `config.toml`. Codex keeps only one of the two, and a block whose address differs even by its final slash needs a sign-in of its own.

**Sign in once:**

```bash
codex mcp login tsp
```

That opens your browser to authorize and caches the result. On a Mac, Codex keeps this sign-in in your Keychain, so every Codex setup on the computer that names `tsp` at the same address shares it.

**Check it.**

1. `codex mcp list` shows **Not logged in** next to `tsp` until you sign in.
2. `/skills` lists the TSP skills.
3. Ask Codex to list your TSP plans. A plan list, even an empty one, means the read path is live.

**If something goes wrong.**

- **`codex exec` runs without TSP's tools.** It doesn't say when you haven't signed in. Run the sign-in above.
- **An older Codex build errors on `codex mcp login`.** Enable Codex's streamable-HTTP client by adding a `[features]` block with `rmcp_client = true`, then retry.
- **You want TSP's tools without the plugin.** Add the server with `codex mcp add tsp --url https://treestructuredplanning.com/mcp/`, then sign in the same way. If your Codex build can't reach the URL directly, use the bridge form instead (keep only one `tsp` block):

```toml
[mcp_servers.tsp]
command = "npx"
args = ["-y", "mcp-remote", "https://treestructuredplanning.com/mcp/"]
```

## What you get

These skills are each triggered by natural language or their slash form. However you connect, you get the same set wired to your plans. Only the way you reach them differs by harness (Claude Code's `/tsp:` commands, Cursor's `/` skill menu, Codex's `/skills`):

| Skill             | Trigger                                         | What it does                                                                                                                      |
| ----------------- | ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `plan-author`     | `/tsp:plan-author`, "create a plan"             | Author or repair plans at generative-grade structure: tree-first decomposition, minimal justified edges, complete node contracts. |
| `plan-repo`       | `/tsp:plan-repo [path]`, "plan this repository" | Write a plan of a repository you already have: a node for each part of the code, relationships where parts use each other.        |
| `implement`       | `/tsp:implement <node-id>`                      | Turn a node or subtree into shipped code; route atomic vs. non-atomic work; record results.                                       |
| `next-actionable` | `/tsp:next-actionable`, "what's next"           | Recommend the next node to pick up, ranked by critical path and dependency readiness.                                             |
| `explain`         | `/tsp:explain <topic>`                          | Explain a topic against the plan: which nodes govern it, how they relate, where the gaps are.                                     |
| `align`           | `/tsp:align <node-id>`                          | Detect drift between your code and a node's intent, scope, and acceptance criteria.                                               |
| `impact`          | `/tsp:impact <path>`                            | Reverse-map a file or diff to the nodes that govern it, before a refactor or review.                                              |
| `debug`           | `/tsp:debug <stack-trace>`                      | From an error or failing test, identify which nodes likely govern the failing path.                                               |
| `handoff`         | `/tsp:handoff`, "wrap up"                       | Close a session: write status, touched files, blockers, and next steps back to the canvas.                                        |

The MCP server has no filesystem access by design. Every skill that needs your code reads it locally inside your agent and ships only bounded snippets to the server, which returns structured results the harness renders.

## Connect this computer for Build

`/tsp:connect` is a command, not a skill your agent picks on its own: it runs only when you type it (its skill file sets `disable-model-invocation`). Run it in Claude Code inside the repository you want TSP to build in. Its whole source is [`tsp/skills/connect/SKILL.md`](./tsp/skills/connect/SKILL.md). It asks Claude to run one command, which you approve through your normal permission prompt:

```sh
npx -y @treestructuredplanning/gateway@0.3.4 connect [folder]
```

That is the published [TSP gateway](https://www.npmjs.com/package/@treestructuredplanning/gateway) package at a pinned version, and `connect` does the following:

- **Pairs this computer** with your TSP account, only after you approve in the browser window it opens. An existing working pairing is kept.
- **Registers the repository folder** (the current one unless you name another). The folder must be a clone with a GitHub remote.
- **Starts the gateway at login** and prints its status, plus the steps left to you in each folder, such as signing in or running `claude` or `codex` once there to accept its trust prompt.

What it writes on your computer:

| What           | Where                                                                                                                              |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Login item     | `~/Library/LaunchAgents/com.treestructuredplanning.tsp-gateway.plist` (macOS), a user systemd unit (Linux), a logon task (Windows) |
| Config         | `~/.config/tsp-gateway/config.json`: pairing identity, registered folders, captured `PATH` (`%APPDATA%\tsp-gateway` on Windows)    |
| Credential     | One OS credential-store entry, `com.treestructuredplanning.gateway`, holding the pairing's refresh token                           |
| State and logs | `~/.local/state/tsp-gateway` (`%LOCALAPPDATA%\tsp-gateway` on Windows)                                                             |

What it never does: it never sets or changes your Claude Code or Codex permission settings, allowed tools or configuration, and never passes a permission-bypassing flag. Sessions TSP launches inherit your own defaults. Only you can start it, approve pairing, sign in to a harness or accept a folder's trust prompt. The gateway only connects outward, to TSP, and accepts no inbound connections. What TSP can ask it to do, and what it sends, is on the [gateway's page](https://www.npmjs.com/package/@treestructuredplanning/gateway).

Running it again is safe and quick: a working pairing, a registered folder and a current login item are left as they are. Run outside a repository, it keeps your registered folders and repairs the rest, which also fixes a stopped or outdated gateway.

To undo it, run:

```sh
npx -y @treestructuredplanning/gateway@0.3.4 disconnect
```

`disconnect` removes the login item and revokes the pairing in TSP (the machine disappears from Settings), then deletes the credential-store entry. Your registered folders stay in the config file; delete `~/.config/tsp-gateway` and `~/.local/state/tsp-gateway` too if you want no trace left.

## Learn more

- Product: [treestructuredplanning.com](https://treestructuredplanning.com)
- In-app setup guide: [Connect your agent](https://treestructuredplanning.com/docs/drive-with-your-agent/connect-your-agent)
- The skill contracts in [`tsp/skills/`](./tsp/skills/) and the harness-portable spec in [`tsp/docs/skill-behavior-spec.md`](./tsp/docs/skill-behavior-spec.md).

## License

[MIT](./LICENSE).
