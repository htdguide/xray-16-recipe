# src/xrGame/ai/monsters/control_run_attack.cpp

> The run-through attack: a creature already running at its enemy plays a strike clip and builds a line that carries it exactly as far as the clip lasts.

**Needs** — [`control_run_attack.h`](control_run_attack.h.md) · [`control_animation_base.h`](control_animation_base.h.md) · [`control_direction_base.h`](control_direction_base.h.md) · [`control_movement_base.h`](control_movement_base.h.md) · [`monster_velocity_space.h`](monster_velocity_space.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [Seam: Configuration](../../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: builds a path and drives the body for the duration of a clip

## Purpose

A strike delivered without stopping. Where the jump leaves the ground and the melee jump
stands still, this ability keeps the creature at running speed through the whole attack: it
seizes the body, faces the enemy, starts the strike clip, and — once the clip has actually
begun and its duration is known — builds a straight line exactly clip-duration long at
running speed and hands the body to it.

Doing the line *after* the clip starts rather than before is the distinguishing decision,
and it is why this ability subscribes to animation start as well as animation end.

## State

```text
RECORD RunAttack
  min_dist, max_dist  : real       # authored distance band to the enemy
  min_delay, max_delay: int (ms)   # authored cooldown window
  time_next_attack    : int (ms)
```

No payload: the ability has no per-use parameters. The clip is looked up by a fixed name.

## `load`

**Contract** — read two authored entries from the creature's configuration section:
`Run_Attack_Dist`, a distance band, and `Run_Attack_Delay`, a cooldown window. Both are
two-value lines parsed as a low and a high.

**Notes** — the delay is a *window*, not a value, and the actual cooldown is drawn uniformly
from it after each attack. As with the rotation jump, the randomization keeps a pack from
striking in unison.

## `check_start_conditions`

**Contract** — five tests: the ability is not running; no other ability holds the body; an
enemy exists; the creature is facing the enemy within a twelfth of a turn; the distance to
the enemy is inside the authored band; the creature's speed is within two units of its
authored running speed; and the cooldown has expired.

**Notes** — the facing tolerance of a twelfth of a turn and the speed tolerance of two
units are constants here, not authored. The speed tolerance is the same number the rotation
jump uses, spelled separately.

## `activate`

**Contract** — seize the body, subscribe to animation start and end, stop the path and the
movement, set a fixed turn rate aimed at the enemy, and start the strike clip — looked up
on the creature's model by the literal name `stand_attack_run_0`.

**Notes** — the clip name is hard-coded, so a creature granted this ability must have a clip
by exactly that name. This is the only ability in the chapter whose clip is not supplied by
the creature through its payload; a rebuild should move it there.

The turn rate of three radians per second is a constant here, in contrast with every other
ability's derived rate. It is fast enough to track a sidestepping enemy through the strike.

## `on_event` — animation start

**Contract** — the clip has begun, so its true duration is now known. Compute how far the
creature travels at running speed in that time, build a line that far ahead toward the
enemy restricted to the running gait or standing, enable and lock the path, and command the
running speed instantly. A line that cannot be built ends the ability.

```text
ON animation_start
  anim_time <- blend's total time / its playback rate
  velocity  <- the creature's authored running gait
  distance  <- anim_time * velocity.linear
  target    <- my position + normalize(enemy - me) * distance
  IF a line to target cannot be built with (run OR stand) THEN
    raise run_attack_end
  ELSE
    enable and LOCK the path
    movement target <- velocity.linear, instantly
```

**Notes** — the duration is read from the *live blend* rather than from the clip, so it
accounts for whatever playback rate the animation base chose. That is the reason the line
is built here and not in `activate`: before the clip starts there is no blend to ask.

The line is aimed along the direction to the enemy at the moment of the strike and then
committed to. The creature therefore runs *through* where the enemy was, not to where it
is — which is the manoeuvre.

## `on_event` — animation end

**Contract** — draw a new cooldown from the authored window and raise the run-attack-end
event.

## `on_release`

**Contract** — unlock the path, release the body, unsubscribe from both events.
