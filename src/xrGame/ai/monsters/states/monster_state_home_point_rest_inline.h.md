# src/xrGame/ai/monsters/states/monster_state_home_point_rest_inline.h

> Drifted out of the middle of your territory while idle — go back, walking if the territory is
> placid and running if it is aggressive.

**Needs** — [`monster_state_home_point_rest.h`](monster_state_home_point_rest.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`monster_state_home_point_rest.h`](monster_state_home_point_rest.h.md)
**Tier floor** — T3: one destination and a two-way gait switch

## Purpose

The reason a lair stays occupied. A resting creature wanders — following graph points, circling a
leader, going about a smart-terrain job — and this leaf is what pulls it back before the wandering
takes it out of its territory entirely. Because it is tested from inside the resting behaviour and
outranks the ordinary idling, an animal's whole peacetime life is bounded by it.

## State

```text
RECORD RestHomeState
  target_vertex : int   # a cell in the territory's inner region, chosen once on entry
```

## `check_start_conditions`

**Contract** — startable when the creature's position is outside the territory's **inner** region.

**Notes** — the *middle* extent, not the outer one. A home region carries three authored radii —
inner, middle and outer — and the gap between middle and outer is the hysteresis that makes this
behaviour stable: the creature is pulled back when it drifts past the middle radius, while
everything else that asks "is this creature at home" asks about the outer one. With a single
radius the creature would re-trigger at the boundary forever. The radii are authored per home
region, not per creature; see [`../monster_home.h`](../monster_home.h.md).

## `initialize`

**Contract** — ask the home component for a cell in the inner region; that is the destination. The
movement base also prepares the path builder.

**Notes** — chosen once, never revised, and never claimed against the squad — unlike every danger
retreat in this directory. Two resting packmates may therefore converge on the same cell, which is
acceptable for idle wandering and is not for cover under fire.

## `execute`

**Contract** — hand the destination to the path builder, set a gait and an acceleration profile
from the territory's temperament, disable cover-biased routing, re-plan the route continuously,
and require an exact arrival.

```text
FUNCTION execute()
  path.target         = (navigation.position_of(target_vertex), target_vertex)
  aggressive          = home.is_aggressive
  acceleration        = aggressive ? aggressive_profile : calm_profile, braking on arrival
  path.rebuild_every  = 0                  # every opportunity
  path.distance_to_end = 0                 # exactly onto the cell
  path.use_covers     = false
  action              = aggressive ? run : walk_forward
  voice               = aggressive ? aggressive_voice : idle_voice
```

**Notes** — the temperament flag is an **authored property of the territory**, not of the creature,
and it switches three things at once: the gait, the acceleration profile and the voice. So the same
species reads as placid wildlife in one authored lair and as an alert, patrolling threat in another,
with no change to the creature's own configuration section. That is the most economical piece of
authoring leverage in this directory, and a rebuild that attaches the flag to the creature instead
of to the territory loses it.

*Cover-biased routing is explicitly disabled.* A creature going home while nothing is threatening
it takes the direct route. Every danger retreat in this directory enables covers; this is the one
that turns them off, and the contrast is the point.

*Braking is enabled*, again unlike the danger retreats: the creature slows into its destination
rather than skidding through it, because it is about to stand there.

Arrival tolerance is zero and the route is re-planned continuously — the opposite settings from the
danger retreats, which commit to one route and accept a metre of slack. Nothing is urgent here, so
precision is affordable.

## `check_completion`

**Contract** — finished when the creature is standing on the destination cell and the path builder
reports no movement in progress.
