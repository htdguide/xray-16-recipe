# src/xrGame/ai/monsters/group_states/group_state_rest_idle_inline.h

> Wander the territory: walk to a spot nobody else has claimed, do something idle there, walk
> somewhere else — and sniff the ground on some of the short walks.

**Needs** — [`group_state_rest_idle.h`](group_state_rest_idle.h.md) · [`../states/state_move_to_point.h`](../states/state_move_to_point.h.md) · [`../states/state_look_point.h`](../states/state_look_point.h.md) · [`../states/state_custom_action.h`](../states/state_custom_action.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../monster_cover_manager.h`](../monster_cover_manager.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`group_state_rest_idle.h`](group_state_rest_idle.h.md)
**Tier floor** — T3: a three-beat loop, a squad reservation, and one probabilistic gait rule

## Purpose

The innermost loop of creature life, and the one the player watches for hours. Three decisions are
worth naming before the details:

**Where** — a spot in the territory, chosen from the *inner* ring if the creature has just finished
eating and from the *middle* ring otherwise, and **reserved with the squad** so pack members do not
stack.

**What to do there** — a numbered flavour animation, chosen by the creature's own weighted picker
(see [`../dog/dog.cpp`](../dog/dog.cpp.md)) — unless the creature has just eaten, in which case it
sits down.

**How to walk** — an ordinary walk for long moves, and for short moves a run of *sniffing* walks
governed by a small counter, which is what makes a creature look like it is following a scent
rather than commuting.

## State

```text
RECORD GroupRestIdleState
  target_vertex : int   # the reserved destination
  move_type     : int   # 0 = sniffing gait, 1 = ordinary walk; chosen per walk
```

The sniffing counter itself lives on the *creature*, not here, because it must survive across
activations of this state.

## `initialize`

**Contract** — choose a destination in two tiers and reserve it with the squad.

```text
FUNCTION initialize()
  IF we have just finished eating
    target = home.a_place_in_the_inner_region()
    IF none -> cover_system.find_cover(around home_point, 1 .. home.inner_radius)
  ELSE
    target = home.a_place_in_the_middle_region()
    IF none -> cover_system.find_cover(around home_point, 1 .. home.middle_radius)

  IF still none -> return with no destination at all
  squad.reserve_cover(target)
```

**Notes** — the just-eaten flag routes the creature to the *inner* ring, closer to the home point.
A pack that has fed gathers tighter; a pack that has not spreads out over the middle ring. That
one flag is set by the feeding composite on its way out (see
[`group_state_eat_inline.h`](group_state_eat_inline.h.md)) and is the only state that crosses
between the two behaviours.

Returning with no destination leaves the target absent, which the parameter setup below handles by
substituting the creature's current cell — so the creature idles where it stands rather than
failing.

## `finalize` / `critical_finalize`

**Contract** — both release the squad's reservation. Identical bodies; a reservation must be
released on either exit or the spot is lost to the pack for the rest of the level.

## `reselect_state`

**Contract** — the three-beat loop, with one resume path.

```text
FUNCTION reselect_state()
  IF we were told to resume looking at the open ground
    clear that note; -> walk to a wander point
    RETURN

  IF nothing has run yet and we have a destination, OR we just walked to a wander point
    -> walk to the reserved spot

  IF we just walked to the reserved spot, or nothing has run yet
    note that we should look at the open ground next
    IF we have just finished eating
      request clip 8 (sit down); clear the just-eaten flag
    ELSE
      request the creature's weighted random idle clip
    -> play the flavour animation
    RETURN

  -> walk to a wander point
```

**Notes** — the "look at the open ground" note is written *before* the flavour animation runs and
consumed on the next pass, because the animation's completion re-enters the brain from the top
(see [`../dog/dog.cpp`](../dog/dog.cpp.md)) and the loop has to be resumable from outside. That is
the same idiom the feeding sequence uses for resuming a meal.

The just-eaten flag is consumed here, not in the entry, and it selects *sitting down* rather than a
random idle. A creature that has eaten walks to the inner ring and sits — which is the behaviour
the player reads as a fed animal settling.

## `setup_substates`

**The two walks** share a body and differ only in destination: the wander walk aims at a spot in
the middle ring, the cover walk at the reserved spot; both substitute the creature's current cell
when their destination is absent. Both stop *exactly* on the point (completion distance zero),
never rebuild the path, and accelerate calmly with braking.

The gait rule is the interesting part and is identical in both:

```text
IF distance to the destination > 8
  move_type = ordinary walk
  reset the creature's sniffing counter          # long walks are never sniffing walks
ELSE
  IF the sniffing counter is absent, or has exceeded 4 + the creature's sniffing allowance
    move_type = random(0 or 1)
    IF sniffing was chosen  -> start the counter at 1
    ELSE                    -> clear it
    draw a fresh sniffing allowance in 0..2
  ELSE
    move_type = sniffing
    increment the counter

action = move_type ? walk_forward : walk_while_smelling
```

**Notes** — this is a *run-length* rule, not a per-walk coin flip, and that distinction is what
makes it read as behaviour. Once a creature starts sniffing it keeps sniffing for between five and
seven consecutive short walks (four plus an allowance of nought to two, counted from one), then
re-rolls. A per-walk coin flip would produce a creature alternating gaits every few metres, which
reads as indecision; a run produces a creature that is *following something* and then stops.

The eight-unit threshold is what confines sniffing to short moves. A creature crossing its
territory walks normally; a creature pottering about within a few metres sniffs. Both the threshold
and the allowance range are authored in code.

**The looking rung** asks the cover system for the least-covered direction from where the creature
stands, looks at a point 10 units along it for 1 second, with the idle sound. It is the same idiom
[`group_state_home_point_attack_inline.h`](group_state_home_point_attack_inline.h.md) uses with a
longer duration and a different sound — looking at the *most exposed* direction, which reads as
watchfulness.

**The flavour-animation rung** stands idle with no timeout — the clip's own length ends it — and
chooses the threat sound when the clip is the growl and the idle sound otherwise. That is a
narrower version of the sound lookup in
[`group_state_custom_inline.h`](group_state_custom_inline.h.md); the two disagree about the howl,
which this rung does not special-case.
