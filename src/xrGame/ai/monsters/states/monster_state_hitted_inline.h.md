# src/xrGame/ai/monsters/states/monster_state_hitted_inline.h

> Shot by something unseen: break away from where it came from, then creep back toward it, over and
> over — unless there is a home region to retreat to.

**Needs** — [`monster_state_hitted.h`](monster_state_hitted.h.md) · [`monster_state_hitted_hide.h`](monster_state_hitted_hide.h.md) · [`monster_state_hitted_moveout.h`](monster_state_hitted_moveout.h.md) · [`monster_state_home_point_danger.h`](monster_state_home_point_danger.h.md)
**Used by** — [`monster_state_hitted.h`](monster_state_hitted.h.md)
**Tier floor** — T3: a three-way transition table

## Purpose

The behaviour that covers being shot by an enemy the creature has not identified. It is the
reaction a player sees when they open fire on wildlife from long range, and its shape is deliberate:
the creature does not charge (it has no target to charge) and does not simply flee (it would never
close). It oscillates.

## State

`Stateless.`

## `reselect_state`

**Contract** — the go-home leaf wins whenever its start condition holds, re-tested on every
selection. Otherwise alternate: break away first, then stalk back, then break away again.

```text
FUNCTION next_leaf(previous) -> leaf
  IF go_home.can_start()       RETURN go_home      # re-tested every time
  IF previous is none          RETURN break_away
  IF previous == break_away    RETURN stalk_back
  RETURN break_away                                 # from stalk_back or anything else
```

**Notes** — the oscillation is the design. Break away puts distance between the creature and the
hit direction; stalk back closes that distance slowly and in cover. A creature under repeated fire
therefore bobs in and out of concealment along the line of fire, which both keeps it alive longer
under a scoped rifle and eventually delivers it to the shooter — an animal that only fled would
never arrive, and one that only advanced would die in the open.

The two leaves also read each other's exit conditions: the break-away ends on distance, the stalk
ends on either closing to within three units of the hit point *or being hit again*. Being hit
during a stalk therefore flips the creature straight back to breaking away, which is what makes
sustained fire pin a creature in cover instead of drawing it in.

As in the frightening-sound behaviour, the go-home test is first and unconditional, so a
territorial creature hit outside its lair abandons the oscillation and heads home. The same
go-home leaf object serves both behaviours — see
[`monster_state_home_point_danger.h`](monster_state_home_point_danger.h.md) — which is why a
creature's reaction to gunfire and to being shot look the same once it has a territory.

There is no exit from this behaviour of its own: none of the three leaves' completions ends it, and
the reselection is total. It runs until the brain's motivation weighting outranks it, typically by
the creature acquiring an enemy and switching to combat. Acquiring the shooter as an enemy is what
normally ends it, and that happens in the memory components, not here.
