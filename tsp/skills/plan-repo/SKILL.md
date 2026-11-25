---
name: plan-repo
description: Author a faithful TSP plan of an existing repository from its code - nodes from its units and modules, edges from its real dependencies. Triggers on "plan this repo", "map this codebase in TSP", "import my repository into TSP", or "/tsp:plan-repo".
argument-hint: "[path inside the repository]"
---

# tsp:plan-repo

First follow [Worker lifecycle](../../docs/application-sessions.md).

Plan the repository as it is built. Every claim the plan makes must be true of the code, and every
part of the code a person would work in must have exactly one node that owns it. Read the code
locally; send TSP only the plan. The node contract and edge gates are the ones in
[plan-author](../plan-author/SKILL.md) ("Node contract" and "Edges are exceptional"); read those two
sections before step 6.

**Nothing comes from a name.** Every title, intent, scope phrase, criterion and edge note rests on
code you opened: a body, a caller, a test, a README. A file, folder or route name only tells you
what to read. Text that could be written from the file tree alone is wrong. A part you could not
read enough to know gets only what it observably does, and goes on the "inferred" list.

## Required MCP tools

`tsp.context.get/set`, `tsp.plan.create`, `tsp.nodes.create`, `tsp.node.update`,
`tsp.edge.create/delete`, `tsp.edges.list`, `tsp.tree.summary`. Never `tsp.node.generate`.

## What becomes what

- **Root**: the system the repository holds. In scope: its top-level source folders. Out of scope:
  vendored and generated code, fixtures, what you could not read. Each path there is left out whole;
  write a file pattern as a glob (`src/**/*.generated.ts`).
- **First level**: the units the repository declares in its manifests: workspace packages, apps and
  services it deploys, libraries it publishes, a CLI. One node per unit. A repository with one
  unit has no unit level: the root is that unit, and its parts are the first level.
- **Below**: one node per cohesive responsibility, a module or folder group one engineer would
  describe in one sentence and change as a unit. Classes and functions are never nodes; name the
  public ones in the in-scope of the node that owns them.
- **Atomic**: a node whose only possible children are HOWs (single functions, one file per endpoint
  of the same kind). Mark it `atomic: true` and stop.
- **Not nodes**: tests, CI, build scripts, docs, third-party libraries and services. Tests ground
  acceptance criteria. The adapter that wraps a service is the node; name the service in its scope.

Edges, from what the code does:

- A imports or calls B's public surface, in another branch: `uses` A to B; the note names the
  module, function, endpoint, event or type.
- A implements an interface, protocol, abstract base or port B defines: `implements` A to B; the
  note names the interface.
- A is built from B's output (generated client, codegen, compiled schema): `depends_on` A to B; the
  note names the artifact.
- Code or docs mark A deprecated in favor of B, both present: `supersedes` B to A.
- No edge for imports inside one node or between parent and child, standard library, third party,
  tests, or a module's own re-exports.

Runtime use is `uses`, never `depends_on`. Never author `relates_to` or `conflicts_with`. If the
server refuses a `depends_on` as a cycle, it was a `uses`.

## Steps

```text
1. Scope. The target is the repository root, or the path the person gave.
   tsp.context.get. Planning a path under an existing plan: find the node
   that owns it and plan beneath it (skip plan.create in step 7).
2. Survey, read-only. List the tracked source files and their folders.
   Read every manifest (package.json, workspace files, pyproject.toml,
   go.mod, Cargo.toml, deploy config) and the READMEs.
   Write down: the declared units, the folders to exclude, and the
   source-file count per folder.
3. Dependencies. Grep the import statements of each unit and top
   folder. Tabulate cross-folder imports: importer folder, imported
   folder, count (a statement counts once per module it imports,
   never per name), and the main symbols that cross.
4. Tree. Level 1 = the declared units, or with only one unit, its
   parts. A unit's node owns its whole folder (`packages/client/`),
   never a list of its files; the root's exclusions remove generated
   code. For each node you decompose,
   name the candidate axes (feature, layer, data domain, the folder
   layout), count from step 3 the imports that would cross between
   siblings under each, and pick the fewest; on a tie, the folder
   layout. One axis per level. Stop at the grain above. Budget about
   150 nodes: past it, stop at the units, leave them non-atomic with
   full contracts, and note which to deepen in a later run.
5. Show the person levels 1 and 2 with their paths and the axis of
   each split. Write nothing until they agree, unless they said to go
   ahead without review.
6. Contracts, from the code you read, for every node:
   - intent: what this part does, 1-2 sentences. A module nothing
     imports says so.
   - in_scope: noun phrases ending in their paths in backticks, e.g.
     "Request parsing (`src/parser/`)". A folder ends in "/" and claims
     all under it: never "except" or "other than"; list the remaining
     folders and files instead. Every path must exist.
   - out_of_scope: never empty; each entry is the exact title of the
     node that owns what a reader would wrongly assume is here.
   - acceptance_criteria: behavior the code provides today, each
     traceable to a test or an entry point you read.
   - rationale on parents: the axis chosen and the runner-up, in words.
   - kind, workstream; atomic on leaves. No issue or ticket numbers.
7. Write the tree. tsp.plan.create with the root, then tsp.context.set
   to that plan. plan.create takes no acceptance criteria: set the
   root's with tsp.node.update now. Then tsp.nodes.create one subtree per batch (at most
   100 nodes), each sent with validate_only=true first, then again
   without it. No edges in these batches.
8. Edges. From the step 3 table, at the grain of the finished tree,
   apply the table above and plan-author's gates: cross-branch only,
   never to an ancestor, one per node pair, no transitive echoes,
   attached at the most specific level where it holds. Each note names
   the importing file and the module it imports, or the endpoint it
   calls (`POST /api/events`). Rank the
   node pairs by imports crossing either way, summed: the heaviest
   tenth each get an edge, even into a shared helper node and even when
   another path implies it.
   Mutual imports between two nodes: re-cut the tree first; if the cut
   is right, one edge in the dominant direction. Create each survivor
   with tsp.edge.create.
9. Boundary pass. Read every node's out_of_scope. An entry that is not
   an exact title in this plan (root excepted) is a bug: fix it with
   tsp.node.update.
10. Coverage pass. Every source folder from step 2 that is not excluded
   falls under some node's paths, every file in a unit's folder
   (config included) falls under that unit's node, and no file falls under two nodes
   unless one is the other's ancestor. Check every path you wrote
   exists. Fix with tsp.node.update.
11. Self-review. tsp.tree.summary: edges well below nodes, no parent
   with a lone child. tsp.edges.list: every edge passes step 8.
   Delete what fails with tsp.edge.delete.
12. Report to the person: the plan's title and id, node and edge
   counts, the "inferred" list, nodes whose code is a stub or
   unfinished, subtrees left to deepen, and that no node is marked
   complete because no attempt ran.
```

## Blocking conditions

- No readable source, or the path holds no code: say so; create nothing.
- Two axes tie and the layout is mixed: ask the person which to use before writing.
- Plan, node cap or tier error: show the server's error; narrow the path, never the contract quality.
- A batch fails validation: fix every listed error and validate again; never drop nodes to pass.

## State writes

Structure only: plan, nodes, edges. Never a workflow or attempt write.
