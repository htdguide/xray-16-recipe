# src/xrGame/ui/UIInvUpgrade.h

> Declares one upgrade-tree node, its ten view states, its four layers, and the point marker that shares its hit area.

**Needs** — [`UIInvUpgrade.cpp`](UIInvUpgrade.cpp.md) · [`UIInventoryUpgradeWnd.h`](UIInventoryUpgradeWnd.h.md) · [`inventory_upgrade.h`](../inventory_upgrade.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIInvUpgrade.cpp`](UIInvUpgrade.cpp.md) · [`UIInventoryUpgradeWnd.cpp`](UIInventoryUpgradeWnd.cpp.md) · [`UIInventoryUpgradeWnd.h`](UIInventoryUpgradeWnd.h.md) · [`UIInventoryUpgradeWnd_add.cpp`](UIInventoryUpgradeWnd_add.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIInvUpgrade.cpp`](UIInvUpgrade.cpp.md). Three
enumerations are the substance and are reproduced in the implementation twin: the ten view
states, the four pointer states, and the five layers. Their *ordering* is load-bearing — the
window's texture tables are indexed by view state — so a rebuild that reorders them must
reorder those tables with them.

## Exported units

- **`UIUpgrade`** — one node.
  - `init_upgrade` — bind to an upgrade identifier and compute the first verdict.
  - `load_from_xml` — place the layers from the cell rectangle and, when present, the border
    rectangle; record the node's column and row in the authored scheme.
  - `set_texture(layer, name)` — the only way a layer's picture changes; a missing optional
    layer silently ignores it.
  - `update_item` — recompute the verdict against an item.
  - `update_upgrade_state` / `update_mask` — derive the view state, and re-choose the colour
    and point textures from the window's tables when it changed.
  - `highlight_relation` — ask the window to mark or unmark this node's prerequisite and
    dependant chain.
  - the pointer handlers and the button-state accessors.
  - `get_upgrade` — resolve the identifier in the game's upgrade registry; the node holds
    the identifier, never the upgrade.
  - `attach_point` — adopt the point marker.
  - `offset` — public, written by the window when it lays the tree out.

- **`CUIUpgradePoint`** — the small marker inside a cell, a second hit target for the same
  node. It must hold a reference to its node; see the implementation twin for the defect
  where it does not.

**Notes** — the node reports the wrong type name to the development inspector — it names the
cell board instead of itself. A copy-paste, with no consequence beyond a confusing inspector.
