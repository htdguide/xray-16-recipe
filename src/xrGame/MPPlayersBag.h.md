# src/xrGame/MPPlayersBag.h

> Declares the multiplayer drop bag implemented in [`MPPlayersBag.cpp`](MPPlayersBag.cpp.md).

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md) · [`MPPlayersBag.cpp`](MPPlayersBag.cpp.md)
**Used by** — [`MPPlayersBag.cpp`](MPPlayersBag.cpp.md) · [`game_sv_artefacthunt.cpp`](game_sv_artefacthunt.cpp.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CMPPlayersBag`, an inventory item that also contains inventory items — the
container a killed player's gear drops into. Substance is in
[`MPPlayersBag.cpp`](MPPlayersBag.cpp.md).

Exported units:

- `CMPPlayersBag` — the bag. Derives from the inventory-item entity, which is what lets a
  bag itself be carried.
- `OnEvent` — handles the take and reject ownership events, reparenting the item and
  snapping it to the bag.
- `NeedToDestroyObject` — the self-removal predicate, gated on the match's item-lifetime
  rule and on this side being the authority.
