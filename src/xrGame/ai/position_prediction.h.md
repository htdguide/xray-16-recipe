# src/xrGame/ai/position_prediction.h

> Leads a moving target: estimate where the enemy will be by the time I get there. Compiled into the game and included by nothing.

**Needs** — _(none)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: two divisions and a sampled velocity

## Purpose

A self-contained value type that turns "where is the enemy now" into "where will the enemy
be when my shot, or my charge, arrives". It is the standard lead-the-target computation, and
it would be used by a creature deciding where to aim a leap or a throw.

**Nothing includes this file.** It appears in the build description as a header and in no
other source file. A rebuilder should treat the computation as a useful piece to have and
the file itself as dead — and, before reimplementing it, should look for the lead
calculations that the shipped creatures actually use, which are elsewhere.

## State

```text
RECORD PositionPrediction
  estimated_velocity : vector3        # the enemy's velocity, re-estimated on a slow cadence
  last_sample_time   : int (milliseconds)   # 0 means "never sampled"
  last_sample_position : vector3
```

**Invariants** — the velocity estimate is *not* the enemy's actual velocity, which the
engine knows; it is a finite difference over the predictor's own sampling interval. That
makes the estimate independent of how the enemy's motion is produced — a script-driven
teleport and a walk both register — at the cost of lag.

## `calculate_predicted_enemy_pos`

**Contract** — given a lead factor, the enemy's position, the predictor's own position and
its own speed, returns the position to aim at. Re-samples the enemy's velocity at most once
every 400 milliseconds; a gap longer than two seconds discards the estimate rather than
dividing a large displacement by a large interval, since across such a gap the enemy was
probably not observed at all. Pure apart from the sampling state it carries.

```text
FUNCTION predict(lead_factor, enemy_pos, self_pos, self_speed) -> vector3
  separation   = magnitude(enemy_pos - self_pos)
  time_to_reach = self_speed > 0.0001 ? separation / self_speed : 0
      # zero speed yields zero lead, not an infinite one

  elapsed = (now() - last_sample_time) / 1000
  IF elapsed > 0.4                             # the sampling cadence
    IF last_sample_time != 0                   # skip the very first call: no baseline yet
      IF elapsed < 2.0
        estimated_velocity = (enemy_pos - last_sample_position) / elapsed
      ELSE
        estimated_velocity = zero               # the observation gap was too long to trust
    last_sample_time     = now()
    last_sample_position = enemy_pos

  RETURN enemy_pos + estimated_velocity * time_to_reach * lead_factor
```

**Invariants** — the lead factor is a separate multiplier on top of a physically correct
lead, which is how a difficulty setting or a creature's skill would weaken the prediction
without changing the maths. A factor of one leads perfectly; zero aims at the present
position.

**Notes** — the 400 millisecond cadence and the 2 second gap threshold have no derivation in
the source. The cadence is long enough that the estimate is smooth and short enough that a
lead computed from it is not badly stale at walking speed; the gap threshold is roughly the
interval over which an unobserved target could have gone anywhere.
