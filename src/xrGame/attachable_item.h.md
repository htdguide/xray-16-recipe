# src/xrGame/attachable_item.h

> Declares the attachable-item mix-in implemented in [`attachable_item.cpp`](attachable_item.cpp.md) and [`attachable_item_inline.h`](attachable_item_inline.h.md).

**Needs** — [`attachable_item_inline.h`](attachable_item_inline.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md)
**Used by** — [`attachable_item.cpp`](attachable_item.cpp.md) · [`attachable_item_inline.h`](attachable_item_inline.h.md) · [`attachment_owner.cpp`](attachment_owner.cpp.md) · [`attachment_owner.h`](attachment_owner.h.md) · [`console_commands.cpp`](console_commands.cpp.md) · [`inventory_item.h`](inventory_item.h.md) · [`script_game_object_inventory_owner.cpp`](script_game_object_inventory_owner.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`attachable_item.cpp`](attachable_item.cpp.md), with
the trivial accessors split out into
[`attachable_item_inline.h`](attachable_item_inline.h.md). A concrete item class inherits
this alongside the inventory-item mix-in; the two find each other at construction.

Exported units:

- **`_construct`** — link to the inventory-item half of the same object.
- **`cast_attachable_item`** — the downcast hook by which any object is asked "are you
  attachable".
- **`reload`, `load_attach_position`** — read bone name and offset from a section.
- **`enable`, `enabled`** — the single door in and out of the hanging state.
- **`can_be_attached`** — whether the item's inventory position permits hanging.
- **`OnH_A_Chield`, `OnH_A_Independent`** — reactions to gaining and losing a carrier.
- **`afterAttach`, `afterDetach`** — scheduler registration, called by the carrier.
- **`renderable_Render`** — submit the visual inside the carrier's render call.
- **`use_parent_ai_locations`** — required of every implementor; the item's location is its
  own while hanging and the carrier's otherwise.
- **`item`, `object`, `bone_name`, `bone_id`, `set_bone_id`, `offset`** — accessors.
- **the debug offset-tuning statics** — the in-game placement tool described in the
  implementation twin.
