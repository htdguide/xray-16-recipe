# src/xrGame/ui/UIInventoryUpgradeWnd.h

> Declares the upgrade bench, its scheme record, and the state texture tables the nodes index.

**Needs** — [`UIInventoryUpgradeWnd.cpp`](UIInventoryUpgradeWnd.cpp.md) · [`UIInventoryUpgradeWnd_add.cpp`](UIInventoryUpgradeWnd_add.cpp.md) · [`UIInvUpgrade.h`](UIInvUpgrade.h.md) · [`inventory_upgrade_manager.h`](../inventory_upgrade_manager.h.md)
**Used by** — [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIActorMenuUpgrade.cpp`](UIActorMenuUpgrade.cpp.md) · [`UIActorMenu_action.cpp`](UIActorMenu_action.cpp.md) · [`UIInvUpgrade.cpp`](UIInvUpgrade.cpp.md) · [`UIInvUpgrade.h`](UIInvUpgrade.h.md) · [`UIInventoryUpgradeWnd.cpp`](UIInventoryUpgradeWnd.cpp.md) · [`UIInventoryUpgradeWnd_add.cpp`](UIInventoryUpgradeWnd_add.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented across
[`UIInventoryUpgradeWnd.cpp`](UIInventoryUpgradeWnd.cpp.md) and
[`UIInventoryUpgradeWnd_add.cpp`](UIInventoryUpgradeWnd_add.cpp.md).

The layout document's name is a **global constant defined here**, because the actor menu also
needs it. A rebuild passes it instead.

The cell limit — twenty-five per scheme — is only a reservation hint; nothing enforces it.

## `Scheme`

```text
RECORD Scheme
  name  : text                 # matched against the registry's per-item scheme name
  cells : list<UpgradeNode>    # owned by the scheme, not by the window tree
```

The cells are owned here and merely *attached* to the display pads, which is why switching
schemes is a detach rather than a rebuild.

## Exported units

- **`CUIInventoryUpgradeWnd`**
  - `Init` — read the layout; returns false when it is absent.
  - `InitInventory(cell item, may upgrade)` — bind an item and a mechanic's willingness.
  - `Show`, `Update`, `Reset` — reset clears every node of every scheme, not only the
    current one.
  - `UpdateAllUpgrades` — recompute every node's verdict against the current item.
  - `get_cell_texture` / `get_point_texture` — the tables, indexed by view state; this is
    the interface the nodes use.
  - `get_scheme_position` / `get_item_position` — geometry the parent screen needs to place
    things over the bench.
  - `AskUsing` / `OnMesBoxYes` — the confirmation handshake.
  - `DBClickOnUIUpgrade` — treat a double click on a node as a click; the entry point a
    script or a keyboard shortcut uses.
  - `HighlightHierarchy` / `ResetHighlight` — delegate the highlight to the registry.
  - `set_info_cur_upgrade` — the dwell-filtered description hand-off.
  - `FindUIUpgrade` — the node bound to an upgrade in the current scheme, if any.
  - `m_btn_repair` — public, because the parent screen binds its handler.

**Notes** — two icon-material fields survive as commented-out remnants of a scheme where the
bench owned its own atlases; they are now fetched from the shared registry. A rebuild deletes
them.
