# src/xrGame/smart_cover_detail.h

> Declares the strict readers that pull a smart cover's authored description out of a script table, and the two reserved loophole names that stand for entering and leaving.

**Needs** — [`restriction_space.h`](../xrServerEntities/restriction_space.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`smart_cover_action.cpp`](smart_cover_action.cpp.md) · [`smart_cover_action.h`](smart_cover_action.h.md) · [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) · [`smart_cover_animation_selector.h`](smart_cover_animation_selector.h.md) · [`smart_cover_default_behaviour_planner.hpp`](smart_cover_default_behaviour_planner.hpp.md) · [`smart_cover_description.cpp`](smart_cover_description.cpp.md) · [`smart_cover_description.h`](smart_cover_description.h.md) · [`smart_cover_detail.cpp`](smart_cover_detail.cpp.md) · [`smart_cover_loophole.cpp`](smart_cover_loophole.cpp.md) · [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`smart_cover_loophole_planner_actions.h`](smart_cover_loophole_planner_actions.h.md) · [`smart_cover_planner_actions.h`](smart_cover_planner_actions.h.md) · [`smart_cover_planner_target_provider.h`](smart_cover_planner_target_provider.h.md) · [`smart_cover_planner_target_selector.h`](smart_cover_planner_target_selector.h.md) · _and 1 more_
**Tier floor** — T2: typed extraction from a dynamically typed table

## Purpose

Declares the surface implemented in
[`smart_cover_detail.cpp`](smart_cover_detail.cpp.md). Everything a smart cover is — its
loopholes, their fields of view, the actions available at each, the transition graph — is
authored as nested script tables rather than in [ltx](../../GLOSSARY.md). This file names
the small vocabulary every part of the smart-cover loader uses to read those tables, so
the readers are written once and behave identically everywhere.

## Exported units

- **`parse_float`** — two forms: one that demands the field, one that reports whether it
  was present. Both range-check.
- **`parse_string`, `parse_bool`, `parse_int`, `parse_table`** — demanding readers.
- **`parse_fvector`** — two forms, demanding and optional.
- **`transform_vertex`** — maps an empty loophole name onto one of the two reserved
  pseudo-loopholes.
- **`parse_vertex`** — read a loophole name from a table and pass it through
  `transform_vertex`.
- **`intrusive_base_time`** — the reference-counted, age-stamped base a cover description
  is held by, so unused descriptions can be evicted; defined in
  [`restriction_space.h`](../xrServerEntities/restriction_space.h.md).
