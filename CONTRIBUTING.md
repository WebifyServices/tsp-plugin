# Contributing

This repository is a published mirror, not the place where the plugin is developed. The skills, manifests, and docs here are generated from the private TSP monorepo and synced out on each release. A pull request opened directly against this repo would be overwritten by the next sync, so the right path depends on what you want to do.

If you found a bug or want to request a change, open an issue here. That's the fastest way to get it in front of the team, and changes are then made upstream and flow back out on the next release.

If you're proposing wording or behavior changes to a skill, an issue with the specific `tsp/skills/<name>/SKILL.md` lines and what should change is ideal. The skill contracts are deliberately harness-portable (no Claude-specific constructs in the tool contracts themselves), so suggestions that keep that portability are easiest to adopt.

If you're porting these skills to another harness (Cursor, Codex, Cline), you don't need to change anything here. Read `tsp/docs/skill-behavior-spec.md`, which documents each skill as a contract any MCP-aware harness can re-implement against, and wire your harness's MCP client to the same endpoint with the same scoped token.

Security issues should not go in a public issue. See [SECURITY.md](./SECURITY.md).
