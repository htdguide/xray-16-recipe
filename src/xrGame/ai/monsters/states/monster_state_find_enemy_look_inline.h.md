# src/xrGame/ai/monsters/states/monster_state_find_enemy_look_inline.h

> A five-step sweep of the area where contact was lost: look, swing 120 degrees to one side, look,
> swing another 120, look — each swing randomly either a turn in place or a short dash.

**Needs** — [`monster_state_find_enemy_look.h`](monster_state_find_enemy_look.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md) · [`state_look_point.h`](state_look_point.h.md) · [`state_custom_action.h`](state_custom_action.h.md) · [`state_data.h`](state_data.h.md)
**Used by** — [`monster_state_find_enemy_look.h`](monster_state_find_enemy_look.h.md)
**Tier floor** — T3: geometry over a heading plus a randomized transition table

## Purpose

The search's only leaf that actually searches. Its job is to cover the arc around the last known
position in a way that (a) looks like an animal casting about rather than a turret sweeping, and
(b) actually moves the creature's vision cone over the hiding places a player is likely in.

The design achieves both with one trick: the sweep is **anchored to the creature's facing at the
moment it entered**, not to the world, and it always turns the *same* way — chosen at random on
entry. A creature that skidded to a halt facing roughly where it last saw the player therefore
sweeps outward from that direction, and two creatures searching together sweep opposite ways about
half the time.

## State

```text
RECORD FindEnemyLookState
  look_right_side : bool     # handedness, chosen once on entry
  current_stage   : int      # 0..5, incremented on every reselect
  current_dir     : vector   # the heading being swung, starts at entry facing
  start_position  : vector   # entry position; every probe point is measured from here
  target_point    : vector   # the point currently being turned toward or run to
```

**Invariant** — `current_dir` and `start_position` are the sweep's frame of reference and are
written exactly once, on entry. Every probe point is `start_position + current_dir * r`, so the
creature always probes around where it *entered*, even after a dash has moved it.

## `initialize`

**Contract** — choose the sweep handedness at random, zero the step counter, and snapshot the
creature's current facing and position as the sweep's frame of reference.

**Notes** — the random handedness is the cheapest possible way to stop a pack from looking
choreographed, and it is chosen per entry, so the same creature searches left this time and right
the next.

## `reselect_state` — the sweep

**Contract** — on steps 1 and 3, rotate the reference heading 120 degrees toward the chosen side,
project a probe point four to five units out from the entry position along the new heading, and
randomly either dash to it or turn to face it. On every other step, look around in place. Increment
the step counter each time.

```text
FUNCTION next_step()
  IF current_stage IN {1, 3}
    heading, pitch = decompose(current_dir)
    heading = heading + (look_right_side ? -120 degrees : +120 degrees)
    current_dir  = normalize(recompose(heading, pitch))
    target_point = start_position + current_dir * random_real(4, 5)
    step = random_bool() ? dash_to_point : turn_to_point
  ELSE
    step = look_around
  current_stage = current_stage + 1
```

**Notes** — several decisions are packed into those lines.

*120 degrees, applied twice, covers 240 of the 360.* The creature ends up having faced its entry
heading, that heading plus 120, and plus 240 — three directions evenly spaced around the circle.
The one direction never explicitly faced is the one it came from, which is also the one it just
ran through and can be assumed clear. Splitting a full circle into three is why the number is 120
and not 90 or 180.

*The pitch is preserved through the rotation.* Only the heading is changed; a creature that entered
looking slightly up or down keeps that. The rotation therefore never makes a creature stare at the
sky or the floor.

*Dash or turn, at even odds.* Both alternatives end facing the same probe point; only whether the
creature's body travels there differs. That single coin flip is what stops the sweep from being a
recognizable animation loop: run/look/turn/look and turn/look/run/look read as different animals.

*The probe distance is randomized in a one-unit band.* Four to five units is short — the dash is a
reposition, not a pursuit — and the randomization keeps two creatures from stacking on the same
spot.

*Steps 0, 2 and 4 are the look-arounds*, so the sweep is look, move, look, move, look. The counter
runs 0 to 4 and completion fires at 5, which is why the rotate-and-move branch tests exactly
`{1, 3}` rather than testing parity.

## `setup_substates` — the parameters of each step

**Contract** — fill the parameter record of whichever step was just selected.

```text
dash_to_point:
  point           = target_point
  vertex          = unknown, let the path builder resolve it
  gait            = run, accelerating, no braking, aggressive profile
  voice           = aggressive, repeat delay = section key "attack_sound_delay"

look_around:
  action   = look around
  time_out = 2000 ms                       # hard-coded
  voice    = aggressive, delay = section key "attack_sound_delay"

turn_to_point:
  point    = target_point
  action   = stand idle
  voice    = aggressive, delay = section key "attack_sound_delay"
```

**Notes** — only the voice repeat delay is authored per creature (`attack_sound_delay`); the
two-second look and the gait choices are compiled in.

The dash leaves the target vertex unresolved and lets the path builder find the nearest walkable
cell to the probe point. That is correct here: the probe point is computed from geometry alone and
may well be inside a wall, and the recovery is "run as close as you can", not "fail the step".

The turn step has no timeout, so it completes when the turn does — its duration is whatever the
creature's turn rate makes it. A slow, heavy creature therefore spends longer on a turn than a fast
one, without any per-creature number saying so.

## `check_completion`

**Contract** — finished once five steps have been taken.

**Notes** — five is hard-coded and is what makes the sweep exactly three looks and two
rotations. Changing it to an even number would end the sweep on a movement rather than on a look,
which reads as the creature losing interest mid-stride.
