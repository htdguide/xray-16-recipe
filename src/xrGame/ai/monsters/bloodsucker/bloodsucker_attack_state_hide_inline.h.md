# src/xrGame/ai/monsters/bloodsucker/bloodsucker_attack_state_hide_inline.h

> Breaking contact mid-fight: cloak, claim one covered spot so no packmate takes it, sprint there, and switch to stalking once arrived.

**Needs** — [`bloodsucker_attack_state_hide.h`](bloodsucker_attack_state_hide.h.md) · [`state_move_to_point.h`](../states/state_move_to_point.h.md) · [`bloodsucker_predator_lite.h`](bloodsucker_predator_lite.h.md) · [`cover_point.h`](../../../cover_point.h.md) · [`monster_cover_manager.h`](../monster_cover_manager.h.md) · [`monster_home.h`](../monster_home.h.md) · [`ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`Actor.h`](../../../Actor.h.md)
**Used by** — [`bloodsucker_attack_state_hide.h`](bloodsucker_attack_state_hide.h.md)
**Tier floor** — T3: cover queries and path parameters over the shared state contract

## Purpose

A two-node composite: go to cover, then camp there as a predator. It is the "vanish" half of the bloodsucker's intended hit-and-fade combat rhythm — the creature turns partly invisible for the whole of it, which is why the cloak is started here and not inside either child.

Unreachable in the shipped build; see the declaration twin.

## State

```text
RECORD BloodsuckerAttackHide
  reserved_vertex : optional<int>   # a navigation vertex claimed from the pack

SUBSTATES
  hide_in_cover : generic "move to point with path options"
  camp_in_cover : the lightweight predator loop
```

**Invariant** — the reserved vertex is released on *every* exit path, ordinary and forced. A vertex left claimed is a spot no packmate will ever use again for the life of the level, so the release is not bookkeeping, it is the thing that keeps a pack from starving itself of cover.

## `BloodsuckerAttackHideState`

**Contract** — on entry, forget any previous reservation and cloak. Selection is positional, not conditional: the first substate of an activation is always the run to cover, and every subsequent one is the camp. Completion is delegated to the camp child and is false while running. Both exits release the reservation. The forced-restart hook does nothing here.

```text
FUNCTION reselect_state()
  IF nothing has run yet   -> hide_in_cover
  ELSE                     -> camp_in_cover
```

## `select_cover_point`

**Contract** — pick the vertex to withdraw to, releasing any vertex previously held and claiming the new one from the pack. Never fails: the fallback is the creature's current vertex.

```text
FUNCTION select_cover_point()
  release any vertex we already hold

  chosen = none
  IF the creature has a home region
    chosen = home.covered_place()        # a covered spot inside the home
    IF chosen is none
      chosen = home.any_place()          # any spot inside the home

  IF chosen is none
    point = cover_system.find_cover(around = my position, between 10 and 30 units)
    IF point EXISTS  chosen = point.vertex

  IF chosen is none
    chosen = my current vertex           # withdraw to where I stand

  claim chosen from the pack
```

**Notes** — the order encodes a priority a rebuild must keep: an authored home region outranks the generic cover query, because a creature with a home is meant to fight inside it and lead the player there. The cover query is only the fallback for creatures placed without one.

The claim/release pair is the pack's only mutual exclusion over terrain. It is advisory — nothing physically stops two creatures standing on one vertex — but it is what spreads a pack across several cover spots instead of stacking them on the single best one.

## `setup_substates`

**Contract** — when the run-to-cover substate is chosen, choose the point first, then fill the movement record: run to that exact vertex, no timeout, arrive exactly, never rebuild the route, aggressive acceleration with braking, and idle vocalisation at the creature's configured delay. The camp substate carries its own parameters and is not filled here.

**Notes** — "never rebuild the route" is deliberate and is the opposite of the pursuit states: the destination is a *place*, not a target, so recomputing the route as the world moves would only cost time. Braking is on for the same reason — the creature is meant to stop on the spot, not slide past it.

This file opens with its include guard written twice. Incidental.
