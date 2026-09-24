# src/xrGame/stalker_movement_manager_smart_cover_fov_range.cpp

> Answers "can I see, and can I reach, that position from this loophole" — the
> questions the combat planner asks before it commits a stalker to a piece of cover.

**Needs** — [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`cover_manager.h`](cover_manager.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: cone and range tests against remembered positions; no device or format contact.

## Purpose

A **loophole** is an authored aperture in a smart cover: a position, a direction, a
field-of-view cone and a distance range, all expressed in the cover object's local frame.
This file is the set of predicates that turn those authored numbers into answers about a
world position. They are separated from the rest of the movement manager only by topic.

The one decision that is not a simple geometric test is *which* position the creature is
asking about, and that is `fill_enemy_position`: a stalker reasons about the enemy's
**remembered** position, not its true one, unless a script has pinned a firing position.

## State

Stateless. Reads the two movement-parameter records and the creature's memory.

## `fill_enemy_position`

**Contract** — produces the position this creature should treat as the enemy's, or
reports that there is none. A script-supplied cover-fire position wins outright; failing
that, the currently selected enemy's *last remembered* position is used. Never reads the
enemy's live transform: a stalker may only act on what it knows.

```text
FUNCTION fill_enemy_position() -> optional<Position>
  IF current_params.cover_fire_position EXISTS
    RETURN current_params.cover_fire_position
  enemy <- memory.enemy.selected
  IF enemy IS none
    RETURN none
  RETURN memory.of(enemy).object_params.position
```

## `enemy_in_fov`

**Contract** — true when the creature is in a smart cover and its enemy can be engaged
from it. Allocates nothing, blocks on nothing.

**Invariants** — requires a current cover; with no cover the answer is a flat false rather
than an error, because the combat planner polls this every cycle regardless of state.

```text
FUNCTION enemy_in_fov() -> bool
  IF NOT current_params.cover EXISTS
    RETURN false
  position <- fill_enemy_position()
  IF position IS none
    RETURN false

  # First ask the cover whether ANY of its loopholes could engage that position.
  # A cover that can re-aim by changing loophole counts as covering the enemy even
  # when the loophole we happen to occupy does not — otherwise a stalker would
  # abandon a good cover because of the aperture it is currently standing at.
  IF cover.best_loophole(position, exclude_current = false, prefer_current = true) EXISTS
    RETURN true

  RETURN cover.is_position_in_fov(current_loophole, position)
     AND cover.is_position_in_range(current_loophole, position)
```

## `in_fov` and `in_range`

**Contract** — the same two questions asked about an arbitrary named cover and loophole
rather than the current one, so that a planner can evaluate a cover it has not entered.
Both resolve the cover by identifier through the cover registry and the loophole by
identifier within that cover's description; both fail loudly if either name is unknown,
because the names come from authored data and a typo must not degrade silently into
"no".

```text
FUNCTION in_fov(cover_id, loophole_id, position) -> bool
  cover    <- cover_registry.smart_cover(cover_id)
  loophole <- cover.description.loophole(loophole_id)   # fails if absent
  RETURN cover.is_position_in_fov(loophole, position)

FUNCTION in_range(cover_id, loophole_id, position) -> bool
  ... same shape, is_position_in_range
```

## `in_current_loophole_fov` and `in_current_loophole_range`

**Contract** — the same two questions about wherever the creature currently *is*, with
one wrinkle that is the reason these exist as separate entry points: while the enter
animation is playing the creature does not yet have a current cover, so the questions are
answered against the cover and loophole it is entering.

**Invariants** — exactly one of "has a current cover" and "is entering with an animation"
must hold when these are called; calling them outside a cover entirely is a programming
error.

```text
FUNCTION in_current_loophole_fov(position) -> bool
  IF current_params.cover EXISTS
    RETURN current_params.cover.is_position_in_fov(current_params.cover_loophole, position)
  # mid-entry: the target cover is not yet "current"
  cover    <- cover_registry.smart_cover(enter_cover_id)
  loophole <- cover.description.loophole(enter_loophole_id)
  RETURN cover.is_position_in_fov(loophole, position)
```

## `in_min_acceptable_range`

**Contract** — a private weakening of `in_range`: true when the position is at least
`min_range` away from the loophole, used where a loophole is unusable against a target
that has closed to point-blank. Resolves cover and loophole by name exactly as `in_fov`
does.

**Notes**

Every one of these delegates the actual cone or interval arithmetic to the cover object,
which owns the loophole description and the cover's world transform. That indirection is
not decoration: loophole geometry is authored in the cover's local frame and a smart
cover can be placed anywhere, so only the cover can transform the test correctly. A
rebuild that inlines the cone test must remember to transform the position into the
cover's frame first.
