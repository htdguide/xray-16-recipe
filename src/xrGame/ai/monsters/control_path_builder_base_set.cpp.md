# src/xrGame/ai/monsters/control_path_builder_base_set.cpp

> The target-setting surface: three ways for the state layer to say where a creature should go, and the reset that clears them.

**Needs** — [`control_path_builder_base.h`](control_path_builder_base.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../xrAICore/Navigation/level_graph.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: field writes and one comparison

## Purpose

Everything a creature's state layer is allowed to say about *where*. The operations here
do no validation at all — they record an intention and update one flag. All checking
happens later, on the per-frame pass, and that separation is the point: a state may set
its target every tick without paying for a path search every tick.

## State

Writes `target_set`, `target_actual` and `target_type` of the record in
[`control_path_builder_base.h`](control_path_builder_base.h.md).

## `set_target_point`

**Contract** — record a destination as a position and optionally a mesh vertex. Clears the
target's *actuality* if the new destination differs from the one already recorded, keeps
it otherwise. Switches the target type to "move to" and the path type to the level path.
Never validates and never searches.

```text
FUNCTION set_target_point(position, node)
  target_actual <- target_actual AND position ~= target_set.position
                                 AND node     == target_set.node
  target_set    <- (position, node)
  target_type   <- move_to
  path_type     <- level
```

**Notes** — actuality is *conjunctive*, not assigned: a target already stale stays stale
even if this call repeats the same destination. That is what stops a state that re-sets
the same point every frame from ever getting a replan when the path underneath it failed.

The second spelling takes a vertex alone and derives the position from the mesh.

## `set_retreat_from_point`

**Contract** — record a position to move *away* from. Same actuality rule, comparing
position only; the node is cleared to unknown, because a retreat has no destination vertex
until one is chosen. Switches the target type to "retreat from" and the path type to the
level path.

**Notes** — the two target types diverge only during resolution, where a retreat is turned
into a concrete destination by projecting a fixed distance directly away from the recorded
point. See [`control_path_builder_base_path.cpp`](control_path_builder_base_path.cpp.md).

## `prepare_builder`

**Contract** — reset the path-following parameters and both targets to a known state:
replanning unthrottled, arrival tolerance one world unit, no failure recorded, cover
approach off, the set target cleared, the found target set to the creature's own current
position, and every timestamp zeroed.

**Notes** — seeding the *found* target with the creature's own position, rather than
leaving it empty, means a freshly reset creature has a valid target that happens to be
where it already is. It will therefore report "arrived" rather than "no target", which is
the quiet state a creature should come up in.

The arrival tolerance of one unit here differs from the three units the generic parameters
install; whichever ran last wins, and in practice a state sets its own.
