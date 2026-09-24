# src/xrGame/ai/monsters/states/monster_state_rest_walk_graph.h

> Declares the wander-between-graph-points leaf, implemented in
> [`monster_state_rest_walk_graph_inline.h`](monster_state_rest_walk_graph_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_rest_walk_graph_inline.h`](monster_state_rest_walk_graph_inline.h.md)
**Used by** — [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md) · [`monster_state_rest_walk_graph_inline.h`](monster_state_rest_walk_graph_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf a creature runs during the wandering third of its peacetime clock: walk from one
game-graph point to the next, indefinitely.

Stateless, with no completion test — the peacetime cascade above it decides when the wandering
window has expired.

## `CStateMonsterRestWalkGraph`

- **execute** — walk toward successive graph points with the idle voice
