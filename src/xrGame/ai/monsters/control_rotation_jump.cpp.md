# src/xrGame/ai/monsters/control_rotation_jump.cpp

> The rotation jump: a running creature that finds its enemy behind it skids to a stop through a turn, then accelerates back out toward the enemy.

**Needs** — [`control_rotation_jump.h`](control_rotation_jump.h.md) · [`control_manager.h`](control_manager.h.md) · [`control_animation_base.h`](control_animation_base.h.md) · [`control_direction_base.h`](control_direction_base.h.md) · [`control_movement_base.h`](control_movement_base.h.md) · [`monster_velocity_space.h`](monster_velocity_space.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — reached through its declarations in [`control_rotation_jump.h`](control_rotation_jump.h.md); callers name that, not this file.
**Tier floor** — T2: builds paths and drives the body for the duration of two clips

## Purpose

A two-stage turn-around at speed. It exists to solve a specific visual problem: a creature
running past its enemy cannot simply reverse its heading, because the body would pivot in
place while still travelling at running speed. Instead it plays a skid-and-turn clip along
a computed deceleration line, then an accelerate-out clip along a second line aimed at the
enemy — each line exactly as long as the clip that covers it and each with the
deceleration or acceleration that carries the creature from one end speed to the other.

Both lines are computed from the same three facts: the clip's duration, the speed at each
end, and constant acceleration.

## State

```text
RECORD RotationJump
  stage             : one of { stop, run, none }
  right_side        : bool                  # which way the turn goes
  start_velocity    : real
  target_velocity   : real
  accel             : real
  dist              : real                  # the length of the line for this stage
  time              : real (seconds)        # the clip's duration
  time_next_allowed : int (ms)

RECORD RotationJumpPayload
  anim_stop_ls,  anim_run_ls  : Motion      # the left-side pair
  anim_stop_rs,  anim_run_rs  : Motion      # the right-side pair
  turn_angle    : real                      # how far the first stage turns
  flags : { stop_at_once, rotate_once }
```

The two flags change the ability substantially:

- **stop at once** — skip the deceleration line entirely and turn on the spot. The
  creature must already be stopping for this to look right.
- **rotate once** — use only the first stage; the creature ends facing the new direction
  and does not accelerate out. Combined with *stop at once*, the first stage turns to face
  the enemy directly rather than by the authored turn angle.

## Authored parameters

The clips and the turn angle come from the creature's own load through its custom manager;
the four constants below are in this file, not in data:

| Constant | Value | Meaning |
|---|---|---|
| cooldown | uniform in 3000–5000 ms | earliest the next rotation jump may start |
| facing check | 150 degrees | the enemy must be at least this far off the creature's facing |
| speed tolerance | 2 world units per second | how close to full running speed the creature must be |

## `check_start_conditions`

**Contract** — five tests, all of which must pass: the ability is not already running; no
other ability holds the body; the creature has an enemy; the cooldown has expired; the
enemy is *not* within 150 degrees of the creature's facing — that is, the enemy is behind
it; and the creature's current speed is within two units of its authored running speed.

**Notes** — the last test is what makes this a *running* manoeuvre. A walking creature that
finds its enemy behind it turns normally; only one at speed pays for the skid.

## `activate`

**Contract** — seize the whole body, subscribe to animation end, stop the path and the
movement, decide which side the turn goes by which side the enemy is on, and then either
turn on the spot or build the first line.

## `build_line_first` — the skid

**Contract** — decelerate to a stop along a straight line while playing the turn clip.

```text
FUNCTION build_line_first()
  time            <- duration of the side-appropriate stop clip
  start_velocity  <- the creature's current speed
  target_velocity <- 0
  accel           <- (target - start) / time
  dist            <- (target^2 - start^2) / (2 * accel)

  heading target  <- my facing rotated by the authored turn angle, toward the turn side
  heading rate    <- (heading error) / time        # arrive exactly as the clip ends
  linear dependency <- off

  target_point <- my position + my facing * dist
  IF a line to it cannot be built with (stand OR run) THEN
    raise rotation_jump_end
  ELSE
    enable and LOCK the path
    movement target <- 0 with the computed deceleration
    start the stop clip
```

**Notes** — the distance formula is the textbook one and, with a target velocity of zero,
reduces to the start speed squared over twice the deceleration. Writing it in the general
form is what lets the second stage reuse it unchanged.

Turning at exactly the heading error divided by the clip duration is the same trick the
jump uses: the body's turn and the clip's turn finish together. Linear dependency must be
off for it to hold, since the body is decelerating.

## `build_line_second` — the burst out

**Contract** — accelerate from rest back to the original running speed along a line aimed at
the enemy, playing the run clip. Same formulae with the two velocities swapped. If the
enemy is gone by now, the ability ends instead.

**Notes** — the target speed is the speed the creature *had* when the manoeuvre began, not
its authored running speed, so a creature that was running at less than full speed comes
out at the same reduced speed.

## `stop_at_once`

**Contract** — the *stop at once* variant: no line, no movement command. The creature turns
in place over the duration of the stop clip. When *rotate once* is also set and an enemy
exists, the turn aims directly at the enemy; otherwise it turns by the authored angle.

## `on_event`

**Contract** — on animation end: if the first stage just finished and *rotate once* is not
set, begin the second stage; otherwise raise the rotation-jump-end event.

## `on_release`

**Contract** — unlock the path, restore linear dependency, release the body, unsubscribe,
and set the cooldown to a uniform random point between three and five seconds ahead.

**Notes** — randomizing the cooldown rather than fixing it keeps a pack of creatures from
performing the manoeuvre in unison, which is visible and strange. The same pattern recurs
in the melee jump with a shorter window.
