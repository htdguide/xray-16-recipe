# src/xrGame/ai/monsters/chimera/chimera_state_threaten_walk_inline.h

> Walk toward the target while it is nearer than eight units, and stop at five.

**Needs** — [`chimera_state_threaten_walk.h`](chimera_state_threaten_walk.h.md) · [`state_move_to_point.h`](../states/state_move_to_point.h.md)
**Used by** — [`chimera_state_threaten_walk.h`](chimera_state_threaten_walk.h.md)
**Tier floor** — T3: parameter authoring

## Purpose

The near-range approach of the intimidation display. Same construction as [`chimera_state_threaten_steal_inline.h`](chimera_state_threaten_steal_inline.h.md) with three differences, and the differences are the whole file: the gait is a walk rather than a creep, the path is rebuilt on a fixed cadence rather than the creature's configured attack cadence, and the stop distance is five units rather than eight.

## State

Stateless.

```text
stop_distance  = 5 world units
start_distance = 8 world units    # this state is legal only inside this range
rebuild_period = 1500 ms
```

## `initialize`

**Contract** — Authors the movement parameters and the initial destination at entry: walk, calm acceleration, no braking, finish within two units of the destination, idle vocalisations spaced by the creature's configured idle-sound delay, rebuild the path every 1500 ms.

## `execute`

**Contract** — Refreshes the destination to the target's current position and vertex, then runs the base state's tick.

## `check_start_conditions`

**Contract** — Legal only while the target is nearer than eight units. This is the exact complement of the creep's condition, which is how the parent tree picks between them by distance alone.

## `check_completion`

**Contract** — Done when the base state says so, or once the target is nearer than five units.

**Notes** — The stop distance is below the display's own abort distance of three units, so a walking chimera stops short and the display continues with another roar rather than tipping into a fight. The three thresholds — five to stop walking, three to abandon the bluff, eight to switch gait — are the whole shape of the behaviour and all three are literals.
