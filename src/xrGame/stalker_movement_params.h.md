# src/xrGame/stalker_movement_params.h

> Declares the record that describes one complete "where and how a stalker
> should be moving" state.

**Needs** — [`stalker_movement_params.cpp`](stalker_movement_params.cpp.md) · [`smart_cover.h`](smart_cover.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md) · [`movement_manager_space.h`](movement_manager_space.h.md) · [`detail_path_manager_space.h`](detail_path_manager_space.h.md)
**Used by** — [`stalker_movement_manager_base.cpp`](stalker_movement_manager_base.cpp.md) · [`stalker_movement_manager_base.h`](stalker_movement_manager_base.h.md) · [`stalker_movement_manager_smart_cover.cpp`](stalker_movement_manager_smart_cover.cpp.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`stalker_movement_params.cpp`](stalker_movement_params.cpp.md) · [`stalker_movement_params_inline.h`](stalker_movement_params_inline.h.md)
**Tier floor** — T2: a data record with lazily-derived fields.

## Purpose

Declares the surface implemented in [`stalker_movement_params.cpp`](stalker_movement_params.cpp.md)
and [`stalker_movement_params_inline.h`](stalker_movement_params_inline.h.md). The
movement manager holds two of these records — *current* and *target* — and the whole of
stalker locomotion is the process of making current equal target.

## Exported units

- `construct(manager)` — binds the record to the movement manager it belongs to, which it
  needs in order to ask "what am I taking cover from" when it selects a loophole.
- `equal_to_target(target)` — the has-nothing-changed test that decides whether a re-plan
  is needed.
- the five plain locomotion fields: body state, movement type, mental state, path type and
  detail-path type.
- `desired_position` / `desired_direction` — optional overrides, get and set.
- `cover_id` (get and set), `cover` — the smart cover this state names.
- `cover_loophole_id` (get and set), `cover_loophole` — the aperture within it, which may
  be explicitly named or left to be selected.
- `cover_fire_object` / `cover_fire_position` — what the cover is being taken against,
  get and set, mutually exclusive.
- copy assignment — present because the record contains self-references that a bitwise
  copy would leave dangling; see the `.cpp` twin.
