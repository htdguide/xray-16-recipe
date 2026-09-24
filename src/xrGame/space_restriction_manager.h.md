# src/xrGame/space_restriction_manager.h

> Declares the per-entity restriction layer on top of the level's restrictor registry.

**Needs** — [`space_restriction_holder.h`](space_restriction_holder.h.md) · [`space_restriction.h`](space_restriction.h.md) · [`space_restriction_manager_inline.h`](space_restriction_manager_inline.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md)
**Used by** — [`Level.cpp`](Level.cpp.md) · [`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md) · [`Level_network.cpp`](Level_network.cpp.md) · [`anomaly_detector.cpp`](ai/monsters/anomaly_detector.cpp.md) · [`patrol_path_manager.cpp`](patrol_path_manager.cpp.md) · [`restricted_object.cpp`](restricted_object.cpp.md) · [`script_game_object_inventory_owner.cpp`](script_game_object_inventory_owner.cpp.md) · [`space_restriction.cpp`](space_restriction.cpp.md) · [`space_restriction_manager.cpp`](space_restriction_manager.cpp.md) · [`space_restriction_manager_inline.h`](space_restriction_manager_inline.h.md) · [`space_restrictor.cpp`](space_restrictor.cpp.md) · [`stalker_anomaly_actions.cpp`](stalker_anomaly_actions.cpp.md)
**Tier floor** — T2: a declaration over two keyed maps

## Purpose

Declares the surface implemented in [`space_restriction_manager.cpp`](space_restriction_manager.cpp.md)
and [`space_restriction_manager_inline.h`](space_restriction_manager_inline.h.md). It
extends the registry rather than holding one, so a single object serves both roles: the
level owns one of these and it answers both "what volume is this restrictor" and "where may
this entity go".

## Exported units

- `restrict`, `unrestrict` — bind and unbind an entity.
- `add_restrictions`, `remove_restrictions`, `change_restrictions` — edit an entity's authored lists.
- `accessible(sphere)`, `accessible(vertex, radius)`, `accessible_nearest` — the per-entity queries.
- `add_border`, `remove_border` — stamp and clear an entity's border on the level graph.
- `in_restrictions`, `out_restrictions` — the merged lists actually in force.
- `base_in_restrictions`, `base_out_restrictions` — the lists as authored or scripted.
- `restriction_presented` — is this name in this list.
- `clear` — level teardown.
- `on_default_restrictions_changed` — the registry's notification, implemented here as a full re-derivation.

## Notes

The per-entity record pairs the resolved shared restriction with the entity's *base* lists;
keeping those separate from the merged result is the decision the whole edit path rests on,
and it is explained in the implementation twin.

Entities are named by the 16-bit entity identifier, so the binding map is keyed the same way
the save file and the network protocol are.
