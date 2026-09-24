# src/xrGame/agent_location_manager.h

> Declares the squad's shared danger map and cover arbiter; behaviour is in [`agent_location_manager.cpp`](agent_location_manager.cpp.md).

**Needs** — [`danger_location.h`](danger_location.h.md) · [`agent_location_manager_inline.h`](agent_location_manager_inline.h.md)
**Used by** — [`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md) · [`agent_location_manager.cpp`](agent_location_manager.cpp.md) · [`agent_location_manager_inline.h`](agent_location_manager_inline.h.md) · [`agent_manager.cpp`](agent_manager.cpp.md) · [`agent_manager_actions.cpp`](agent_manager_actions.cpp.md) · [`ai_stalker_misc.cpp`](ai/stalker/ai_stalker_misc.cpp.md) · [`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md) · [`stalker_danger_in_direction_actions.cpp`](stalker_danger_in_direction_actions.cpp.md) · [`stalker_danger_unknown_actions.cpp`](stalker_danger_unknown_actions.cpp.md) · [`stalker_movement_restriction.h`](stalker_movement_restriction.h.md) · [`stalker_movement_restriction_inline.h`](stalker_movement_restriction_inline.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares one of the squad brain's four managers: the list of active danger locations, each
reference-counted because squad members hold them too. Substance is in
[`agent_location_manager.cpp`](agent_location_manager.cpp.md).

Exported units:

- `add` — register a danger, merging it with any danger already at that position.
- `danger` — score a cover point's safety for a member, in the unit interval.
- `suitable` — may this member claim this cover point without crowding a comrade.
- `make_suitable` — commit the claim and evict whoever it displaces.
- `update` — expire stale dangers and notify the members that held them.
- `location` by position or by causing object, `locations`, `clear`, `remove_links` —
  lookup, read, reset, and teardown for a departing object.

**Notes** — dangers are held through shared, counted references rather than by value
because the same danger record is handed to squad members, who may outlive the manager's
own interest in it. In a rebuild with a tracing collector this distinction vanishes; in one
without, the same sharing has to exist in some form.
