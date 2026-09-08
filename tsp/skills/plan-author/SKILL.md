---
name: plan-author
description: Author high-quality TSP plans over MCP (tsp.plan.create, tsp.nodes.create, tsp.edge.create, tsp.node.update). Encodes the generative system's structure discipline - the tree carries the architecture, the plan stays as lean as the requirements permit, edges are rare and justified, every node ships a complete contract. Triggers on "create a plan", "author a plan in TSP", "add nodes", "model this in TSP", any bulk node/edge authoring, or "/tsp:plan-author".
---

# tsp:plan-author

First follow [Worker lifecycle](../../docs/application-sessions.md).

Turn requirements into a
plan or subtree with a
decomposition tree that carries the structure, a minimal set of edges
that each earn their place, a complete contract on every node — or
repair an existing plan to that standard. Hand-authored plans fail two
ways: a flat tree wrapped in a dependency blanket, and a plan inflated
past the requirements; this skill prevents both.

## Required MCP tools

`tsp.context.get/set`, `tsp.plan.create`, `tsp.nodes.create`,
`tsp.node.update`, `tsp.edge.create/delete`, `tsp.edges.list`,
`tsp.tree.summary`. Optional: `tsp.node.generate` (metered).

## Right-sized leanness

**The plan is exactly as lean as the requirements permit.** Plans are
executed by coding agents downstream; every node an agent reads is
context spent, so an inflated plan is worse even when nothing in it
is wrong. Match plan complexity to requirements complexity — intricate
system, intricate plan; modest tool, modest plan. Hold both bounds at
once:

- **Coverage floor**: every surface, capability, or constraint the
  source states gets an owning node. Leanness never deletes
  stated scope.
- **Leanness ceiling**: nothing the source does not state —
  no speculative infrastructure, no padding nodes added to look
  complete. When genuinely unsure, leave it out and
  record it in the root's out_of_scope.

## The structure discipline

**The tree carries the architecture.** Decompose parent into children;
the tree already orders every child before its parent. Never draw an
edge to say "the parent needs its parts" or "B comes after A" —
sequencing is scheduling, not dependency.

Decomposition:

1. **One honest step down in specificity per decomposition.** A child
   is a semantically coherent part of the parent, one level more
   concrete. Both failure smells are bad jumps: _too small_ — a lone
   child rephrasing the parent (the parent is atomic, or the cut is
   wrong); _too large_ — a flood of children at catalog granularity
   (80 chart types straight under a catalog node). The domain's
   semantics name the missing middle layer — categories, stages,
   domains; introduce it and hang the specifics beneath.
2. **Child count is an output of the semantics, never a target.** A
   two-part concern gets two children; a many-sided one gets many only
   when no honest grouping exists. Depth that mirrors real structure
   beats a shallow, wide flattening of it.
3. **One partition axis per level.** Sketch candidate axes for THIS
   node — by capability, layer, user journey, lifecycle stage, data
   domain are examples; the right axis is whatever the node's own
   semantics suggest. Pick the one whose children need the fewest
   inter-child contracts; never mix axes among siblings.
4. **Siblings are comparable in kind, not size.** Subtree weight
   mirrors semantic weight — an honestly heavier concern earns the
   bigger subtree. Fold a child into a sibling only when it owns no
   distinct concern; split one out only when the source makes it a
   real, separate concern.
5. **Atomicity is semantic, never depth.** When the only children you
   can name are HOWs — CRUD verbs, per-channel/per-role copies,
   pipeline micro-steps — the node is atomic: mark it and stop.

Node contract — every node, parents included:

- `intent`: 1-2 sentences, WHAT not HOW, strictly narrower than the
  parent's; if it reads as true for the parent too, it is too broad.
- `in_scope`: concrete noun phrases naming what lives inside the
  boundary — never implementation steps or the title restated, no
  padding.
- `out_of_scope`: never empty — name what a reader would wrongly
  assume is inside. Every entry IS the owning node's exact title,
  verbatim: no rephrasing, no commentary like "(owned by X)".
  Project non-goals live on the root ONLY — never pasted into
  children. Ambiguous item: one owner; list it in the other's.
  Enforced by step 4.
- `acceptance_criteria`: observable, independently verifiable
  outcomes — enough to define done, nothing decorative.
- Ground everything in the source; an invented entry poisons every
  decomposition below it.
- Set `kind` and `workstream`; mark leaves `atomic: true`.

**Edges are exceptional.** The goal is the MINIMAL set that tells an
engineer something the tree does not already say; zero is a perfectly
good outcome. Gate every candidate:

- **depends_on test:** would you genuinely refuse to start the
  source until the target ships? Parallel against an agreed
  interface means `uses`, not `depends_on`.
- **uses test:** name the concrete thing crossing the boundary
  (endpoint, function, event, dataset) in the edge `note`; if you
  cannot name it, the edge is decorative — drop it.
- **Never:** edges to a parent, child, or any ancestor (the tree
  encodes those); transitive depends_on echoes (A→B→C implies
  A→C); a second edge on a node pair in any direction or type;
  foundation/setup blankets; sequencing-as-dependency.
- **Attach at the most specific level.** If ALL children of A relate
  to B the same way, one edge from A beats N from its children.
- **Far fewer edges than nodes.** A node needing many edges is usually
  cut wrong — fix the tree, don't spend more edges. `relates_to` is
  reserved for human annotation; don't author it.
- **Whiteboard test:** would an engineer explaining this system draw
  that arrow? Every kept edge carries a `note` naming what flows.

## Tool sequence

```text
1. tsp.context.get; then tsp.plan.create (always with the inline
   root), or locate the parent node for a subtree.
2. TREE first: tsp.nodes.create in batches per subtree (children
   reference batch node_ids), full contracts, NO edges.
3. Edge pass, once, over the finished tree: list candidates, apply
   the gates, create only survivors — each with a note.
4. Boundary pass: read every node's out_of_scope. Empty, not an
   exact owner title, "(owned by X)" commentary, or a root
   disclaimer in a child — each is a bug; fix via tsp.node.update.
5. Coverage pass: re-read the source; every stated surface needs an
   owning node — auth/accounts, admin UI, data schema are classic
   drops. Add what is missing.
6. Self-review (below); delete edges that fail a gate.
7. Large or unfamiliar decomposition? Prefer tsp.node.generate;
   review its proposal by these same rules.
```

## Self-review gates

- Leanness ceiling: walk the plan — every node traces to something
  the source states.
- `tsp.tree.summary`: edge_count well below total_nodes; depth
  reflects real structure (the canonical failure: a flat root
  fanning into everything); each decomposition is one honest
  specificity step — no catalog floods, no lone rephrasers.
- `tsp.edges.list`: every edge cross-branch, noted, non-transitive,
  one per pair, none touching an ancestor of its other end.
- Every node's contract fields: non-empty and honest.

## Blocking conditions

| Condition                    | Behavior                                                              |
| ---------------------------- | --------------------------------------------------------------------- |
| Source too thin to decompose | Ask for the missing context; never pad with invented children.        |
| Plan/node cap or tier errors | Surface the server error; narrow scope explicitly, not quality.       |
| Existing plan violates this  | Offer a repair pass (edge deletions first); don't mirror the pattern. |

## State writes

Structure only: plan, nodes, edges. No workflow writes — status stays
`draft` until implementation sessions claim nodes.
