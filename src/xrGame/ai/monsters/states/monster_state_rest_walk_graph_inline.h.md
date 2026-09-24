# src/xrGame/ai/monsters/states/monster_state_rest_walk_graph_inline.h

> Wander: let the path system lead you from one game-graph point to the next, walking, indefinitely.

**Needs** — [`monster_state_rest_walk_graph.h`](monster_state_rest_walk_graph.h.md)
**Used by** — [`monster_state_rest_walk_graph.h`](monster_state_rest_walk_graph.h.md)
**Tier floor** — T3: three calls

## Purpose

The other half of peacetime, and the reason the world is not static. Where the idling composite
settles a creature into cover, this one moves it across the level — and because it moves on the
**game graph**, the coarse cross-level graph the alife simulation itself uses, a wandering creature
drifts toward the same places the offline simulation would have taken it. Online and offline
wandering therefore agree, which is what stops a creature from teleporting when it is promoted or
demoted.

## State

`Stateless.` The "which point next" bookkeeping belongs to the path component, not to this leaf,
which is why the leaf is three lines and has nothing to reset.

## `execute`

**Contract** — ask the path component to keep touring graph points, request the walking action, and
play the idle voice. Runs unchanged every update.

**Notes** — the tour directive is *continuous*, not a one-shot destination: each update restates it
and the path component keeps a creature moving to the next point when it reaches one. A rebuild
that models this as "pick a graph point, walk there, finish" needs an outer loop; the original has
none because the leaf has no completion test at all. The peacetime cascade ends it by clock, after
thirty seconds.

Nothing constrains the tour to the creature's home region or to its restrictors at this level. The
peacetime cascade re-runs every update and its higher-priority branches — restrictor compliance and
the pull back toward the territory — catch a creature that has wandered out. So the containment is
in the cascade, not in this leaf, and a rebuild that moves the containment down here will find
creatures that never leave their starting cell.

No number is authored and none is compiled in: the gait is fixed, the voice throttle comes from the
creature's section through the generic voice path, and everything else is the path component's.
