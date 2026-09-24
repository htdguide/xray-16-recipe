# src/xrGame/ai/monsters/control_animation_base_accel.cpp

> Acceleration and braking: how fast a creature may change speed, which clip matches the speed it is actually moving at, and when to start slowing down for the end of the path.

**Needs** — [`control_animation_base.h`](control_animation_base.h.md) · [`control_path_builder_base.h`](control_path_builder_base.h.md) · [`control_movement.h`](control_movement.h.md) · [`detail_path_manager.h`](../../detail_path_manager.h.md) · [`monster_velocity_space.h`](monster_velocity_space.h.md) · [Seam: Configuration](../../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: runs on the frame tick and walks the path's waypoint list

## Purpose

Three related jobs that all come down to "the clip and the ground speed must agree".

First, a creature has two authored acceleration rates — a calm one and an aggressive one —
and whichever is in force bounds how fast its movement speed may change. Second, a set of
clips covering overlapping speed ranges forms a *chain*, and given the body's measured
speed the chain says which clip to play and how fast to play it. Third, a creature
approaching a stop must begin decelerating early enough that it arrives at walking pace
rather than skidding, which means looking ahead along the path for the point where it must
stop.

## State

Operates on the acceleration fields of
[`control_animation_base.h`](control_animation_base.h.md). Owns nothing else.

## `accel_load`

**Contract** — reads the two authored acceleration rates from the creature's configuration
section, under the keys `Accel_Calm` and `Accel_Aggressive`. Both are required. Units are
world units per second squared.

**Notes** — these are the only two numbers in the whole acceleration machinery that come
from data. Everything else is derived from the authored clip speeds.

## `accel_activate` / `accel_deactivate` / `accel_set_braking`

**Contract** — turn bounded acceleration on with a chosen rate (calm or aggressive), or
off. Activating also enables braking; deactivating disables both. Braking can be toggled
separately, because a creature may want bounded acceleration while refusing to slow down.

## `accel_get`

**Contract** — the acceleration in force, or an unbounded value when the relevant mode is
off. The unbounded value is the chapter's convention for *instant*: every consumer divides
by it or passes it straight to the movement channel, where an unbounded rate means the
target speed is reached this frame.

## `accel_chain_add`

**Contract** — declare that two clips form a speed chain, slower first. Chains are
independent of each other; a creature may have one for its healthy gait and another for
its wounded gait.

**Notes** — the registration takes exactly two clips and pushes them as a new chain, so a
three-stage chain cannot be declared through this entry point even though the lookup
handles arbitrary lengths. No shipped creature needs one.

## `accel_chain_get`

**Contract** — given the body's measured speed and the clip the selector wants, find the
chain containing that clip and return the chain member whose authored speed range covers
the measurement, together with the playback rate that makes the clip's feet match the
ground. Answers whether such a chain existed. Does not allocate.

```text
FUNCTION chain_lookup(current_speed, wanted_motion) -> optional<(motion, play_rate)>
  FOR EACH chain IN chains
    best   <- none
    found  <- false
    FOR EACH member IN chain
      range <- [member.linear * member.min_factor, member.linear * member.max_factor]
      # the first and last members also absorb everything below and above the chain
      IF current_speed IS IN range
         OR (current_speed < range.low  AND member IS first in chain)
         OR (current_speed >= range.high AND member IS last in chain) THEN
        best <- member
      IF member IS wanted_motion THEN found <- true
      IF found AND best EXISTS THEN BREAK
    IF NOT found THEN CONTINUE
    REQUIRE best EXISTS                 # otherwise the authored ranges have a hole
    rate <- member_play_rate(best) * current_speed / best.linear
    IF rate < 0.5 THEN rate <- rate + 0.5
    RETURN (best, rate)
  RETURN none
```

**Notes** — two decisions here deserve naming. The first and last members of a chain absorb
everything outside the chain's range, so a creature moving faster than its fastest clip
was authored for plays that clip sped up rather than falling through to nothing. And the
playback rate is floored by *adding* half rather than clamping: a rate below one half
becomes that rate plus one half, so a very slow crawl plays at somewhere between a half
and one rather than at a fixed floor. Nothing records why the correction is additive; the
visible effect is that a creature inching forward still moves its legs.

The search stops at the first chain containing the wanted clip, so a clip must not appear
in two chains.

## `accel_chain_test`

**Contract** — a debug-build authoring check, run once per creature at load. Walks each
chain and asserts that consecutive members' authored speed ranges overlap: the earlier
clip's upper bound must exceed the later clip's lower bound. Names the creature and both
clips in the failure.

**Notes** — this is the only validation of the animation tables anywhere, and it catches
the one authoring error that produces a silent freeze rather than a visible glitch. A
rebuild should keep it and should consider running it in release too.

## `accel_check_braking`

**Contract** — decide whether the creature should be decelerating right now. Answers yes
when the remaining path is shorter than the distance needed to come to rest at the braking
rate, or when a waypoint that demands a stop lies within that distance. Records the answer
so the next call can use the *nominal* speed rather than the current one, which keeps the
decision from oscillating once braking has begun. Answers no whenever the creature is not
moving on a path or braking is disabled.

```text
FUNCTION check_braking(lead_in, nominal_speed) -> bool
  IF not moving on a path OR braking disabled THEN
    braking <- false; RETURN false

  a <- braking acceleration
  # while already braking, assume the nominal speed; otherwise use the measured one
  v <- IF braking THEN nominal_speed ELSE current movement speed
  distance <- nominal_speed * v / (2 * a) + lead_in

  IF the path ends within `distance` THEN
    braking <- true; RETURN true

  walk forward from the current waypoint accumulating segment lengths
  AT the first waypoint whose speed is "stand"
    braking <- (accumulated distance < distance)
    STOP
  RETURN braking
```

**Notes** — the caller passes a *negative* lead-in of two world units, which shortens the
braking distance rather than lengthening it. Read plainly, that means the creature is
allowed to overshoot its stopping point by two units before it is considered to be
braking. Nothing in the source explains the number or its sign; it is tuned, and it is one
of the values a rebuild will have to re-tune against the original's look.

The mixed use of nominal and measured speed in the braking distance is deliberate
hysteresis, not an error: once braking, the creature computes its distance from the speed
it *would* be travelling, so a momentary slowdown does not cancel the brake.
