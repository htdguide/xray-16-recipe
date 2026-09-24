# src/xrGame/ai/monsters/states/state_move_to_restrictor_inline.h

> Implements the return-to-permitted-space correction: sprint to the nearest accessible navigation cell and stop the moment you are legal again.

**Needs** — [`state_move_to_restrictor.h`](state_move_to_restrictor.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`state_move_to_restrictor.h`](state_move_to_restrictor.h.md)
**Tier floor** — T3: one nearest-accessible query and a fixed movement assertion

## Purpose

A creature's permitted space is the algebra of its restrictors, and that algebra can change
under it — a script adds a forbidden volume, a smart terrain revokes a permission, a
physics push shoves it through a wall. When it does, every route search the creature issues
will fail, because no accessible route exists from where it now stands. This state is the
escape hatch: it is checked before anything else, and it overrides whatever the creature
was doing until the violation is repaired.

The decision worth keeping is that repair is not a special movement mode. The state picks
one destination — the nearest cell the creature *is* allowed to occupy — and then runs the
creature's ordinary aggressive run toward it.

## `CStateMonsterMoveToRestrictor`

**Contract** — the start condition is true exactly when the creature's current position is
not accessible under its restrictions. Entry clears the path builder, asks the restriction
set for the nearest accessible navigation vertex, and sets that vertex's position as the
destination — chosen once, not re-evaluated. Execution asserts a run with the aggressive
acceleration profile and braking on, and arms the creature's idle vocalisation. Completion
is true exactly when the position is accessible again.

```text
FUNCTION check_start_conditions() -> bool
  RETURN NOT object.path_builder.accessible(object.position)

FUNCTION initialize()
  object.path.prepare_builder()
  (vertex, position) = object.path_builder.restrictions.accessible_nearest(object.position)
  object.path.set_target_point(level_graph.vertex_position(vertex), vertex)

FUNCTION execute()
  object.set_action(run)
  object.animation.acceleration_activate(aggressive)
  object.animation.acceleration_set_braking(true)
  object.set_state_sound(idle)

FUNCTION check_completion() -> bool
  RETURN object.path_builder.accessible(object.position)
```

**Invariants** — start condition and completion test are exact negations of each other, so
the state cannot be entered and immediately satisfied, and cannot loop. Its destination is
frozen at entry, so a restrictor that moves again mid-correction will leave the creature
running toward a stale target until the completion test happens to fire or the behaviour
above reselects the state — which it will, since the violation persists.

**Notes** — the destination is reported by the restriction algebra, not by a path search,
so there is no guarantee a route to it exists. The creature runs at it and relies on the
ordinary path builder to do something sensible. In practice the nearest accessible cell is
adjacent, because the violation is usually a step over a boundary.
