# src/xrGame/ai/monsters/states/monster_state_rest_fun_inline.h

> An idle creature knocking a corpse around for eight seconds. **Dead behaviour** — implemented,
> registered, and selected by nothing.

**Needs** — [`monster_state_rest_fun.h`](monster_state_rest_fun.h.md) · [`../monster_corpse_manager.h`](../monster_corpse_manager.h.md) · [`../monster_home.h`](../monster_home.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`monster_state_rest_fun.h`](monster_state_rest_fun.h.md)
**Tier floor** — T2: computes and applies impulses to every element of a rigid-body assembly

## Purpose

Written to give idle predators something to do near a kill: run at the body, hit it, watch it
tumble, hit it again. Nothing in the shipped source ever selects it — see
[`monster_state_rest_inline.h`](monster_state_rest_inline.h.md), which registers it under the
"playing" identifier and never names that identifier again.

It is described here in full because it is the only place in this directory that pushes on the
physics world rather than reading from it, and because a rebuild that wanted the behaviour back
needs the numbers.

## State

```text
RECORD RestFunState
  time_last_hit : int (milliseconds)   # last swipe, for the hundred-millisecond floor
```

## `check_start_conditions`

**Contract** — startable when the corpse memory names a corpse and that corpse lies inside the
creature's home region.

**Notes** — the same territorial gate the feeding behaviour uses. A creature does not cross the
level to play with a body.

## `execute`

**Contract** — run to a point two units *beyond* the corpse along the line from the creature
through it, re-planning the route on a delay proportional to the remaining distance, without cover
bias. When within the authored corpse distance plus half a unit, and at least a tenth of a second
since the last swipe, apply an impulse to every element of the corpse's rigid-body assembly along a
direction that is the sum of the creature's facing and the direction to the body, tilted five
degrees upward.

```text
FUNCTION execute()
  to_corpse = corpse.position - self.position
  distance  = length(to_corpse)
  target    = corpse.position + normalize(to_corpse) * 2

  action             = run
  path.target        = target
  path.rebuild_every = 100 + 50 * distance     # milliseconds
  path.use_covers    = false
  path.distance_to_end = 0.5
  acceleration       = calm, no braking
  voice              = idle

  IF distance < section["distance_to_corpse"] + 0.5
     AND time_last_hit + 100 < now()
    IF corpse has a rigid-body assembly
      # a blow that both carries the creature's momentum and lifts
      dir     = (corpse.position - self.position) + self.facing
      heading, pitch = decompose(dir)
      dir     = normalize(recompose(heading, pitch + 5 degrees))
      share   = assembly.total_mass / assembly.element_count
      FOR EACH element IN assembly
        element.apply_impulse(dir, 15 * share)
      time_last_hit = now()
```

**Notes** — four decisions are worth keeping.

*The run target is two units past the corpse, not at it.* The creature charges through rather than
stopping, which is what turns a nudge into a swipe. It is the same overshoot idea the lost-contact
charge uses.

*The re-plan delay grows with distance.* Far away, the route is re-planned rarely — it cannot have
gone far wrong; close in, where the corpse is being knocked about and the target moves every swipe,
it is re-planned every tenth of a second. That is a genuinely good rule and it is expressed in one
line: a hundred milliseconds plus fifty per unit of distance.

*The impulse direction is the sum of two vectors, not one.* Taking only the direction to the corpse
would shove the body directly away; adding the creature's facing biases the blow toward where the
creature is actually heading, so a body hit while the creature runs past it flies sideways. The
five-degree upward tilt is what makes it tumble rather than slide, and it is applied in the
heading/pitch decomposition so it lifts relative to the world, not to the creature.

*The impulse is divided by element count and multiplied by total mass*, then scaled by fifteen. So
the effect is mass-proportional — a heavy body moves as far as a light one — and does not depend on
how finely the body happens to be modelled. Fifteen is the only free parameter and nothing derives
it.

The only authored number is `distance_to_corpse`, reused here from the feeding behaviour as the
reach of a swipe. The tenth-of-a-second floor, the two-unit overshoot, the half-unit arrival
tolerance, the delay coefficients, the five degrees and the fifteen are all compiled in.

## `check_completion`

**Contract** — finished when the corpse memory no longer names a corpse, or eight seconds after
entry.
