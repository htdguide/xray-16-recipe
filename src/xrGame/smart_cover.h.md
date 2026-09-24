# src/xrGame/smart_cover.h

> Declares one smart cover placed on a level: a description, the subset of its loopholes this instance enables, their navigation vertices, and the choice of which loophole to use against a given position.

**Needs** — [`smart_cover_description.h`](smart_cover_description.h.md) · [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`smart_cover_object.h`](smart_cover_object.h.md) · [`cover_point.h`](cover_point.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`ai_stalker_cover.cpp`](ai/stalker/ai_stalker_cover.cpp.md) · [`cover_evaluators.cpp`](cover_evaluators.cpp.md) · [`cover_manager.cpp`](cover_manager.cpp.md) · [`cover_manager.h`](cover_manager.h.md) · [`script_game_object_smart_covers.cpp`](script_game_object_smart_covers.cpp.md) · [`smart_cover.cpp`](smart_cover.cpp.md) · [`smart_cover_animation_selector.cpp`](smart_cover_animation_selector.cpp.md) · [`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md) · [`smart_cover_inline.h`](smart_cover_inline.h.md) · [`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md) · [`smart_cover_object.cpp`](smart_cover_object.cpp.md) · [`smart_cover_object.h`](smart_cover_object.h.md) · [`smart_cover_object_script.cpp`](smart_cover_object_script.cpp.md) · [`smart_cover_planner_actions.cpp`](smart_cover_planner_actions.cpp.md) · _and 9 more_
**Tier floor** — T2: an instance record plus a short search, consulted during planning

## Purpose

Declares the surface implemented in [`smart_cover.cpp`](smart_cover.cpp.md) and
[`smart_cover_inline.h`](smart_cover_inline.h.md).

This is the *placed instance*: one authored description
([`smart_cover_description.h`](smart_cover_description.h.md)) plus a world transform, plus
a per-instance mask of which loopholes are enabled, plus the navigation vertices computed
for this placement. It presents itself to the rest of the AI as an ordinary cover point,
flagged so that code which cares can tell it apart.

## Exported units

- **The class** — a cover point that knows it is a smart cover.
- **`loopholes`** — the enabled subset, in the description's order.
- **`get_object` / `get_description` / `id`** — the placed entity, the shared template, and
  the instance's name.
- **Local-to-world transforms** — `fov_position`, `fov_direction`,
  `danger_fov_direction`, `enter_direction`, `position`: the loophole's authored geometry
  in world space, computed on demand from the placed object's transform.
- **`level_vertex_id`** — the navigation vertex a loophole stands on.
- **`action_level_vertex_id`** — the navigation vertex a *moving* action of a loophole
  takes the creature to.
- **`best_loophole`** — the central query: which loophole to use against a given world
  position.
- **`evaluate_loophole` / `evaluate_loophole_for_default_usage`** — the two scoring rules
  `best_loophole` selects between.
- **Predicates** — `is_position_in_fov`, `is_position_in_danger_fov`,
  `is_position_in_range`, `in_min_acceptable_range`: the arc and distance tests the
  planner's evaluators ask.
- **`is_combat_cover` / `can_fire`** — whether this placement is for fighting from, and
  whether firing is permitted at all.

## State

```text
RECORD cover                          # extends a cover point
  description     : description        # shared, reference-counted
  loopholes       : list<loophole>     # the enabled subset of the description's
  vertices        : list of
      loophole        : loophole
      level_vertex_id : int
      action_vertices : list<(action_name, level_vertex_id)>
  object          : smart cover object  # the placed entity; supplies the transform
  id              : text                # the placed entity's name
  is_combat_cover : bool
  can_fire        : bool
```

**Invariants** — `vertices` is parallel to `loopholes`: one entry per enabled loophole, in
the same order. Every navigation vertex stored in it is valid on the loaded level, which
is checked at construction.

## Notes

`can_fire` as exposed is the *disjunction* of the two flags: a combat cover always permits
firing regardless of its own fire flag. The two are stored separately because
`is_combat_cover` additionally selects which of the two loophole-scoring rules the planner
uses.
