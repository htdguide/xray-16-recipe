# src/xrGame/purchase_list.h

> Declares the trader restocking list and its per-section deficit table.

**Needs** — [`xrCore/Containers/AssociativeVector.hpp`](../xrCore/Containers/AssociativeVector.hpp.md) · [`purchase_list_inline.h`](purchase_list_inline.h.md)
**Used by** — [`InventoryOwner.cpp`](InventoryOwner.cpp.md) · [`purchase_list.cpp`](purchase_list.cpp.md) · [`purchase_list_inline.h`](purchase_list_inline.h.md)
**Tier floor** — T3: a declaration over one map

## Purpose

Declares the surface implemented in [`purchase_list.cpp`](purchase_list.cpp.md) and
[`purchase_list_inline.h`](purchase_list_inline.h.md).

Exported units:

- `process(ini_file, section, owner)` — restock an inventory owner from a shopping-list
  section.
- `deficit(section)` — the recorded price multiplier for one item section, defaulting to
  `1.0` when the section was never rolled.
- `deficit(section, value)` — overwrite one multiplier.
- `deficits()` — the whole table.

The per-line roll is private; its algorithm is in the implementation twin.
