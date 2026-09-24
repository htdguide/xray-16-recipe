# src/xrGame/ai/monsters/states/monster_state_attack_run_inline.h

> Closing with the enemy: a path request re-issued every tick against the enemy's current navigation vertex, with route extrapolation on so the creature aims where the enemy is going, and a squad command that can override which way it faces when it gets there.

**Needs** — [`monster_state_attack_run.h`](monster_state_attack_run.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`../../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`monster_state_attack_run.h`](monster_state_attack_run.h.md)
**Tier floor** — T3: a path request with cover weighting and a squad-supplied facing

## Purpose

The default way a monster gets to its target. Four decisions distinguish it from a plain "walk
to a point": it targets a *vertex* rather than a position, it extrapolates, it weights the route
toward cover, and it lets the squad dictate the arrival facing.

## `initialize`

**Contract** — tells the path builder a new request is starting, so the previous route is
discarded rather than extended.

## `execute`

**Contract** — one tick. Re-asserts the whole path request, chooses between running and standing
idle, sets the sound, enables extrapolation, and consults the squad for an arrival direction.

```text
FUNCTION execute()
  animation.accelerate(aggressive, braking: false)

  enemy_vertex = the mesh vertex my enemy occupies
  path.target  = (position of enemy_vertex, enemy_vertex)

  IF enemy_vertex == my own vertex
    set_action(stand_idle)          # we are on the same cell: stop, don't jitter
  ELSE
    set_action(run)

  path.rebuild_interval = creature.attack_rebuild_time      # authored per creature
  path.use_covers       = true
  path.cover_weights    = (0.1, 30, 1, 30)
  path.try_min_time     = false                             # prefer a short route, not a fast one
  state_sound           = aggressive
  path.extrapolate      = true

  path.use_destination_facing = false
  squad = my squad
  IF squad EXISTS AND squad is active
    command = squad.command_for(me)
    IF command IS an attack command
      path.use_destination_facing = true
      path.destination_facing     = command.direction
```

**Invariants**

- **The target is a mesh vertex, not a position.** That matters because the enemy's raw position
  may be somewhere the creature cannot stand — mid-air, on a ledge, inside a doorway — and
  snapping to the vertex gives the pathfinder a destination it can actually reach. It also means
  the creature converges on a cell rather than on a point, which is what allows several
  creatures to converge on one target without fighting over the same coordinate.
- **Same-vertex means stop.** Without this the creature would keep issuing a zero-length route
  and jitter in place; standing idle hands the situation to the melee test on the next tick.
- **Extrapolation is on.** The builder is told to lead the target rather than chase its current
  cell, which is what makes a charging monster intercept a running player rather than trail
  behind them. It is turned off again on *both* exits — completion and displacement — because it
  is a builder-wide setting and leaving it on would affect whatever behaviour runs next. That
  pairing is the one piece of tear-down this state has and it is easy to lose.
- **Cover weighting is on during an attack**, which is not obvious: a charging creature still
  prefers a route through cover when one is available at comparable cost. The four weights
  balance how strongly cover attracts against how far the route may deviate for it.
- **Short route, not fast route.** Explicitly asking *not* to minimise time means the creature
  takes the geometrically shorter path even when a longer one would be quicker at its speeds. A
  rebuild that flips this gets monsters taking wide fast arcs, which reads as evasion.
- **The squad may dictate the arrival facing.** When a squad has issued an attack command with a
  direction, the creature is told to end its route facing that way — which is how a pack
  surrounds a target instead of all arriving from the same side. Without a squad or without a
  command, the facing is left to the builder.

**Notes** — the rebuild interval is the creature's own authored number, so a slow, heavy creature
replans rarely (and can be dodged) while a fast one replans constantly. It is one of the most
behaviourally visible numbers a creature's configuration carries.

The four cover weights are compiled in and shared by every creature. Nothing records their
derivation.

## `check_start_conditions` / `check_completion`

**Contract** — start when the enemy is further than the melee checker's maximum distance;
complete when it is nearer than the checker's minimum.

**Invariants** — the two thresholds come from the same checker that gates melee, and the gap
between them is the hysteresis band. Because completion uses the *minimum* and the melee start
uses the *maximum*, the approach hands over to melee well before it would otherwise report
completion — the parent chain in
[`monster_state_attack_inline.h`](monster_state_attack_inline.h.md) tries melee first. The
completion test therefore only fires when melee has refused (typically because the enemy is not
visible), and it is what stops the creature from running into its target.
