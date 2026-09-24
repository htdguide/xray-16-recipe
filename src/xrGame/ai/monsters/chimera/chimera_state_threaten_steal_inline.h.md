# src/xrGame/ai/monsters/chimera/chimera_state_threaten_steal_inline.h

> Creep toward the target until within eight units, then stop and let the display continue.

**Needs** — [`chimera_state_threaten_steal.h`](chimera_state_threaten_steal.h.md) · [`state_move_to_point.h`](../states/state_move_to_point.h.md)
**Used by** — [`chimera_state_threaten_steal.h`](chimera_state_threaten_steal.h.md)
**Tier floor** — T3: parameter authoring

## Purpose

One of two approach flavours in the intimidation display (the other is [`chimera_state_threaten_walk_inline.h`](chimera_state_threaten_walk_inline.h.md)). Both are the same shared movement state wearing different parameters; the difference between them is gait, pacing and when they stop.

This file is a good short illustration of the chapter's general technique: a "state" is very often nothing but an authored parameter record handed to a shared implementation, and the interesting content is the numbers.

## State

Stateless; the parameter record belongs to the base state.

```text
min_distance_to_enemy = 8 world units    # the boundary this state is defined by
```

## `initialize`

**Contract** — Authors the movement parameters once at entry.

```text
FUNCTION initialize()
  base.initialize()
  data.action          = creep
  data.accelerated     = true
  data.braking         = false
  data.accel_type      = calm            # ramps up gently, unlike an attack approach
  data.completion_dist = 2 world units
  data.sound_type      = idle
  data.sound_delay     = the creature's configured idle-sound spacing
```

## `execute`

**Contract** — Refreshes the destination to the target's current position and navigation vertex and the path-rebuild interval to the creature's configured attack rebuild time, then runs the base state's tick. The refresh is what makes this a pursuit rather than a move to a fixed spot.

## `check_start_conditions`

**Contract** — Legal only while the target is farther than eight units.

## `check_completion`

**Contract** — Done when the base state says so, or as soon as the target is nearer than eight units — whichever comes first. The distance test is what hands control back to the display before the creature gets close enough to look committed.
