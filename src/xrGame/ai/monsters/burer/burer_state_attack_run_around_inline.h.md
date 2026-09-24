# src/xrGame/ai/monsters/burer/burer_state_attack_run_around_inline.h

> Three range bands decide where a burer goes when it cannot attack: close in, back off, or sidestep somewhere random.

**Needs** — [`burer_state_attack_run_around.h`](burer_state_attack_run_around.h.md) · [`burer.h`](burer.h.md) · [`control_path_builder_base.h`](../control_path_builder_base.h.md) · [`control_direction_base.h`](../control_direction_base.h.md)
**Used by** — [`burer_state_attack_run_around.h`](burer_state_attack_run_around.h.md)
**Tier floor** — T3: target selection and a movement command

## Purpose

The burer's only movement decision. It is chosen either because the creature has just been hurt or because the enemy has got too close, so its job is to change the geometry — not to reach any particular place.

## State

```text
RECORD BurerAttackRunAroundState
  destination      : position
  time_started     : int
  arrival_facing   : direction    # zero means "no preferred facing"
```

```text
step = 10 world units    # every destination is exactly this far from the start
```

Read from the creature: `max_runaway_time`.

## `initialize` — the three bands

**Contract** — Chooses the destination and the arrival facing once, at entry, from the distance to the enemy. Prepares the path builder.

```text
FUNCTION initialize()
  time_started   = now()
  arrival_facing = none
  distance       = |enemy - self|

  IF distance > 30 world units
     destination = self + unit(toward enemy) * step       # close the gap
     # no arrival facing: the creature is already heading at the enemy
  ELSE IF distance < 20 AND distance > 4
     destination = self + unit(away from enemy) * step    # break off
     arrival_facing = unit(enemy - destination)
  ELSE
     destination = a random position within `step` of own position
     arrival_facing = unit(enemy - destination)

  prepare the path builder
```

**Notes** — The bands do not cover the whole line, and the gaps are the interesting part. Between twenty and thirty units the creature falls into the *random* band, and inside four units it does too. So a burer at medium range mills about instead of retreating, and a burer with the player right on top of it sidesteps rather than backing off — which, given that the tree selects this state precisely when the player is inside the run-away distance, is the common case. A rebuild that "fixes" the bands into a clean partition produces a creature that always retreats cleanly and is much easier to corner.

Every destination is exactly ten units away regardless of band, so the move always takes about the same time.

## `execute`

**Contract** — Issues the run each tick: run action, target the chosen point, generic path parameters, covers disabled, aggressive vocalisation. Sets the destination orientation only when an arrival facing was chosen.

**Notes** — Covers are disabled because a burer does not want cover — it wants distance and a clear line for its wave. Asking the pathfinder for a covered route would send it behind geometry that its own attacks cannot shoot through.

## `check_start_conditions`

**Contract** — Always true. The decision of *whether* to reposition belongs entirely to the attack tree.

## `check_completion`

**Contract** — Done when the authored maximum runaway time has elapsed, or when the creature is following a path and is within two units of its end. On either, it turns to face the enemy before reporting completion.

**Notes** — The timeout is the load-bearing half: a burer whose destination turns out to be unreachable gives up after the authored time rather than grinding against geometry. Turning to face the enemy inside the completion test, rather than in a teardown, means the creature is already aimed when the tree gets control back and can fire on the next tick.
