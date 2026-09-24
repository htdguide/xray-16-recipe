# src/xrGame/ai/monsters/states/monster_state_squad_rest_follow_inline.h

> Following the pack: walk to the spot the squad order names, and pause there for a couple of
> seconds whenever you are within a randomly-chosen slack of it.

**Needs** — [`monster_state_squad_rest_follow.h`](monster_state_squad_rest_follow.h.md) · [`state_custom_action.h`](state_custom_action.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md) · [`state_data.h`](state_data.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../../../restricted_object.h`](../../../restricted_object.h.md)
**Used by** — [`monster_state_squad_rest_follow.h`](monster_state_squad_rest_follow.h.md)
**Tier floor** — T3: one randomized threshold and a two-leaf alternation

## Purpose

Selected by the peacetime cascade when the squad leader's standing order is *follow*. Each squad
member is given its own commanded position by the squad — a formation slot, not the leader's own
position — and this behaviour is what drives the member to it.

It exists separately from the loitering behaviour because a *following* pack must keep moving with
its leader while a *resting* one must stay put, and the difference is not one of speed but of what
the destination is: here the destination is re-read from the squad every time a leaf is chosen, so
the member tracks a slot that moves.

## State

```text
RECORD SquadFollowState
  last_point : vector   # DEAD: written on entry, never read
```

## `reselect_state` — the randomized slack

**Contract** — read the commanded position from the squad. If the creature is closer to it than a
threshold drawn fresh from the range two to ten units, pause; otherwise walk to it.

```text
FUNCTION next_leaf() -> leaf
  commanded = squad.order_for(self).position
  slack     = random_real(2, 10)                  # redrawn on every reselection
  IF distance(self, commanded) < slack  RETURN pause
  RETURN walk_to_commanded
```

**Notes** — the randomized threshold is the whole trick and it is worth stating precisely: the
stopping distance is not a constant with noise added to the destination, it is a **threshold
redrawn on every decision**. The consequences are exactly what a formation wants.

A member that is eleven units out always walks. A member that is three units out always pauses. A
member anywhere in between pauses or walks depending on the draw, and redraws every time a leaf
ends — so it drifts in and out, tightening the formation without ever locking to a fixed radius. A
whole pack following a leader therefore breathes rather than marching in a rigid pattern.

The lower bound is the same two units the walk leaf uses as its arrival tolerance, so a member that
has actually arrived always pauses. The upper bound is five times that, and it is where the
formation's looseness comes from.

## `check_force_state`

**Contract** — overridden to do nothing.

**Notes** — the override is deliberate and not vestigial: the base's pre-emption hook would
otherwise be inherited, and this behaviour explicitly wants no mid-leaf interruption. A member that
has started walking to its slot finishes the walk even if the slot has moved; it picks up the new
slot on the next reselection. Re-targeting mid-walk would make following packs jitter.

## `setup_substates`

**Contract** — fill the parameters of whichever leaf was selected. The walk's destination is
re-read from the squad here, not taken from the entry snapshot.

```text
pause:
  action   = rest
  time_out = random_int(2000, 3000) ms
  voice    = idle, delay = section key "idle_sound_delay"

walk_to_commanded:
  destination     = squad.order_for(self).position,
                    replaced by the nearest permitted position if outside the restrictors
  gait            = walk forward, accelerating, no braking, calm profile
  completion_dist = 2
  rebuild         = never
  voice           = idle, delay = section key "idle_sound_delay"
```

**Notes** — the restrictor check on the destination is the same inline guard the loitering
behaviour uses, for the same reason: the commanded position was computed by the squad from
formation geometry and may land outside the member's permitted volumes. Replacing it with the
nearest permitted position keeps the member legal without failing the order.

*Re-planning is switched off*, so the member commits to the route it plans when the leaf starts.
Combined with the empty pre-emption hook, that means one walk leaf equals one committed route,
and formation updates take effect between legs. That is what keeps a following pack from
recomputing routes every frame as the leader moves.

The two-to-three-second pause is short — shorter than the loitering behaviour's five to ten —
because a following member must be ready to move again quickly.

`idle_sound_delay` is the only authored number.
