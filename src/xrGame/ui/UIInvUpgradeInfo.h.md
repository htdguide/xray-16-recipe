# src/xrGame/ui/UIInvUpgradeInfo.h

> Declares the upgrade description panel.

**Needs** — [`UIInvUpgradeInfo.cpp`](UIInvUpgradeInfo.cpp.md) · [`UIInvUpgradeProperty.h`](UIInvUpgradeProperty.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`inventory_upgrade.h`](../inventory_upgrade.h.md)
**Used by** — [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIActorMenuUpgrade.cpp`](UIActorMenuUpgrade.cpp.md) · [`UIInvUpgradeInfo.cpp`](UIInvUpgradeInfo.cpp.md) · [`UIInvUpgradeProperty.cpp`](UIInvUpgradeProperty.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIInvUpgradeInfo.cpp`](UIInvUpgradeInfo.cpp.md).

## Exported units

- **`UIInvUpgradeInfo`**
  - `init_from_xml` — build from a named layout document, including the nested property list.
  - `init_upgrade(upgrade, item)` — set the subject and recompose; **returns false when the
    subject is unchanged or empty**, which is what keeps a hover from recomputing the panel
    every frame.
  - `is_upgrade` / `get_upgrade` — the current subject.
  - `Draw` — skipped entirely when there is no subject.
