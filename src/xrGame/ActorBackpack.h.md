# src/xrGame/ActorBackpack.h

> Declares the backpack item implemented in [`ActorBackpack.cpp`](ActorBackpack.cpp.md).

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md)
**Used by** — [`ActorBackpack.cpp`](ActorBackpack.cpp.md) · [`Actor_Movement.cpp`](Actor_Movement.cpp.md) · [`InventoryOwner.h`](InventoryOwner.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CBackpack`, an inventory item that is nothing but a set of carry-limit and
movement modifiers the actor reads. Substance is in
[`ActorBackpack.cpp`](ActorBackpack.cpp.md).

Exported units:

- `CBackpack` — the item. Its seven modifier fields are public and read directly by the
  actor's condition and movement code; a rebuild may prefer an accessor, but the fields
  *are* the interface.
- `Load` — reads the modifiers from the section.
- `Hit` — wears the item by the immunity-scaled damage the wearer took.
- `install_upgrade_impl` — applies the upgradeable subset.

The class is final: nothing derives from a backpack.
