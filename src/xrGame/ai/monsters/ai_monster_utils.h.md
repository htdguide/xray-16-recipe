# src/xrGame/ai/monsters/ai_monster_utils.h

> The small shared arithmetic of the creature layer: angle tests, rate-limited approach, navigation-position validation, and the two configuration-line shapes creatures use.

**Needs** — [`ai_monster_utils.cpp`](ai_monster_utils.cpp.md) · [`xrCore/xr_ini.h`](../../../xrCore/xr_ini.h.md) · [`xrEngine/device.h`](../../../xrEngine/device.h.md)
**Used by** — [`ai_monster_utils.cpp`](ai_monster_utils.cpp.md) · [`base_monster.h`](basemonster/base_monster.h.md) · [`bloodsucker_alien.cpp`](bloodsucker/bloodsucker_alien.cpp.md) · [`controlled_actor.cpp`](controlled_actor.cpp.md) · [`controller_direction.cpp`](controller/controller_direction.cpp.md)
**Tier floor** — T3: scalar arithmetic and two text-parsing shapes

## Purpose

A header of free functions, most of them small enough to be inlined. It is the substance
holder for everything except the four routines implemented in
[`ai_monster_utils.cpp`](ai_monster_utils.cpp.md). Nothing here is deep, but several of
these are used in dozens of places and getting one of them subtly wrong changes how every
creature moves, so they are written out.

## `random_position`

**Contract** — a point uniformly offset from a centre within a square of half-width `R`,
**in the horizontal plane only** — the height is copied unchanged. Every caller intends a
position on the ground and hands the result to something that resolves it onto the
navigation mesh.

**Notes** — the offset is drawn per-axis, so the distribution is a square, not a disc. No
caller depends on it being either.

## `from_right`

**Contract** — is the target heading to the right of the current heading, taking the
shortest way round. The whole of the creature layer's left/right turn selection is this one
comparison.

## `is_angle_between`

**Contract** — is a heading inside the arc bounded by two others. Requires the arc to be
**less than half a turn** and asserts it, because an arc of half a turn or more has no
unambiguous inside.

```text
FUNCTION is_angle_between(yaw, from, to) -> bool
  span = shortest angular difference between from and to
  FAIL WITH ambiguous arc IF span >= PI
  RETURN shortest_diff(yaw, from) < span AND shortest_diff(yaw, to) < span
```

**Notes** — the test is "closer to each endpoint than the endpoints are to each other",
which is exactly inside-ness for a sub-half-turn arc and is why the precondition is not
optional. It is used for melee hit cones, so a creature with a cone of half a turn or wider
cannot be expressed and must be given the no-trace flag instead.

## `velocity_lerp`

**Contract** — moves a current speed toward a target at a fixed acceleration over a time
step. Snaps to the target when it would overshoot upward, but when moving **downward it
clamps at zero, not at the target**.

**Notes** — the asymmetry is the one interesting thing in this file. Decelerating past the
target all the way to zero is what makes a creature that is told to slow down actually
stop and then re-accelerate, rather than settling smoothly at the lower speed. Whether that
was intended is not recoverable; the symmetric version is right beside it as `def_lerp` and
is used where a smooth settle is wanted. A rebuild must keep both, because creature
movement is tuned against this behaviour.

## `def_lerp`

**Contract** — the symmetric version: moves toward the target at a rate and snaps to it from
either side.

## `time`

**Contract** — the global clock in milliseconds. Named locally so the creature layer's
call sites read as domain code; it is a reach into the device singleton and a rebuild
passes a clock.

## `read_delay`

**Contract** — reads a configuration value as a delay *range*. A two-value line gives the
minimum and the maximum; a single value gives a maximum with a minimum of zero. Values are
integer milliseconds.

**Notes** — the single-value form meaning "zero to N" rather than "exactly N" is the
convention every creature's sound and action pacing depends on, and it is stated nowhere
but here.

## `read_distance`

**Contract** — reads a configuration value as a distance range. Requires exactly two values
and asserts it — unlike the delay reader, there is no single-value shorthand.

## `get_valid_position` / `object_position_valid`

**Contract** — see [`ai_monster_utils.cpp`](ai_monster_utils.cpp.md).

## `get_bone_position` / `get_head_position`

**Contract** — see [`ai_monster_utils.cpp`](ai_monster_utils.cpp.md).
