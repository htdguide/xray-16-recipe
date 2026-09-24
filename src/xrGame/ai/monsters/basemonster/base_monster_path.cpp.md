# src/xrGame/ai/monsters/basemonster/base_monster_path.cpp

> How a creature turns to face a point, and the four cover queries every creature's states are written in terms of.

**Needs** — [`base_monster.h`](base_monster.h.md) · [`corpse_cover.h`](../corpse_cover.h.md) · [`cover_manager.h`](../../../cover_manager.h.md) · [`cover_point.h`](../../../cover_point.h.md) · [`cover_evaluators.h`](../../../cover_evaluators.h.md) · [`control_direction_base.h`](../control_direction_base.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: four parameterised searches over the level's precomputed cover points

## Purpose

Two unrelated things, both small.

The first is the default implementation of "look at this point", which every creature may
override — a bloodsucker turns its head, a burer turns its whole body — and which by default
just retargets the body's heading.

The second, and the reason the file matters, is the **four cover queries**. Cover is a
per-navigation-vertex precomputed measure shipped with the level (see the glossary), and the
cover manager can find the best point near a position under a supplied *evaluator*. This
file names the four evaluations creatures actually use, fixes their search radii, and hands
each one a shared evaluator object. Every creature state that hides, flanks or retreats is
written against one of these four.

## `look_at`

**Contract** — points the creature's body heading at a world position. The angular speed
argument is accepted and ignored by the default implementation; the turn rate comes from the
direction control channel.

**Notes** — the heading is negated relative to the direction vector's own heading. That is
the model format's sign convention for facing, and it appears at every site that converts a
world direction into a creature's heading.

## the cover queries

All four share a shape: configure a pre-built evaluator, ask the cover manager for the best
point within a search radius of the creature, and report the point's position and navigation
vertex. All four return false when no point qualifies, and **none of them falls back** —
a creature with no cover available must decide what to do about that itself.

```text
FUNCTION find_cover(evaluator, search_radius) -> optional<(position, vertex)>
  point = cover_manager.best_cover(my position, search_radius, evaluator)
  IF none THEN RETURN none
  RETURN (point.position, point.vertex)
```

### `cover_for_dragging_a_corpse`

**Contract** — a place to drag a corpse to and eat it undisturbed, between 10 and 50 units
away, searched within 30 units of the creature. Uses the corpse-specific evaluator, which
scores a point by how concealed it is rather than by its relation to any enemy.

**Notes** — the distance band is wider than the search radius, which looks contradictory and
is not: the band constrains *which* points the evaluator accepts, measured from the corpse's
situation, while the radius bounds the search around the creature.

### `cover_from_enemy`

**Contract** — a place hidden from a known enemy position, between 30 and 50 units from that
enemy, searched within 40 units of the creature. Uses the far-from-enemy evaluator.

**Invariants** — the minimum of 30 units is what makes this a *retreat*, not a flank: a
creature asking for cover from its enemy is asking to be a long way away from it.

### `cover_from_point`

**Contract** — the same evaluator with every bound supplied by the caller: the point to hide
from, the distance band, and the search radius. This is the general form; the previous query
is it with three constants.

### `cover_close_to_point`

**Contract** — a place *near* a destination rather than away from a threat, with a distance
band, a permitted angular deviation from the destination's bearing, and a search radius. Uses
the close-to-enemy evaluator.

**Notes** — the deviation parameter is what distinguishes this from the others: it constrains
the *direction* of the cover point relative to the creature's line to the destination, which
is how a creature approaches a position obliquely instead of walking straight at it.

**Invariants shared by all four** — the three evaluator objects are constructed once at load
and reconfigured per query, so none of these routines allocates. They are also not reentrant
for that reason: a second query while one is outstanding would overwrite its configuration.
Nothing in the chapter issues nested cover queries.
