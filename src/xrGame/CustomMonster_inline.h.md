# src/xrGame/CustomMonster_inline.h

> The small, hot operations of the creature base: a bounded angle turn that cannot overshoot, a normalization that cannot fail, and the accessors for the four subsystems.

**Needs** — [`CustomMonster.h`](CustomMonster.h.md)
**Used by** — [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`CustomMonster_VCPU.cpp`](CustomMonster_VCPU.cpp.md)
**Tier floor** — T2: per-tick arithmetic that must not allocate or branch much

## Purpose

Header-resident because it is called several times per creature per tick, but two of the
routines carry real decisions and belong in the recipe on their own merits. The rest are
accessors that exist to assert their subsystem is present before handing it over — an
artifact of the subsystems being separately-allocated and constructed in a specific order.

## `angle_lerp_bounds`

**Contract** — turns an angle toward a target at a given rate over a given time, and answers
whether it arrived. If the allowance for this tick equals or exceeds the remaining
difference, the angle is set exactly to the target and the answer is true; otherwise it is
moved by the allowance and the answer is false.

```text
FUNCTION turn_toward(current, target, rate, dt) -> arrived
  IF rate * dt >= angular_difference(current, target) THEN
    current = target
    RETURN true
  current = angle_lerp(current, target, rate, dt)      # takes the short way round
  RETURN false
```

**Invariants** — this is the reason creatures never jitter around a facing. A plain rate
integration overshoots by up to one tick's worth and then comes back, which at a 100 to
250 millisecond tick is a visible twitch on every turn. The snap-on-arrival is what removes
it, and the returned flag is what lets a caller know a turn has completed without
re-comparing angles.

## `vfNormalizeSafe`

**Contract** — normalizes a vector, substituting a fixed unit direction along the first
axis when the vector is too short to have a direction. Never fails, never yields a
not-a-number.

**Notes** — the substitute is an arbitrary direction, which is only acceptable because
every caller is asking "which way is this creature facing" about something that has no
meaningful facing. A rebuild that returns an optional instead will find callers that have
to invent the same arbitrary answer.

## `left_angle`

**Contract** — answers whether the second heading lies to the left of the first, by the sign
of the cross product of their unit vectors. A free function, not a member.

## `panic_threshold`

**Contract** — the health fraction below which this species panics.

## `client_update_delta` / `client_update_fdelta` / `last_client_update_time`

**Contract** — the clamped per-frame elapsed time, in milliseconds and in seconds, and when
the last frame tick ran. The clamp is applied where the value is computed
([`CustomMonster.cpp`](CustomMonster.cpp.md)); these only publish it.

## `critical_wound_type` / `critically_wounded` / `critical_wounded_state_stop`

**Contract** — the critical-wound flag, expressed as a sentinel-valued integer: a specific
out-of-range value means "not wounded", and stopping the state writes that value back.
A rebuild should use an optional and delete the sentinel.

## `invulnerable`

**Contract** — read and write the flag that makes damage vanish entirely.

## `memory` / `movement` / `sound` / `sound_user_data_visitor` / `get_moving_object`

**Contract** — the five subsystem accessors. Each asserts its subsystem exists before
returning it, because they are created in a specific order during construction and
anything reaching for one too early is a real bug. In a rebuild that constructs a creature
with its subsystems in one step, the assertions vanish along with the possibility.
