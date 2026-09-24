# src/xrGame/ai/monsters/group_states/group_state_hear_danger_sound_inline.h

> A pack that hears gunfire puts its home region between itself and the sound — one member leads
> and the rest gather round it.

**Needs** — [`group_state_hear_danger_sound.h`](group_state_hear_danger_sound.h.md) · [`../states/state_move_to_point.h`](../states/state_move_to_point.h.md) · [`../states/monster_state_home_point_danger.h`](../states/monster_state_home_point_danger.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../state.h`](../state.h.md) · [Level graph — chapter 14](../../../../xrAICore/README.md)
**Used by** — [`group_state_hear_danger_sound.h`](group_state_hear_danger_sound.h.md)
**Tier floor** — T3: a squad-role decision plus a bounded random-point search on the navigation mesh

## Purpose

The pack version of "I heard something frightening". Where a solitary creature simply hides, a
pack has to avoid the failure mode of every member independently choosing its own hiding place and
the group dissolving. The answer here is a *role split*: exactly one member decides where to go,
and every other member goes to wherever that member is.

## State

```text
RECORD GroupHearDangerState
  target_vertex : int    # the hiding place the leader chose
```

Two authored constants, both in code: followers gather within 20 world units of the leader, and the
navigation mesh is sampled at most 5 times looking for a spot in that band.

## `reselect_state`

**Contract** — choose one of three substates. Requires the creature to be in a squad — there is no
solitary path through this state.

```text
FUNCTION reselect_state()
  IF the go-home substate will accept        # we are outside our territory
    select go_home; RETURN

  IF the squad is active AND my command is "rest"
    IF I am not the leader   -> follow the leader
    ELSE                     -> take cover
  ELSE
    # no resting formation exists: make one, and lead it
    squad.set_leader(me)
    squad.publish_goal(rest)
    select take_cover
    squad.recompute_commands()
```

**Notes** — the *else* branch is the one that matters. A pack whose squad has no standing
formation does not fall back to individual behaviour; the creature that heard the sound seizes
leadership, declares a resting formation, and the squad recomputes everyone's orders — so on the
next update the other members find themselves with a *rest* command and a leader, and take the
follower branch. The role split bootstraps itself from whichever member hears first.

Going home wins over both roles. A creature outside its territory returns to it regardless of what
the pack is doing, which is the same territorial anchoring the attack brain shows.

## `setup_substates`

**Contract** — fill in the parameter record for whichever movement substate just became active.

**The follower's destination** is a point near the leader, chosen in two tiers:

```text
  ask the path builder for a navigation vertex between 8 and 20 units from
     the leader's vertex, sampling at most 5 times
  IF that succeeds -> that vertex and its position
  ELSE
     pick a uniformly random position within 20 units of the leader
     IF the creature's movement restrictions forbid it
        -> the nearest permitted position instead
     ELSE -> that position, with no vertex constraint
```

then run to it, calmly accelerated without braking, stopping within 3 units, with the idle sound.

**Notes** — the inner bound of 8 units is what stops followers piling onto the leader; the sampling
budget of 5 is what stops the search being unbounded on a crowded mesh. The fallback tier accepts a
*position without a vertex*, which lets the path builder solve the last step itself rather than
failing, and consults the creature's movement restrictions so that a pack confined to a region by
the level data does not scatter out of it.

**The leader's destination** is the pack's own territory, on the far side from the sound:

```text
  direction = normalize(home_point - sound_position)
  target_vertex = home.a_place_in_the_outer_region_toward(direction)
  IF none exists -> stay where we are
  run to it, aggressively accelerated with braking, stopping within 1 unit,
     rebuilding the path never, and playing no sound at all
```

**Notes** — the leader's choice is the whole behaviour in one line: take the vector *from the
sound toward home* and ask the home region for a spot on that side. The pack ends up with its
territory between itself and whatever made the noise, which is why shooting at a pack from one
direction moves it consistently away rather than scattering it.

The leader's movement is explicitly **silent** — the sound type is the dummy value — while
followers play the idle sound. A pack retreating from gunfire does not bark, and the creature that
decided to retreat is the quietest of them. That is one enumerated value carrying a piece of
characterisation.

The leader accelerates aggressively *with* braking and stops within 1 unit, against the follower's
calm, brakeless, 3-unit stop. The leader arrives precisely at the chosen cover; the followers
arrive roughly, around it. Both halves of that contrast are authored here and neither is in data.
