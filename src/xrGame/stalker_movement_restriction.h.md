# src/xrGame/stalker_movement_restriction.h

> Declares the predicate a stalker hands to the cover search so that squadmates
> do not all pick the same cover point.

**Needs** — [`stalker_movement_restriction_inline.h`](stalker_movement_restriction_inline.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_location_manager.h`](agent_location_manager.h.md) · [`ai/stalker/ai_stalker_impl.h`](ai/stalker/ai_stalker_impl.h.md)
**Used by** — [`ai_stalker_cover.cpp`](ai/stalker/ai_stalker_cover.cpp.md) · [`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md) · [`stalker_danger_in_direction_actions.cpp`](stalker_danger_in_direction_actions.cpp.md) · [`stalker_danger_property_evaluators.cpp`](stalker_danger_property_evaluators.cpp.md) · [`stalker_get_distance_actions.cpp`](stalker_get_distance_actions.cpp.md) · [`stalker_movement_restriction_inline.h`](stalker_movement_restriction_inline.h.md) · [`stalker_search_actions.cpp`](stalker_search_actions.cpp.md)
**Tier floor** — T2: a small predicate object over squad-level bookkeeping.

## Purpose

Declares the surface implemented in
[`stalker_movement_restriction_inline.h`](stalker_movement_restriction_inline.h.md).

The cover search takes a caller-supplied filter so that the *same* search code serves
every creature while each creature applies its own squad's occupancy rules. This is that
filter for stalkers: three operations — admit a candidate, weight it, and claim it — bound
to one stalker and its squad's agent manager.

## Exported units

- construction from (stalker, whether to consider enemy information, whether to claim
  points) — the third argument defaults to claiming.
- the admission test, applied to each candidate cover point.
- `weight(point)` — the candidate's danger, used to rank admitted points.
- `finalize(point)` — called on the chosen point, marking it taken.
