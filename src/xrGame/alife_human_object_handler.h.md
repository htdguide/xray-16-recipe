# src/xrGame/alife_human_object_handler.h

> Declares the offline inventory manager for a human server record.

**Needs** — [`alife_human_object_handler.cpp`](alife_human_object_handler.cpp.md) · [`alife_human_object_handler_inline.h`](alife_human_object_handler_inline.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md)
**Used by** — [`alife_human_abstract.cpp`](alife_human_abstract.cpp.md) · [`alife_human_object_handler.cpp`](alife_human_object_handler.cpp.md) · [`alife_human_object_handler_inline.h`](alife_human_object_handler_inline.h.md) · [`ef_primary.cpp`](ef_primary.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface stubbed in
[`alife_human_object_handler.cpp`](alife_human_object_handler.cpp.md) and implemented for
real in the preserved
[`alife_human_object_handler_save.h`](alife_human_object_handler_save.h.md). Bound to one
human server record for its lifetime.

Three groups:

- **Ammunition** — `get_available_ammo_count` (by object list, or by item list with an
  optional object list), `attach_available_ammo`, `collect_ammo_boxes`.
- **Inventory upkeep** — `detach_all`, `update_weapon_ammo`, `process_items`,
  `attach_items`, `best_detector`, `best_weapon`, `can_take_item`.
- **Equipment choice** — `choose_equipment`, `choose_weapon` by priority category,
  `choose_food`, `choose_medikit`, `choose_detector`, `choose_valuables`, `choose_fast`,
  `choose_group`.

**Notes** — Almost every method takes an optional *object list* it may draw from in addition
to the character's own inventory. That parameter is what lets the same routines serve three
situations: equipping from what you carry, from a corpse, and from a pile being divided
between two parties. A rebuild should keep the parameter rather than write three variants.
