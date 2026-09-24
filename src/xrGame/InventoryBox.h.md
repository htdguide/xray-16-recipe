# src/xrGame/InventoryBox.h

> Declares the world container, implemented in [`InventoryBox.cpp`](InventoryBox.cpp.md).

**Needs** — [`inventory_space.h`](../xrServerEntities/inventory_space.h.md) · [`GameObject.h`](GameObject.h.md)
**Used by** — [`ActorInput.cpp`](ActorInput.cpp.md) · [`InventoryBox.cpp`](InventoryBox.cpp.md) · [`script_game_object.cpp`](script_game_object.cpp.md) · [`script_game_object4.cpp`](script_game_object4.cpp.md) · [`script_game_object_inventory_owner.cpp`](script_game_object_inventory_owner.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CInventoryBox`, a game object that holds items without being an inventory owner.
Substance is in [`InventoryBox.cpp`](InventoryBox.cpp.md).

The shape it fixes: a box derives from the plain game object, not from the inventory owner.
It therefore has no slots, no weight, no active item and no trade personality — only a list
of contained entity identifiers and three booleans. That is the decision worth carrying into
a rebuild; everything else follows from it.

Exported units:

- `CInventoryBox` — the box.
- `m_items` — the contents, by entity identifier, public because screens iterate it directly.
- `net_Spawn` / `net_Destroy` / `net_Relcase` / `UpdateCL` — the object lifecycle; only the
  spawn does anything of its own.
- `OnEvent` — the four ownership events: take, reject, buy, sell.
- `AddAvailableItems` — resolve the contents for a screen.
- `IsEmpty` — nothing inside.
- `set_in_use` / `in_use` — a screen is open on this box; gates the script callback.
- `set_can_take` / `can_take` — items may be removed.
- `set_closed` / `closed` — the box refuses to open, with a reason that becomes the prompt.
- `SE_update_status` — broadcast the three status values as one message.
