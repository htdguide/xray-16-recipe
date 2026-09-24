# src/xrGame/smart_cover_animation_planner_inline.h

> The planner's timing accessors, and the two random dwell draws that keep a squad in cover from moving in unison.

**Needs** — [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: field reads and two random draws

## Purpose

Carries the bodies for
[`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md). One thing here is
load-bearing: the dwell intervals are *drawn*, not stored.

## `default_idle_interval` / `default_lookout_interval`

**Contract** — each returns a fresh uniform draw between the authored minimum and maximum
for its phase, converted from seconds to milliseconds. Every call is a new draw; there is
no cached value.

```text
FUNCTION default_idle_interval() -> int (ms)
  RETURN 1000 * uniform(idle_min_time, idle_max_time)
```

**Invariants** — the draw comes from the **planner's own** random stream, seeded per
instance, not from the global one. That is what makes two creatures in the same cover, with
the same authored bounds, peek out at different moments. A rebuild that shares one stream
across creatures will produce a squad that bobs up in unison, which is the single most
noticeable failure mode of this system.

**Notes** — the conversion truncates toward zero, so an authored minimum below a
millisecond yields zero and the phase expires on the next evaluation. Authored bounds are
whole or half seconds in shipped data, so this has never mattered.

## Phase state accessors

**Contract** — `stay_idle`, `last_idle_time` and `last_lookout_time` each read and write a
field. They are the shared state of the two dwell evaluators in
[`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md), which is why they are
writable from outside.

## Dwell bound accessors

**Contract** — `idle_min_time`, `idle_max_time`, `lookout_min_time` and `lookout_max_time`
read and write the four bounds, in seconds. They are written by the loophole actions when a
creature takes up a loophole, from the authored cover data.

## `time_object_hit` / `last_transition_time`

**Contract** — field reads (and a write for the transition stamp) of two global-clock
stamps in milliseconds.

## `loophole_value` / `decrease_loophole_value`

**Contract** — read and subtractive update of the decaying loophole preference. The
subtraction is unguarded: there is no floor, and the value is an unsigned count, so a large
enough decrease wraps. Nothing reads it, so nothing depends on that.

## `property_storage` / `cName`

**Contract** — the planner's own world state, handed to nested machinery that needs to read
or pin properties, and a fixed diagnostic name.
