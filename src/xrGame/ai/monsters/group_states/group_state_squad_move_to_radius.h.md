# src/xrGame/ai/monsters/group_states/group_state_squad_move_to_radius.h

> Declares the two "close to a ring around the enemy" states, implemented in
> [`group_state_squad_move_to_radius_inline.h`](group_state_squad_move_to_radius_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`../states/state_data.h`](../states/state_data.h.md) · [`group_state_squad_move_to_radius_inline.h`](group_state_squad_move_to_radius_inline.h.md)
**Used by** — [`group_state_attack_inline.h`](group_state_attack_inline.h.md) · [`group_state_squad_move_to_radius_inline.h`](group_state_squad_move_to_radius_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the two rungs of the pack stalking ladder. Both approach a ring at a given radius around the
enemy; they differ in *where on the ring* they aim, which is the only thing that separates them.

Each owns the parameter record its caller fills in — the composite above writes the radius, the
gait, the sounds and the acceleration into it at the moment the state is entered (see
[`group_state_attack_inline.h`](group_state_attack_inline.h.md)) — so the two classes are
parameterised leaf states rather than fixed behaviours. Split from their bodies only because C++
splits templates that way.

## `CStateGroupSquadMoveToRadiusEx`

The **fanned** variant: each squad member is assigned an arc of the ring from its index within the
squad, so the pack spreads around the enemy.

- **enter** — prime the path builder
- **execute** — compute this member's point on the ring and drive toward it
- **is_finished** — three ways: a timeout, close enough to the enemy, or arrived at the point
- **remove_links** — forward the destruction notice

## `CStateGroupSquadMoveToRadius`

The **radial** variant: aim at the point on the ring directly between the creature and the enemy,
with no squad awareness at all.

Same four members, with a simpler point computation and a two-way completion test. Contracts are
in the implementation twin.
