# src/xrGame/ui/UIInvUpgradeProperty.h

> Declares the property row and the list that stacks the applicable ones.

**Needs** — [`UIInvUpgradeProperty.cpp`](UIInvUpgradeProperty.cpp.md) · [`inventory_upgrade_property.h`](../inventory_upgrade_property.h.md) · [`inventory_item.h`](../inventory_item.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIInvUpgradeInfo.cpp`](UIInvUpgradeInfo.cpp.md) · [`UIInvUpgradeInfo.h`](UIInvUpgradeInfo.h.md) · [`UIInvUpgradeProperty.cpp`](UIInvUpgradeProperty.cpp.md) · [`UIItemInfo.cpp`](UIItemInfo.cpp.md) · [`UIItemInfo.h`](UIItemInfo.h.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in
[`UIInvUpgradeProperty.cpp`](UIInvUpgradeProperty.cpp.md).

## Exported units

- **`UIProperty`** — one row: an icon and a value.
  - `init_from_xml` — build the icon and the value label from the shared row element.
  - `init_property(id)` — bind to a property in the registry and take its icon and tint;
    **returns false for an unknown identifier**, which is how a bad configuration entry is
    dropped.
  - `compute_value(upgrades)` — does this row apply to this upgrade set, and what does it
    say? See the implementation twin.
  - `show_result(sections)` — hand the contributing sections to the property's script
    function and display what comes back.
  - `read_value_from_section` — a configuration read that tolerates absence; unused.

- **`UIInvUpgPropertiesWnd`** — the list.
  - `init_from_xml` — build one row per entry of the configured property section; returns
    false if the layout document is absent.
  - `set_upgrade_info(upgrade)` — fill from one upgrade.
  - `set_item_info(item)` — fill from an item's installed upgrade set.
