# src/xrGame/stalker_movement_manager_smart_cover_inline.h

> The field accessors of the smart-cover movement layer, split out of the header
> for compilation reasons only.

**Needs** — [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: field reads and writes.

## Purpose

Substance lives in [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md)
and its three implementation files. This file exists because the original language wants
small functions visible to every caller at compile time; a rebuild has no reason to keep
the split.

## Accessors

- `animation_selector` — the in-cover animation planner; must be present when asked for.
- `property_storage(storage)` — installs the planner world-state this layer writes into.
- `current_transition_animation` — the animation of the transition being traversed; must
  be present when asked for.
- `non_animated_loophole_change(value)` — private setter for the walked-instead-of-played
  loophole change mode.
- `apply_loophole_direction_distance` (get and set) — the distance at which a creature
  approaching a loophole stops looking along its path and starts looking along the
  loophole's authored direction, so that it arrives already aimed.
- `target_selector` — the object deciding what the creature aims at inside a cover; must
  be present when asked for.
- `entering_smart_cover_with_animation` — true between the start of an enter animation and
  the moment the cover becomes current.
- `check_can_kill_enemy` (get and set) — whether the in-cover behaviour is allowed to
  consider firing.
- `combat_behaviour` (get and set) — whether the cover is being used in combat, which
  selects a different set of in-cover animations from idle occupancy.
